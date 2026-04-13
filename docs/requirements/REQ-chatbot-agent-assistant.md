# Requirements: AI Chatbot for Insurance Agents

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | N/A (Source requirement) |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Overview

Insurance agents need an AI-powered chatbot that helps them quickly find product information, underwriting guidelines, and customer-specific answers during client conversations. The chatbot must handle both simple FAQ lookups and complex multi-step queries that require combining product knowledge with customer data.

---

## Requirements

### REQ-CHAT-001: Simple FAQ Handling
**Priority:** Must

Agents should be able to ask simple questions about products, policies, and procedures, and get accurate answers sourced from approved company knowledge bases.

**Acceptance Criteria:**
- Agent types a question in natural language
- System returns an answer within 3 seconds
- Answer includes the source document reference
- Covers product brochures, FAQ documents, and standard operating procedures

**Technical Notes:** _This is handled by Pega Knowledge Buddy RAG. Does not route to the custom orchestrator._

---

### REQ-CHAT-002: Complex Query Handling
**Priority:** Must

When a question requires combining product knowledge with business rules or customer-specific data (e.g., "Is this customer eligible for the critical illness rider given their medical history?"), the system must route to an intelligent orchestrator that can assemble context from multiple sources and generate a reasoned answer.

**Acceptance Criteria:**
- System detects that the query requires more than FAQ lookup
- Orchestrator retrieves relevant product docs AND customer data
- LLM generates an answer that cites both knowledge sources and customer context
- Response time under 5 seconds for 90th percentile

**Technical Notes:** _Routes from Pega → Axway → Custom Orchestrator → RAG + On-Prem APIs → LLM → Response._

---

### REQ-CHAT-003: Intent Classification and Routing
**Priority:** Must

The system must determine whether an agent's question is a simple FAQ (handled by Pega Knowledge Buddy) or a complex query (routed to the custom orchestrator). Misrouting should be less than 5%.

**Acceptance Criteria:**
- Pega performs intent classification on every incoming message
- Simple intents resolved locally within Pega
- Complex intents forwarded to orchestrator with extracted slots/entities
- Fallback mechanism if classification confidence is below threshold

---

### REQ-CHAT-004: Source Attribution
**Priority:** Must

Every answer must clearly indicate where the information came from, so agents can verify and cite sources when speaking to customers.

**Acceptance Criteria:**
- Each claim in the response is tagged with its source type: product doc, UW guideline, customer record, etc.
- Agent can click/tap to see the source document or data point
- No hallucinated sources — if the system cannot find a source, it must say so

---

### REQ-CHAT-005: Conversation Context
**Priority:** Should

The chatbot should maintain context within a session so agents don't have to repeat information across follow-up questions.

**Acceptance Criteria:**
- System retains conversation history for the current session
- Follow-up questions like "what about for the spouse?" are correctly interpreted
- Session context is cleared when agent explicitly starts a new conversation

---

### REQ-CHAT-006: Guardrails and Compliance
**Priority:** Must

The system must not provide financial advice, make coverage promises, or disclose sensitive customer data inappropriately.

**Acceptance Criteria:**
- Responses include appropriate disclaimers where required by regulation
- PII is redacted from LLM prompts where not necessary for the query
- System refuses to answer questions outside its approved scope
- All interactions are logged for audit

---

## Out of Scope (v1)
- Direct customer-facing chatbot (agents only for v1)
- Voice input (separate project: Voice-to-Text Meeting Assistant)
- Automated policy issuance or claims processing
- Multi-language support (English only for v1)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft | Tech Architect |
