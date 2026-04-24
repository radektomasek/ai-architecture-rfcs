# AI Architecture RFCs

Architecture decision records exploring how to build AI systems that actually work in production. Topics include MCP server federation, RAG ingestion pipelines, service-to-service auth, and multi-agent coordination patterns.

Not tutorials. Not demos. The design decisions that matter when systems need to scale, recover, and satisfy an enterprise security team.

---

## What is an RFC?

Each document in this repo is a **Request for Comments** — a structured proposal that captures not just the recommended approach but the reasoning behind it: motivations, tradeoffs, alternatives considered, and open questions.

RFCs are useful because they force precise thinking. Writing a proposal that includes drawbacks and unresolved questions produces better architecture than one that only argues for its own conclusions.

These RFCs are written from practical experience building and deploying AI-native systems. They are intended as living references — updated as the field evolves and new patterns emerge.

---

## RFCs

| # | Title | Topics | Status |
|---|-------|--------|--------|
| 001 | [MCP Federation & Service-to-Service Authentication](rfcs/001-mcp-federation-auth.md) | MCP topology, OAuth2, token delegation, OBO, mTLS | draft |
| 002 | [RAG Ingestion Pipeline with MCP and Antfly](rfcs/002-rag-pipeline-antfly.md) | RAG, source connectors, chunking, vector search, recovery | draft |

---

## Themes

These RFCs are organized around a few recurring concerns that production AI systems consistently run into:

**Federation and orchestration** — how to decompose a monolithic AI tool surface into independently deployable services, and how to coordinate them without tight coupling.

**Authentication at every layer** — the difference between human-to-service auth, service-to-service auth, and credential management for backend APIs. Why these are three separate problems and why conflating them is the most common enterprise auth mistake.

**Resilience and recoverability** — cursors, job state machines, dead letter queues, idempotent writes. The infrastructure that makes a pipeline recoverable from any failure point without data loss or duplication.

**Identity propagation** — how user identity survives a multi-hop service chain, why it matters for audit trails, and how OAuth2 On-Behalf-Of delegation works in practice.

---

## Structure

```
ai-architecture-rfcs/
  README.md
  rfcs/
    001-mcp-federation-auth.md
    002-rag-pipeline-antfly.md
    ...
```

Each RFC follows a consistent template:

1. Executive Summary
2. Motivation
3. Proposed Implementation
4. Metrics & Dashboards
5. Drawbacks
6. Alternatives
7. Potential Impact and Dependencies
8. Unresolved Questions
9. Conclusion

---

## Background

These documents emerged from hands-on work building production AI systems — MCP server implementations, RAG pipelines, agent orchestration, and enterprise integrations. The goal is not to document what vendors claim their products do, but to capture the architectural decisions that come up when you're actually building these systems and need them to work reliably.

The field is moving fast. Some of these patterns will be superseded. The unresolved questions in each RFC are honest — they reflect genuine open problems, not gaps that were overlooked.

---

## Contributing

Feedback, corrections, and alternative perspectives are welcome via issues or pull requests. If something in an RFC is wrong, underspecified, or ignores a better approach — open an issue. The goal is accurate, useful architecture guidance, not defended positions.

---

*Written by [@radektomasek](https://github.com/radektomasek)*
