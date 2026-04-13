# Architecture: Chatbot Orchestrator Flow

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-CHAT-001, REQ-CHAT-002, REQ-CHAT-003, REQ-CHAT-004, REQ-CHAT-005 |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## High-Level Flow

```mermaid
sequenceDiagram
    participant Agent as Agent (Pega UI)
    participant Pega as Pega Platform
    participant Axway as Axway API Gateway
    participant Orch as Custom Orchestrator
    participant RAG as OpenSearch (Cloud RAG)
    participant APIs as On-Prem APIs
    participant LLM as Bedrock Claude

    Agent->>Pega: Sends message
    Pega->>Pega: Intent classification + slot extraction
    
    alt Simple FAQ
        Pega->>Pega: Knowledge Buddy RAG lookup
        Pega-->>Agent: FAQ answer with source
    else Complex Query
        Pega->>Axway: Forward query + extracted slots
        Axway->>Orch: Route to orchestrator
        
        par Parallel Retrieval
            Orch->>RAG: Retrieve relevant docs
            RAG-->>Orch: Product/UW/Playbook chunks
        and
            Orch->>APIs: Fetch customer data
            APIs-->>Orch: Profile/policies/claims
        end
        
        Orch->>Orch: Assemble context with provenance labels
        Orch->>Orch: Construct prompt (RAG chunks + customer data + conversation history)
        Orch->>LLM: Send structured prompt
        LLM-->>Orch: Generated response
        Orch->>Orch: Post-process (guardrails check, source attribution)
        Orch-->>Axway: Return response
        Axway-->>Pega: Return response
        Pega-->>Agent: Display answer with source tags
    end
```

---

## Component Responsibilities

<!-- TODO: Expand each component with detailed responsibilities, error handling, and scaling characteristics -->

### Pega Platform
- _To be detailed_

### Orchestrator
- _To be detailed_

### OpenSearch Cloud RAG
- _To be detailed_

### On-Prem APIs
- _To be detailed_

### Bedrock Claude
- _To be detailed_

---

## Design Decisions

<!-- Link to specific ADRs as they are created -->
- Routing: See `ADR-001-orchestrator-owns-rag-logic.md`
- _More to be added_

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft with sequence diagram | Tech Architect |
