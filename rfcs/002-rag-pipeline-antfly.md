# [RFC] RAG Ingestion Pipeline Architecture with MCP Federation and Antfly

**Authors:**

- @radektomasek

## 1. Executive Summary

This RFC describes the end-to-end architecture for a production-grade Retrieval-Augmented Generation (RAG) ingestion and query pipeline, built on top of a federated MCP hierarchy with Antfly as the vector search sink.

The system has two distinct operational modes:

- **Ingestion mode** — a scheduled or on-demand pipeline that discovers, extracts, chunks, enriches, and indexes documents from heterogeneous sources (databases, document stores, web, APIs) into Antfly
- **Query mode** — a real-time path where an AI client (Claude) retrieves relevant context from Antfly to ground its responses

Both modes share the same MCP federation infrastructure and the same service-to-service auth model described in the companion RFC on MCP Federation and Authentication. This document extends that foundation with the RAG-specific layers: source connectors, chunking strategy, ingestion state management, Antfly integration, and the query path.

## 2. Motivation

RAG is the dominant pattern for grounding AI responses in proprietary enterprise data. However, most RAG implementations fail in production for one of three reasons:

**Ingestion is brittle** — a single failed batch corrupts or halts the entire pipeline. There is no cursor, no retry, no dead-letter queue. A re-run means starting over from scratch.

**Sources are siloed** — each data source (DBs/Warehouses like Snowflake, Confluence, Google Drive, web) has a bespoke connector with no shared interface. Adding a new source requires re-architecting the pipeline.

**Auth is bolted on** — credentials are hardcoded or shared across services. There is no audit trail of what the pipeline accessed on whose behalf.

This RFC addresses all three by combining the MCP federation pattern (uniform source interface, isolated credentials, propagated identity) with Antfly's native async enrichment and linear merge capabilities (resilient bulk ingestion, idempotent upserts, background embedding).

The result is a pipeline that can be described to an enterprise security team ("every access is audited, every credential is scoped, every failure is recoverable") and to an engineering team ("add a new source by implementing four methods, deploy as a Docker service, done").

---

## 3. Proposed Implementation

### 3.1 Full System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  AI CLIENT LAYER                                                 │
│                                                                  │
│   Claude ──── OAuth2 ────► Root MCP (public HTTPS)              │
│                             │                                    │
│                             ├── available_sources()             │
│                             ├── plan_scan(sources, depth)       │
│                             ├── ingest_all(job_id)              │
│                             ├── query_status(job_id)            │
│                             └── search(query, collection)       │
└─────────────────────────────────────────────────────────────────┘
                              │
          ════════════════════╪════════ trust boundary
                              │
┌─────────────────────────────────────────────────────────────────┐
│  INGESTION LAYER (private network)                               │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────┐ │
│  │ Snowflake   │  │  Docs MCP   │  │  Web MCP    │  │  API   │ │
│  │    MCP      │  │ GDrive/Conf │  │  scrape/    │  │  MCP   │ │
│  │             │  │             │  │  crawl      │  │        │ │
│  │ describe()  │  │ describe()  │  │ describe()  │  │describe│ │
│  │ stream_rows │  │ list_pages  │  │ crawl_site  │  │fetch   │ │
│  │ list_tables │  │ read_doc    │  │ scrape_url  │  │paginate│ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───┬────┘ │
│         │                │                │              │      │
│         └────────────────┴────────────────┴──────────────┘      │
│                                    │                             │
│                          ┌─────────▼─────────┐                  │
│                          │  Chunking &        │                  │
│                          │  Enrichment MCP    │                  │
│                          │                    │                  │
│                          │ semantic_chunk()   │                  │
│                          │ summarize()        │                  │
│                          │ assign_collection()│                  │
│                          │ deduplicate()      │                  │
│                          └─────────┬─────────┘                  │
└────────────────────────────────────┼────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────┐
│  STORAGE LAYER                     │                             │
│                                    │                             │
│                          ┌─────────▼─────────┐                  │
│                          │   Antfly MCP       │                  │
│                          │                    │                  │
│                          │ upsert()           │                  │
│                          │ linear_merge()     │                  │
│                          │ query()            │                  │
│                          │ list_collections() │                  │
│                          │ get_status()       │                  │
│                          └─────────┬─────────┘                  │
│                                    │                             │
│                     ┌──────────────▼──────────────┐             │
│                     │      Antfly Cluster           │             │
│                     │                              │             │
│                     │  multi-Raft shards           │             │
│                     │  BM25 + vector (RaBitQ)      │             │
│                     │  async enrichment            │             │
│                     │  HTTP/3 + QUIC               │             │
│                     └──────────────────────────────┘             │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐          │
│   │  Ingestion state DB  │    │  Secret store         │          │
│   │  (SQLite / Redis)    │    │  (GCP Secret Manager) │          │
│   │                      │    │                       │          │
│   │  job plans           │    │  per-service creds    │          │
│   │  cursors per source  │    │  OAuth client ids     │          │
│   │  error log / DLQ     │    │  signing keys         │          │
│   └──────────────────────┘    └──────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 The Source Connector Contract

Every specialist MCP implements the same four-method interface. This is the key abstraction that makes adding a new source a self-contained unit of work:

```python
class SourceConnector:

    def describe(self) -> SourceManifest:
        """
        Called by root MCP on startup and periodically.
        Returns capability declaration — used for discovery and
        scope negotiation with the user before any ingestion begins.
        """
        return SourceManifest(
            name="snowflake-mcp",
            source_type="database",
            capabilities=["query", "stream_rows", "list_tables"],
            estimated_doc_count=2_300_000,
            last_indexed=datetime(2025, 4, 20),
            health="ok",
            collections_available=["product_docs", "support_tickets"]
        )

    def list_items(
        self,
        collection: str,
        cursor: str | None = None
    ) -> ItemPage:
        """
        Paginated listing of available items.
        cursor=None means start from beginning.
        Returns items + next_cursor (None if exhausted).
        """

    def stream_content(
        self,
        item_id: str,
        context: MCPContext
    ) -> AsyncIterator[ContentChunk]:
        """
        Stream raw content for a single item.
        context carries user identity for OBO auth where needed.
        Yields chunks — allows backpressure, avoids memory spikes
        on large documents.
        """

    def get_item_hash(self, item_id: str) -> str:
        """
        Returns a stable hash of item content.
        Used by deduplication layer to skip unchanged items
        on incremental re-scans without fetching full content.
        """
```

### 3.3 Ingestion Flow — Step by Step

**Phase 1: Discovery and scope negotiation**

```
root polls all MCPs: describe() ──► builds registry
root presents to Claude: available_sources() ──► manifest[]
Claude / user selects: plan_scan({ sources: ["snowflake", "gdrive"],
                                   depth: "full",
                                   collection: "company-kb" })
root creates job plan in state DB ──► returns { job_id, estimated_docs }
```

**Phase 2: Incremental extraction**

For each selected source, the root MCP runs an extraction worker:

```
loop:
  page = source_mcp.list_items(collection, cursor=last_cursor)
  for item in page.items:
    hash = source_mcp.get_item_hash(item.id)
    if hash == state_db.get_hash(item.id):
      continue                          # unchanged — skip
    content = source_mcp.stream_content(item.id, context)
    queue.push(content)
  state_db.save_cursor(source, page.next_cursor)
  if page.next_cursor is None:
    break
```

The cursor is written to the state DB **after each page**, not after the full source. A crash mid-source resumes from the last committed page, not from scratch.

**Phase 3: Chunking and enrichment**

Raw content flows through the chunking MCP before reaching Antfly:

```python
chunks = chunking_mcp.semantic_chunk(
    content=raw_content,
    strategy="sentence_window",   # or "recursive", "fixed_size"
    chunk_size=512,
    overlap=64,
    metadata={
        "source": "snowflake",
        "item_id": item.id,
        "collection": "company-kb",
        "user_id": context.user_id,   # audit trail
        "ingested_at": utcnow()
    }
)
```

Antfly's own Termite chunking service can handle this natively — the chunking MCP is a thin wrapper that allows strategy selection per source type and adds metadata enrichment before the Antfly write.

**Phase 4: Antfly ingestion via linear_merge**

```python
antfly_mcp.linear_merge(
    collection="company-kb",
    documents=chunks,              # pre-sorted by doc_id
    embedding_provider="openai",   # declared per collection
    sync_level="write"             # or "full_text", "enrichments"
                                   # depending on consistency needs
)
```

`linear_merge` is the right call here rather than `upsert` — it is designed for bulk import of sorted records without read-modify-write overhead. Antfly handles embedding generation asynchronously in the background via its enrichment pipeline. The MCP layer does not need to call any embedding API directly.

**Phase 5: Progress reporting**

The root MCP streams progress events back to Claude via chunked HTTP:

```
event: progress
data: { job_id, source: "snowflake", docs_processed: 12400,
        docs_total: 2300000, eta_seconds: 3200, errors: 0 }

event: progress
data: { job_id, source: "gdrive", docs_processed: 8400,
        docs_total: 8400, eta_seconds: 0, errors: 2 }

event: completed
data: { job_id, total_docs: 20800, duration_seconds: 4100,
        collections: ["company-kb"], failed_items: [...] }
```

### 3.4 Fault Tolerance and Recovery

**Cursor-based resumability**

Every source maintains an independent cursor in the state DB. Sources run in parallel — a failure in the Snowflake extractor does not block the Google Drive extractor. On restart, each source resumes from its last committed cursor.

```
state_db schema:
  jobs(job_id, status, created_at, completed_at, config_json)
  source_cursors(job_id, source_name, cursor, updated_at)
  item_hashes(source_name, item_id, hash, indexed_at)
  dead_letter(job_id, source_name, item_id, error, retry_count, next_retry_at)
```

**Job state machine**

```
queued ──► running ──► completed
                  └──► failed (all sources exhausted retries)
                  └──► partial (some sources ok, some failed)
```

**Dead letter queue**

Failed items (network errors, parse failures, upstream API errors) are written to the DLQ with `retry_count` and `next_retry_at` (exponential backoff). A background worker re-attempts DLQ items independently of the main ingestion flow. Items that fail more than `MAX_RETRIES` times are flagged for human review.

**Idempotent upserts**

Antfly's `linear_merge` is idempotent by `doc_id`. Re-processing a batch that was already ingested produces no duplicate documents. Combined with the item hash check in Phase 2, incremental re-scans are efficient — only changed items are re-chunked and re-ingested.

### 3.5 Query Path

Once ingested, retrieval is a single tool call from Claude through the root MCP:

```
Claude ──► root_mcp.search(
             query="what is our refund policy",
             collection="company-kb",
             mode="hybrid",           # BM25 + vector, RRF fusion
             top_k=10,
             filters={ "source": ["confluence", "gdrive"] }
           )
     ◄── ranked_results[{ content, score, metadata, source }]
```

The root MCP calls `antfly_mcp.query()` which hits Antfly's hybrid search engine — BM25 full-text and vector similarity fused via Reciprocal Rank Fusion. Results are returned with source metadata intact, allowing Claude to cite the original document.

For streaming RAG (long answers generated while results are retrieved), Antfly's built-in SSE streaming can be exposed directly through the Antfly MCP's `stream_rag()` tool.

### 3.6 Auth Integration

The auth model from the companion RFC applies across both ingestion and query paths:

| Path | Auth mechanism |
|------|---------------|
| Claude → Root MCP | OAuth2 Bearer token |
| Root → Source MCPs | Shared secret (Stage 1) / OAuth2 client credentials (Stage 2+) |
| Root → Antfly MCP | Shared secret / OAuth2 client credentials |
| Source MCP → Google Drive | Service account (shared content) or OBO token (personal content) |
| Source MCP → Snowflake | Service account + role in secret store |
| Source MCP → Confluence | API token per service in secret store |
| All internal hops | Signed X-MCP-Context header carrying user_id + request_id |

The `X-MCP-Context` header carries `user_id` on every ingestion call. This means every document written to Antfly has an `ingested_by` metadata field traceable to a real user identity — satisfying enterprise audit requirements even for background pipeline runs initiated by a human via Claude.

### 3.7 Deployment Shape

**Development (single VM, Docker Compose):**

```yaml
services:
  root-mcp:
    ports: ["443:8000"]
    environment:
      SNOWFLAKE_MCP_URL:  http://snowflake-mcp:8001
      DOCS_MCP_URL:       http://docs-mcp:8002
      WEB_MCP_URL:        http://web-mcp:8003
      CHUNKING_MCP_URL:   http://chunking-mcp:8004
      ANTFLY_MCP_URL:     http://antfly-mcp:8005
      INTERNAL_SECRET:    ${INTERNAL_SECRET}
      ROOT_SIGNING_KEY:   ${ROOT_SIGNING_KEY}

  snowflake-mcp:
    environment:
      SNOWFLAKE_ACCOUNT:  ${SNOWFLAKE_ACCOUNT}
      SNOWFLAKE_TOKEN:    ${SNOWFLAKE_TOKEN}
      INTERNAL_SECRET:    ${INTERNAL_SECRET}

  docs-mcp:
    environment:
      GOOGLE_SA_KEY:      ${GOOGLE_SA_KEY}
      INTERNAL_SECRET:    ${INTERNAL_SECRET}

  web-mcp:
    environment:
      INTERNAL_SECRET:    ${INTERNAL_SECRET}

  chunking-mcp:
    environment:
      INTERNAL_SECRET:    ${INTERNAL_SECRET}

  antfly-mcp:
    environment:
      ANTFLY_URL:         http://antfly:8080
      ANTFLY_API_KEY:     ${ANTFLY_API_KEY}
      INTERNAL_SECRET:    ${INTERNAL_SECRET}

  antfly:
    # Antfly in swarm mode — single binary, all components
    ports: []             # internal only
    volumes:
      - antfly-data:/data

  state-db:
    image: sqlite-web     # or redis for multi-instance
    volumes:
      - state-data:/data

volumes:
  antfly-data:
  state-data:
```

**Production (GCP / GKE):**

- Root MCP: Cloud Run (stateless, auto-scales, public HTTPS via Cloud Load Balancing)
- Source MCPs: GKE internal services (no external ingress)
- Antfly: GKE StatefulSet (separate metadata nodes + storage nodes, production mode)
- State DB: Cloud SQL (PostgreSQL) or Firestore for cursor + job state
- Secret Manager: GCP Secret Manager for all credentials
- mTLS: Linkerd sidecar on all internal services (Stage 3 auth)

## 4. Metrics & Dashboards

**Ingestion pipeline health:**

| Metric | Description | Alert threshold |
|--------|-------------|-----------------|
| `ingestion_docs_per_second` | Throughput per source | < 10 docs/s for >5 min |
| `ingestion_cursor_lag` | Offset behind source head | > 100k docs for >1hr |
| `dlq_depth` | Failed items pending retry | > 1000 |
| `dlq_max_retry_exceeded` | Items requiring human review | > 0 |
| `job_duration_seconds` | End-to-end job time | p95 > 2x baseline |
| `chunk_size_bytes` | Distribution of chunk sizes | p99 > 2048 bytes (chunker misconfiguration) |

**Source connector health:**

| Metric | Description |
|--------|-------------|
| `source_mcp_describe_latency_ms` | Health poll response time |
| `source_mcp_stream_error_rate` | Upstream API failures by source |
| `source_mcp_hash_cache_hit_rate` | Efficiency of incremental scans |

**Antfly ingestion:**

| Metric | Description |
|--------|-------------|
| `antfly_linear_merge_duration_ms` | Bulk write latency |
| `antfly_enrichment_queue_depth` | Pending async embedding jobs |
| `antfly_index_size_docs` | Total indexed document count per collection |
| `antfly_shard_replication_lag` | Multi-Raft replication health |

**Query path:**

| Metric | Description |
|--------|-------------|
| `query_latency_ms` | End-to-end search latency (p50/p95/p99) |
| `query_result_count` | Docs returned per query (low = collection problem) |
| `hybrid_search_bm25_weight` | BM25 vs vector contribution ratio |
| `rag_answer_latency_ms` | Time to first streamed token |

## 5. Drawbacks

**Chunking strategy is hard to get right and costly to fix.** The chunk size, overlap, and strategy (sentence window vs recursive vs fixed) determine retrieval quality. A bad chunking configuration produces low-recall results that are hard to diagnose. Changing the strategy requires re-ingesting the entire collection. Mitigation: run chunking strategy experiments on a small collection subset before committing to a full ingestion run.

**Antfly async enrichment means embedding lag.** `linear_merge` returns before embeddings are generated. A document ingested at time T is not searchable via vector similarity until Antfly's enrichment workers process it — which could be seconds to minutes under load. For real-time ingestion use cases this lag may be unacceptable. Mitigation: use `sync_level=enrichments` on the Antfly write, which blocks until embeddings are generated. This trades throughput for freshness.

**State DB becomes a single point of failure for ingestion.** If the cursor state DB is unavailable, all ingestion jobs cannot commit progress and will stall. Mitigation: use a replicated store (Cloud SQL with read replica, or Redis Sentinel) in production rather than single-instance SQLite.

**The chunking MCP adds a hop and a failure mode.** In the happy path this is fine. Under load, a slow chunking MCP becomes a bottleneck for all sources simultaneously. Mitigation: the chunking MCP should be stateless and horizontally scalable — multiple instances behind an internal load balancer.

**Web scraping is inherently unreliable.** The Web MCP's `crawl_site` and `scrape_url` tools depend on external site structure, robots.txt compliance, and rate limiting. Sites change their structure, add bot detection, or go down. This source type will always have a higher DLQ depth than database or document store sources. Mitigation: treat web sources as best-effort with a lower priority in the DLQ retry policy.

## 6. Alternatives

**LangChain / LlamaIndex document loaders.** Mature ecosystems with many pre-built connectors. Tightly coupled to Python and to specific orchestration frameworks. Harder to deploy as independent services and harder to integrate with enterprise auth patterns. Appropriate for single-team, single-language, lower-compliance deployments. The MCP connector pattern proposed here trades the breadth of pre-built connectors for deployment independence and auth flexibility.

**Direct Antfly SDK ingestion (no MCP layer).** Simpler for single-source pipelines. Loses the uniform connector interface — each new source requires bespoke integration code in the ingestion orchestrator. Loses the auth isolation — all source credentials must be available to the orchestrator process. Not recommended for multi-source, multi-team environments.

**Dedicated ETL platform (Airbyte, Fivetran, Keboola).** Best-in-class connector ecosystem, built-in scheduling, monitoring, and data quality. High operational overhead and cost for teams that primarily need to feed a RAG store rather than run general-purpose data pipelines. The MCP-based approach is lighter weight and AI-native — the same infrastructure that drives ingestion also drives real-time tool calls from Claude.

**Managed vector stores (Pinecone, Weaviate Cloud, Qdrant Cloud).** Reduces operational burden for the vector store layer. Loses the hybrid BM25+vector search that Antfly provides natively. Adds external network dependency and data residency concerns. Appropriate when Antfly's operational complexity (multi-Raft cluster management) is not justified by the deployment scale.

## 7. Potential Impact and Dependencies

**Teams affected:**

- **Engineering teams** owning each data source must implement or validate the four-method `SourceConnector` interface for their system
- **Security team** must review the OBO token delegation flow for sources that access user-scoped content (Google Drive, personal docs)
- **Data team** must define collection schema and chunking strategy per source type before ingestion begins — these decisions are expensive to reverse
- **Platform team** manages Antfly cluster sizing, replication topology, and embedding provider quotas

**External service dependencies:**

| Dependency | Impact if unavailable |
|------------|----------------------|
| Antfly cluster | Ingestion stalls; query path fails entirely |
| Embedding provider (OpenAI etc.) | Async enrichment queue backs up; vector search degrades to BM25-only |
| IdP (OAuth2 token endpoint) | User-initiated ingestion jobs cannot start; OBO flows fail |
| Source APIs (Snowflake, Google, etc.) | That source stalls; others continue independently |

**Security considerations:**

- The `item_hash` cache must be invalidated correctly when source content is deleted. A hash for a deleted document will cause the extractor to skip it on re-scans, leaving a stale chunk in Antfly. A periodic full-reconciliation scan (hash all current Antfly docs against source) is needed to catch deletions.
- Antfly collections should have per-collection access control. A user with access to the root MCP `search()` tool should not be able to query collections they were not granted access to. Collection-level auth in Antfly must be enforced by the Antfly MCP, not assumed to be enforced by the caller.
- Web scraping surfaces URLs to the scraper — a malicious user could craft a `plan_scan` request targeting internal infrastructure URLs. The Web MCP must maintain an allowlist of permitted domains and reject any URL not on the list.

## 8. Unresolved Questions

**Deletion propagation.** When a document is deleted from the source (a Confluence page is removed, a Snowflake row is deleted), how does the pipeline detect this and remove the corresponding chunks from Antfly? The current design detects additions and changes via item hash but has no deletion signal. Options: periodic full reconciliation scan, source-level change feed / CDC (where available), explicit delete events from source MCP.

**Multi-tenant collection isolation.** In a multi-tenant deployment, documents from different customers must not be mixed in the same collection or retrievable across tenant boundaries. The current design does not specify collection naming conventions or per-tenant access control enforcement at the Antfly layer.

**Chunking strategy versioning.** If the chunking strategy is changed after initial ingestion, the existing chunks are stale (different boundaries, different metadata). A migration path for re-chunking existing collections without a full re-ingest is not defined.

**Embedding model versioning.** If the embedding model is upgraded (e.g. OpenAI releases a new text-embedding model), existing vectors are incompatible with new query vectors. The collection must be re-embedded. Antfly's enrichment pipeline supports re-embedding but the trigger mechanism from the MCP layer is not defined.

**Ingestion rate limiting per source.** The current design does not specify how to prevent the pipeline from exceeding source API rate limits (Google Drive API quota, Snowflake query concurrency limits). Each source MCP should implement its own rate limiter, but the interface does not enforce this.

**Query result freshness SLA.** There is no defined SLA between a document being ingested via `linear_merge` and it being searchable via `query()`. This depends on Antfly's async enrichment queue depth and the `sync_level` parameter. A freshness SLA needs to be agreed per collection type.

## 9. Conclusion

This architecture delivers a production-grade RAG ingestion and query pipeline that satisfies three properties that naive RAG implementations typically lack: **resilience** (cursor-based recovery, idempotent upserts, dead letter queue), **security** (isolated credentials per source, signed identity propagation, OBO delegation for user-scoped APIs), and **extensibility** (uniform four-method connector interface means adding a new source is a self-contained deployment, not a pipeline rewrite).

The choice of Antfly as the vector store is load-bearing — its native hybrid BM25+vector search, async enrichment pipeline, and linear merge capability are not incidental. They directly remove the need to orchestrate embedding API calls in the MCP layer, handle re-indexing manually, or implement custom score fusion. The MCP federation layer handles source diversity and auth; Antfly handles storage, search, and enrichment. The boundary is clean.

The architecture is deployable today on a single VM with Docker Compose (development) and graduates to GKE with Cloud Run, Linkerd mTLS, and GCP Secret Manager (production) without a rewrite — only the deployment configuration changes.

The unresolved questions (deletion propagation, multi-tenant isolation, embedding model versioning) are real gaps that must be addressed before production deployment in a regulated or multi-tenant environment. They are sequenced as follow-up RFCs rather than blockers to the core architecture decision.
