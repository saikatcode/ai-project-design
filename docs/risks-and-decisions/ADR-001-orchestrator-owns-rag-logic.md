# ADR-001: Orchestrator Owns RAG Retrieval and Prompt Logic

| Field | Value |
|-------|-------|
| **Status** | Approved |
| **Derived From** | REQ-CHAT-002, REQ-NBE-002, REQ-NBE-003 |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Context

The AI team proposed exposing a `/retrieveChunks` API that would handle RAG retrieval and return pre-assembled chunks. The orchestrator would then pass these to the LLM. An alternative is to have the orchestrator own the full RAG pipeline: embedding the query, calling OpenSearch directly, ranking/filtering results, and constructing the prompt.

## Decision

**The orchestrator owns RAG retrieval and prompt construction.** The AI team provides model hosting (Bedrock) and embedding generation endpoints, but does not own retrieval logic.

## Rationale

1. **Prompt-retrieval coupling**: The way you retrieve chunks depends on what the prompt needs. Decoupling them creates an abstraction that leaks constantly.
2. **LLM switching flexibility**: If we change LLM providers, the prompt structure changes, which changes what context is needed, which changes retrieval strategy. Owning all three in one layer avoids cross-team dependencies.
3. **Metadata filtering**: The orchestrator knows the business context (jurisdiction, product line, customer segment) and can apply metadata filters at retrieval time. A generic `/retrieveChunks` API would need to accept increasingly complex filter parameters.
4. **Latency control**: The orchestrator can run RAG retrieval in parallel with on-prem API calls. A separate API adds an extra network hop.
5. **Debugging**: When an answer is wrong, having retrieval + prompt + LLM call in one service makes root-cause analysis straightforward.

## Consequences

- The orchestrator team must understand OpenSearch query DSL and embedding semantics
- The AI team's scope is narrower: model hosting, embedding generation, guardrails configuration
- The orchestrator has a direct dependency on OpenSearch Serverless

## Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| AI team owns `/retrieveChunks` | Leaky abstraction, extra hop, limits prompt flexibility |
| Shared library for RAG logic | Versioning across teams, language mismatch (AI team uses Python, orchestrator uses Java) |

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial decision documented | Tech Architect |
