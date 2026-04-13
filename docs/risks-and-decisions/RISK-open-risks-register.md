# Open Risks Register

| Field | Value |
|-------|-------|
| **Status** | Living Document |
| **Derived From** | All REQ-* documents |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Active Risks

### RISK-001: STT Provider Lock-in and Mixed Language Accuracy
**Project:** Voice-to-Text Meeting Assistant
**Severity:** High | **Likelihood:** Medium

AWS Transcribe may not meet accuracy targets for Chinese-English code-switching common in Malaysia/Singapore agent meetings. Switching STT providers mid-project has cost and integration implications.

**Mitigation:** Abstract STT behind an adapter interface in the orchestrator. Conduct POC with all four providers (Transcribe, Whisper, Deepgram, AssemblyAI) before committing.

**Related:** REQ-VOICE-003, ARCH-voice-meeting-flow

---

### RISK-002: On-Prem API Latency Under Load
**Project:** NBE, Chatbot
**Severity:** High | **Likelihood:** Medium

The 120ms target for parallel retrieval depends on on-prem APIs responding within 80ms via Direct Connect. Legacy systems may not consistently meet this under load during peak hours (agent morning login surge).

**Mitigation:** Redis caching layer (Phase 2) for frequently accessed customer profiles. Circuit breaker pattern in orchestrator. Graceful degradation with partial results.

**Related:** REQ-NBE-002, REQ-CHAT-002

---

### RISK-003: Bedrock Claude Availability in ap-southeast-1
**Project:** All
**Severity:** Medium | **Likelihood:** Low

Bedrock model availability and capacity in Singapore region may face constraints during high-demand periods. PrivateLink adds network dependency.

**Mitigation:** Multi-model fallback strategy (Sonnet → Haiku for degraded responses). Prompt caching for 75-90% cost reduction also reduces token throughput pressure.

**Related:** ARCH-nbe-pattern3-hybrid

---

### RISK-004: Data Residency Compliance for Malaysia
**Project:** All
**Severity:** High | **Likelihood:** High

Malaysian agents' customer data may be subject to PDPA (Malaysia) requirements that restrict cross-border data transfer. Current architecture routes through Singapore AWS region.

**Mitigation:** Phase 3 multi-region deployment (MY/SG). Short term: ensure customer PII is redacted before LLM calls. Legal review needed.

**Related:** REQ-VOICE-007

---

### RISK-005: Pega Intent Classification Accuracy
**Project:** Chatbot
**Severity:** Medium | **Likelihood:** Medium

If Pega's intent classifier misroutes more than 5% of queries, agents will have poor experience — simple FAQs getting slow orchestrator responses, or complex queries getting shallow Knowledge Buddy answers.

**Mitigation:** Implement confidence threshold with fallback. Log misroutes for continuous tuning. Consider dual-path: always return Knowledge Buddy answer while orchestrator processes in background.

**Related:** REQ-CHAT-003

---

## Closed Risks

_None yet._

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial 5 risks documented | Tech Architect |
