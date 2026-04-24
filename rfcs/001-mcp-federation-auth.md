# [RFC] MCP Federation Architecture & Service-to-Service Authentication

**Authors:**

- @radektomasek

## 1. Executive Summary

This RFC proposes a production-grade architecture for connecting multiple Model Context Protocol (MCP) servers in a federated hierarchy, along with a layered service-to-service authentication strategy to support enterprise-scale deployments.

The design consists of:

- A single **root MCP server** as the only public-facing surface, exposed via HTTPS with OAuth2
- Multiple **private specialist MCP servers** on an internal network, unreachable from the internet
- A **context propagation mechanism** carrying user identity and delegated tokens across every internal hop
- A **progressive auth model** — from shared secrets in early deployments up to OAuth2 client credentials and mTLS at enterprise scale
- A **token delegation pattern** (OAuth2 On-Behalf-Of) for cases where downstream services must act with user identity (e.g. Google Docs, user-scoped APIs)

## 2. Motivation

As AI-powered systems mature, the naive pattern of connecting a single AI client (e.g. Claude) directly to a single MCP server breaks down under three enterprise pressures:

**Capability sprawl** — a single MCP server accumulates tools for Snowflake, Google Docs, Slack, GitHub, scraping, vector search, and more. This creates an unmanageable, monolithic surface with no isolation between concerns.

**Auth complexity** — enterprise customers require auditability ("who accessed what, when, through which path"), least-privilege access, and blast radius containment. A single credential shared across all tools cannot satisfy these requirements.

**Operational risk** — a monolithic MCP has no fault isolation. A bug in the Snowflake tool can affect Slack delivery. A compromised credential exposes every backend simultaneously.

The federation pattern addresses all three by decomposing the monolith into a hierarchy of focused, independently deployable services — while the auth layering ensures that the decomposition does not introduce new attack surfaces.

This architecture is also the foundation for any RAG ingestion pipeline where an AI orchestrator needs to read from heterogeneous sources (databases, document stores, web) and write to a vector search engine (e.g. [Antfly](https://antfly.io)) — a class of problem that is increasingly common in enterprise AI adoption.

## 3. Proposed Implementation

### 3.1 Topology

The architecture follows a **federation (fan-out) pattern**:

```
                    [Claude / AI Client]
                           │
                    OAuth2 Bearer token
                           │
                    ┌──────▼──────┐
                    │  Root MCP   │  ← only public HTTPS surface
                    │ orchestrator│
                    └──────┬──────┘
               ════════════╪════════════ trust boundary (private network)
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │ Snowflake   │  │  Docs MCP   │  │  Search MCP │
   │    MCP      │  │ (GDrive etc)│  │ (Confluence)│
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
   Snowflake API     Google API       Confluence API
```

Key properties:
- Private MCPs bind to `127.0.0.1` or internal CIDR only — no public port, no exposed auth surface
- Each private MCP owns its own backend credentials (Snowflake service account, Google SA key, etc.)
- The root MCP is simultaneously an MCP *server* (to Claude) and an MCP *client* (to each private MCP)
- Transport on all legs: **Streamable HTTP** (MCP 2025-03-26 spec) — single `POST /mcp` endpoint, chunked response, stateless-friendly

### 3.2 Service Discovery — the `describe()` contract

Every specialist MCP implements one mandatory tool:

```python
@mcp.tool()
def describe() -> SourceManifest:
    return SourceManifest(
        name="snowflake-mcp",
        source_type="database",
        capabilities=["query", "stream_rows", "list_tables"],
        estimated_doc_count=2_300_000,
        last_indexed=datetime(2025, 4, 20),
        health="ok"
    )
```

On startup, the root MCP polls all registered MCPs in parallel via `describe()` and builds a live capability registry. This allows Claude to call `available_sources()` on the root and receive a structured manifest before committing to any ingestion plan — enabling user-driven scope selection rather than blind full-crawls.

### 3.3 Context Propagation

User identity and delegated tokens travel through the call chain as a signed HTTP header on every internal MCP call:

```
X-MCP-Context: {
  "user_id":         "user@company.com",
  "request_id":      "req_01jx...",        // distributed trace ID
  "delegated_token": "eyJ...",             // OBO token or null
  "scopes":          ["docs.readonly"],
  "issued_at":       1714000000,
  "signature":       "..."                 // signed by root MCP private key
}
```

- The root MCP **creates** the context from the incoming user token
- Private MCPs **validate** the signature before trusting any field
- Every layer **logs** `request_id` + `user_id` against every operation — full distributed trace
- Private MCPs **never** accept a context not signed by the root — prevents identity forgery from internal callers

### 3.4 Authentication Layers

Three distinct auth problems, three distinct solutions:

| Leg | Problem | Solution |
|-----|---------|----------|
| Claude → Root MCP | Human authenticates to public surface | OAuth2 (PKCE for interactive, client credentials for automated) |
| Root MCP → Private MCPs | Machine-to-machine, internal network | Shared secret (early) → OAuth2 client credentials (growth) → mTLS (scale) |
| Private MCP → Backend APIs | Service credentials for external APIs | Static credentials in secret store (GCP Secret Manager / Vault) |

**Progressive rollout:**

- **Stage 1 (ship):** shared secret in env var on Docker bridge network — zero infra, rotate manually
- **Stage 2 (growth):** OAuth2 client credentials via lightweight IdP (Zitadel, Keycloak, Auth0) — auditable, scoped, auto-rotating tokens
- **Stage 3 (scale):** mTLS via service mesh (Linkerd on K8s) — no code changes, sidecar handles cert lifecycle
- **Stage 4 (enterprise):** SPIFFE workload identity — cryptographic identity per workload, no secrets to rotate

### 3.5 Token Delegation for User-Scoped APIs

When a private MCP must act as the user (e.g. reading their personal Google Docs), a shared service account is insufficient. The solution is **OAuth2 On-Behalf-Of (OBO)**:

```
User token (token A)
       │
       ▼
Root MCP: POST /token
  grant_type = urn:ietf:params:oauth:grant-type:jwt-bearer
  assertion  = token_A
  scope      = docs.readonly
       │
       ▼
IdP / Google issues token B
  - same user identity
  - narrower scope: docs.readonly
  - short TTL
       │
       ▼
Root passes token B in X-MCP-Context.delegated_token
       │
       ▼
Docs MCP uses token B for Google API calls
Google audit log shows real user identity ✓
```

**When to use OBO vs service account:**

| Scenario | Approach |
|----------|----------|
| User's personal documents | OBO — their token, their docs, their audit trail |
| Shared company drive | Service account — SA granted on shared drive by admin |
| Regulated access (must prove user accessed) | OBO — Google audit log shows real identity |
| Background ingestion / no live user session | Service account — no user session available |

### 3.6 Deployment Shape

**Development / early production:**

```yaml
# compose.yml
services:
  root-mcp:
    ports: ["443:8000"]          # only public port in the stack
    environment:
      SNOWFLAKE_MCP_URL: http://snowflake-mcp:8001
      DOCS_MCP_URL:      http://docs-mcp:8002
      INTERNAL_SECRET:   ${INTERNAL_SECRET}
      ROOT_SIGNING_KEY:  ${ROOT_SIGNING_KEY}

  snowflake-mcp:                 # no ports: block = unreachable from internet
    environment:
      SNOWFLAKE_ACCOUNT: ${SNOWFLAKE_ACCOUNT}
      SNOWFLAKE_TOKEN:   ${SNOWFLAKE_TOKEN}
      INTERNAL_SECRET:   ${INTERNAL_SECRET}

  docs-mcp:
    environment:
      GOOGLE_SA_KEY:   ${GOOGLE_SA_KEY}
      INTERNAL_SECRET: ${INTERNAL_SECRET}
```

Docker's internal DNS handles service discovery. Private services have no public port binding — not a firewall rule but a binding constraint.

**Production (GCP / K8s):** Root MCP on Cloud Run or GKE with public ingress. Private MCPs as internal GKE services. Caddy or Envoy handles TLS termination at the root. Secret Manager for all credentials. Linkerd for mTLS on internal legs when required.

## 4. Metrics & Dashboards

**Per-service:**

- `mcp_tool_call_duration_ms` — p50/p95/p99 per tool per service
- `mcp_tool_call_error_rate` — by error type (auth, timeout, upstream, validation)
- `mcp_active_connections` — concurrent client sessions on root MCP

**Auth layer:**

- `token_exchange_duration_ms` — OBO token exchange latency
- `token_refresh_failures` — IdP unavailability detection
- `context_validation_failures` — invalid signatures (potential attack signal)

**Federation health:**

- `describe_poll_duration_ms` — specialist MCP response time at startup
- `specialist_mcp_health` — per-service UP/DOWN from last describe() poll
- `downstream_api_error_rate` — Snowflake, Google, etc. broken out by service

**Ingestion (if used for RAG pipeline):**

- `ingestion_docs_per_second` — throughput per source
- `ingestion_cursor_lag` — distance between last committed cursor and source head
- `dead_letter_queue_depth` — failed batches pending retry

## 5. Drawbacks

**Operational complexity scales with service count.** Each new specialist MCP is a new deployment, a new set of credentials to rotate, a new service to monitor. The federation pattern pays off at 3+ specialists — below that a single MCP server with well-separated tool modules is simpler.

**The context propagation mechanism is custom.** MCP does not have a built-in identity propagation standard. The `X-MCP-Context` header and signing scheme described here is an application-level convention. Any new MCP implementation must implement it correctly or identity guarantees break silently.

**OBO adds an IdP dependency on the critical path.** Every user-scoped Google Docs call requires a token exchange round-trip to the IdP. If the IdP is slow or unavailable, user-facing calls fail. Mitigation: cache derived tokens with TTL slightly shorter than their expiry.

**Blast radius of root MCP.** While private MCPs are isolated from each other, the root MCP is a single point of failure. A bug or outage there affects all specialists simultaneously. Mitigation: keep root MCP logic thin — routing and context propagation only, no business logic.

## 6. Alternatives

**Single monolithic MCP server.** Simpler to deploy and debug. Acceptable for small tool sets (<5 tools, single team, single backend). Does not scale to multi-team, multi-backend, or compliance-sensitive deployments. Becomes the bottleneck for all tool development.

**A2A (Agent-to-Agent protocol) instead of MCP-to-MCP.** A2A is appropriate when specialist agents are operated by different organizations or need formal capability discovery across trust domains. For internal multi-service architectures within a single organization, MCP-to-MCP is simpler — no agent card infrastructure, no A2A task lifecycle management. A2A can be layered on top of this architecture later for cross-org federation.

**Service mesh from day one (Istio/Linkerd).** Provides mTLS and workload identity without any application code. High operational overhead for early-stage deployments. Recommended only when moving to Kubernetes with multiple engineering teams.

**API Gateway pattern (Kong, AWS API Gateway).** Route all internal MCP calls through a centralized gateway. Adds a network hop and a new infrastructure dependency. Appropriate if the organization already operates an API gateway and wants centralized rate limiting and observability. Not recommended as the first choice for a greenfield MCP architecture.

## 7. Potential Impact and Dependencies

**Teams affected:**

- Any team building or consuming MCP tools needs to implement the `describe()` contract and accept the `X-MCP-Context` header
- Security/infra team needs to provision the IdP client registrations for OAuth2 client credentials
- Platform team needs to manage the secret store (GCP Secret Manager or equivalent)

**External dependencies:**

- IdP availability (Auth0, Keycloak, or Zitadel) becomes a dependency for the auth layer at Stage 2+
- Google's token endpoint availability for OBO flows
- Each specialist MCP inherits its backend's availability (Snowflake SLA, Google Docs API SLA, etc.)

**Security attack surface:**

- The root MCP is the primary attack surface — it must validate every incoming token rigorously and never pass unvalidated input to private MCPs
- The `X-MCP-Context` signature prevents internal identity forgery, but only if private MCPs actually validate it — this must be enforced via shared middleware, not left to individual implementations
- OBO token caching must respect token TTL precisely — a stale token used after expiry could produce unexpected access denial or, worse, access with a revoked token
- Service account keys stored in secret manager must have minimum necessary scopes — a key leaked from one specialist MCP should not grant access to any other system

## 8. Unresolved Questions

- **Context signing key rotation.** How does the root MCP rotate its signing key without invalidating in-flight requests? A key versioning scheme (kid header) is implied but not specified here.

- **Specialist MCP versioning.** When a specialist MCP changes its `describe()` manifest (adds/removes capabilities), how does the root registry detect and react? Currently a restart of the root is implied.

- **Multi-tenant isolation.** In a multi-tenant deployment (multiple customers sharing the same MCP infrastructure), how is tenant identity carried in context and enforced at each specialist? Out of scope for this RFC but a necessary follow-up.

- **OBO token caching strategy.** The RFC recommends caching derived tokens — the exact cache key design (user_id + scope + target service), cache backend, and invalidation strategy are not specified.

- **Rate limiting.** No per-user or per-tenant rate limiting is specified at the root MCP. This needs to be addressed before public deployment.

- **Streaming backpressure.** For long-running ingestion jobs where a specialist MCP streams large result sets, backpressure handling between the specialist and the root is not defined.

## 9. Conclusion

The federation pattern described here — single public root, private specialist MCPs, context propagation with signed identity, progressive auth model — is the right architectural foundation for enterprise-grade MCP deployments at this point in time.

It satisfies the three core enterprise requirements: auditability (full distributed trace + user identity on every hop), least privilege (scoped tokens per downstream call, SA permissions bounded to specific resources), and blast radius containment (specialist compromise does not propagate).

The progressive auth model means the architecture is not over-engineered for early deployments — a team can ship with shared secrets and Docker Compose and graduate to OAuth2 client credentials and mTLS as operational maturity and compliance requirements demand, without a rewrite.

The design is intentionally conservative on A2A adoption — the MCP-to-MCP pattern is simpler and sufficient for intra-organization federation. A2A can be introduced at the boundary layer when cross-organization agent coordination becomes a concrete requirement.
