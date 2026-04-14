# Intent Analysis

## Request Classification

| Dimension | Assessment |
|-----------|-----------|
| **Request Type** | New Product / Platform (Greenfield) |
| **Clarity** | Moderate — rich design context provided but explicitly marked as draft/not final |
| **Scope** | System-wide — multiple components, three distinct use cases, two team boundaries |
| **Complexity** | Complex — multi-layer AI platform, external API dependencies, regulatory context, production insurance |

## Requirements Depth: COMPREHENSIVE

Rationale:
- Production-grade, regulated insurance platform (MAS/BNM applicable)
- Multiple use cases with distinct flows and data requirements
- Two-team ownership boundary (UI/Orchestration vs AI/data team)
- Security Baseline enforced as blocking constraints
- Multiple open design decisions requiring validation

## What the Context File Provides (Clear)

- Three use cases: Meeting Prep, Meeting Summary, Agent Chatbot
- API strategy: 5 endpoints (meeting-prep, meeting-summary, chat, retrieve, llm/query)
- Orchestrator ownership model and responsibilities
- Intent handling hybrid approach (fixed journeys + registry-driven chatbot)
- Response envelope design (contentBlocks, citations, usage)
- Correlation identifier strategy (requestId, interactionId, conversationId)
- Retrieval/LLM separation principle
- Guardrails layering approach
- Intent registry — 18 intents across 3 use cases
- Sequence diagrams for all 3 flows
- Technology stack notes (Pega, Java/Spring Boot, enterprise observability)

## What Is Missing or Ambiguous (Requires Clarification)

| Area | Gap |
|------|-----|
| **Release scope** | Which use case(s) are in scope for first release? |
| **Performance/latency** | No response time targets defined per use case |
| **Language support** | SG/MY context implies multilingual — not specified |
| **Authentication** | How Pega authenticates to the orchestrator — not specified |
| **Async strategy** | Transcription polling/callback approach for meeting summary not finalized |
| **Regulatory constraints** | MAS/BNM applicability not confirmed |
| **Data residency** | Where LLM/vector data can be stored/processed — not specified |
| **Intent registry governance** | Who owns and deploys the registry |
| **Orchestrator tech stack** | Java/Spring Boot mentioned but not confirmed |
| **Pega → Orchestrator integration** | Direct call vs API gateway (Axway T2 mentioned but not confirmed) |
| **Error/retry strategy** | Orchestrator-level fallback on timeout/partial data — not designed |
| **Caching strategy** | Mentioned as desirable but not designed |
| **Availability/SLA** | No uptime or degraded-mode requirements |
| **Compliance review** | Process for approving generated insurance guidance — not specified |
| **Knowledge ingestion pipeline** | SharePoint-based mentioned but not designed |
