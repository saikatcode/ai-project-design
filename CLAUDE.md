# AI Platform — Claude Code Project Instructions

## Who You Are Working With
- Tech Architect at a Singapore-based insurance company
- Responsible for AI roadmap across multiple projects
- Strong in distributed systems, Java/Spring Boot, API design, enterprise platforms (Pega, AEM), cloud (AWS)
- Currently ramping up in LLMs, RAG, embeddings, and AI architecture patterns

---

## Project Portfolio

### Project 1 — AI Chatbot for Insurance Agents
Hybrid architecture: Pega (UI, intent classification, slot filling, Knowledge Buddy RAG for FAQs) + Custom Orchestrator (API orchestration, business logic, prompt construction, LLM interaction). All calls via Axway API gateway. AWS via landing zone.

### Project 2 — Voice-to-Text Meeting Assistant
Mobile app (Pega Theme Cosmos) → Orchestrator → AWS Transcribe → AI summarization → Advisor Platform. Evaluating STT providers at ~10M mins/month. Offline support via Pega case persistence.

### Project 3 — Customer Intelligence & Next-Best-Engagement (NBE)
Proactive engagement plan when agent opens customer profile. Parallel retrieval: Cloud RAG (product knowledge, UW rules, playbooks, competitor intel) + on-prem APIs (profile, policies, interactions, claims, life events). LLM cross-references both for prioritized recommendations. Pattern 3 Hybrid Architecture.

### Personal Project — RAG Chatbot MVP
Spring Boot + Bedrock/OpenAI + OpenSearch vector DB. Hands-on AI architecture learning.

---

## Architectural Principles (NON-NEGOTIABLE)

1. **Orchestrator Owns Intelligence**: The orchestrator layer owns prompt engineering, RAG retrieval logic, context assembly, and LLM switching flexibility. Never delegate prompt construction or retrieval logic to downstream AI team APIs.
2. **API-First via Axway**: All service-to-service communication goes through the Axway internal API gateway. No direct service calls.
3. **Controlled AWS Access**: All AWS resources accessed via the landing zone. No direct provisioning outside the landing zone.
4. **Clear Layer Separation**: Pega = UI + intent classification. Orchestrator = business logic + RAG + prompt engineering. AI Team = model hosting + embedding generation. Cloud = compute + storage.
5. **Cost & Scalability Awareness**: Every design decision must consider enterprise scale. Include cost implications and scaling characteristics.
6. **Pattern 3 Hybrid Architecture**: Cloud RAG (OpenSearch Serverless, nightly sync) for static content + Real-Time API Federation via AWS Direct Connect for live customer data. Bedrock Claude via PrivateLink.
7. **Data Classification**: Tier 1 (Vector DB only) → Tier 2 (Dual-path) → Tier 3 (API only). Never embed raw transcripts, raw customer records, or pricing tables.
8. **Security Baseline**: TLS 1.3, SSE-KMS with CMKs, mTLS/OAuth 2.0 between cloud and on-prem, Bedrock Guardrails for content filters and PII redaction.

---

## Repository Structure

```
ai-platform/
├── CLAUDE.md                        ← YOU ARE HERE — project rules
├── docs/                            ← SPACE 1: Source of truth (approved artifacts)
│   ├── requirements/                ← Layer 1: WHAT (plain English, business intent)
│   ├── architecture/                ← Layer 2: HOW (design, sequences, patterns)
│   ├── api-contracts/               ← Layer 3: INTERFACE (OpenAPI-style specs)
│   ├── risks-and-decisions/         ← Layer 4: WHY (ADRs, open risks, trade-offs)
│   └── test-data/                   ← Layer 5: VALIDATION (sample payloads, edge cases)
└── dev/                             ← SPACE 2: Code (generated from docs)
    └── orchestrator/
        ├── src/
        └── tests/
```

---

## Cascade Dependency Rules

Documents follow a strict dependency hierarchy. When an upstream document changes, all downstream documents MUST be reviewed and updated.

```
requirements/*.md          → Layer 1: Source requirements (plain English)
    ↓ drives
architecture/*.md          → Layer 2: Design decisions and diagrams
    ↓ drives
api-contracts/*.md         → Layer 3: API specifications
    ↓ drives
test-data/*.md             → Layer 4: Test payloads and scenarios
    ↓ drives
dev/orchestrator/          → Layer 5: Implementation code
```

### Cascade Rules

1. **Every non-requirement file MUST have a "Derived From" header** listing the requirement IDs it traces back to.
2. **When a requirement changes**, Claude MUST identify all downstream files that reference it and flag them for update.
3. **Never update a downstream artifact without first checking** if the upstream requirement still supports the change.
4. **When asked to cascade**, work layer by layer: requirements → architecture → api-contracts → test-data. Do NOT skip layers.
5. **Mark changed sections** with `<!-- UPDATED: YYYY-MM-DD | Reason: ... -->` comments so diffs are meaningful in Git.

---

## File Naming Conventions

- Requirements: `REQ-{project}-{topic}.md` (e.g., `REQ-chatbot-intent-routing.md`)
- Architecture: `ARCH-{project}-{topic}.md` (e.g., `ARCH-chatbot-orchestrator-flow.md`)
- API Contracts: `API-{project}-{endpoint-group}.md` (e.g., `API-chatbot-orchestrator-endpoints.md`)
- Risks/Decisions: `ADR-{number}-{short-title}.md` or `RISK-{project}-{topic}.md`
- Test Data: `TEST-{project}-{scenario}.md` (e.g., `TEST-chatbot-complex-query-payloads.md`)

---

## Document Template Rules

### Every document MUST include:
1. **Metadata header** with: Title, Status (Draft/Review/Approved), Derived From (requirement IDs), Last Updated, Author
2. **Change Log** section at the bottom tracking all modifications
3. **Mermaid diagrams** for any flow or sequence (use ```mermaid code blocks)

### Requirements documents specifically MUST include:
- Requirement ID (format: `REQ-{project}-{NNN}`)
- Plain English description (no technical jargon in the requirement itself)
- Acceptance criteria
- Priority (Must/Should/Could)
- "Technical Notes" section (optional, for architect's implementation hints)

---

## Behavioral Rules for Claude

### When generating or updating documents:
1. **Always read the relevant requirements file first** before generating any downstream artifact.
2. **Always check CLAUDE.md** at the start of every session for the latest rules and context.
3. **Preserve existing content** unless explicitly asked to replace it. Append and refine, don't overwrite.
4. **Use the provenance labels** from Pattern 3 architecture in all relevant docs: `[PRODUCT]`, `[UNDERWRITING]`, `[PLAYBOOK]`, `[COMPETITOR]` for RAG sources; `[PROFILE]`, `[PORTFOLIO]`, `[HISTORY]`, `[CLAIMS]`, `[LIFE EVENTS]` for API sources.
5. **Include cost/scale annotations** on architecture and API docs where relevant.

### When generating code (dev/ folder):
1. **Read the API contract first** — code MUST match the approved API spec.
2. **Java 17+ / Spring Boot 3.x** conventions.
3. **Package structure**: `com.insurance.orchestrator.{module}` — controller, service, client, model, config.
4. **Every endpoint MUST have** a corresponding integration test stub in `tests/`.
5. **Never hardcode LLM provider** — use a strategy/adapter pattern so Bedrock/OpenAI/others can be swapped.

### When discussing vs. documenting:
- If the user says "just exploring" or "thinking about" or "what if" — this is SANDBOX mode. Respond conversationally. Do NOT update any files.
- If the user says "update", "add to docs", "make it official" — this is DOCUMENTATION mode. Update the appropriate .md file.
- If ambiguous, ASK which mode before proceeding.

---

## Technology Stack Reference

| Layer | Technology | Notes |
|-------|-----------|-------|
| UI | Pega Theme Cosmos | Intent classification, slot filling, UI rendering |
| API Gateway | Axway | All internal API routing |
| Orchestrator | Spring Boot 3.x (Java 17+) | Core intelligence layer |
| LLM | AWS Bedrock (Claude) via PrivateLink | Primary model |
| Vector DB | OpenSearch Serverless | Titan Embeddings v2, 1536 dims |
| On-Prem APIs | REST via AWS Direct Connect | Customer data federation |
| Transcription | AWS Transcribe (evaluating alternatives) | Voice-to-text |
| Caching | Redis (Phase 2) | Hot data caching |
| Graph DB | Neptune (Phase 2) | Relationship queries |
| Cloud Platform | AWS (via Landing Zone) | All cloud resources |

---

## What NOT to Do

- ❌ Never put RAG retrieval logic inside the AI team's API — it stays in the orchestrator
- ❌ Never embed raw customer records, transcripts, or pricing tables in vector DB
- ❌ Never call AWS services directly — always through landing zone
- ❌ Never bypass Axway for service-to-service calls
- ❌ Never update docs/ files during sandbox/exploration discussions
- ❌ Never generate code that doesn't trace back to an approved API contract
- ❌ Never remove the "Derived From" or "Change Log" sections from any document
