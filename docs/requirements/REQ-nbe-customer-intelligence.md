# Requirements: Customer Intelligence & Next-Best-Engagement (NBE)

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | N/A (Source requirement) |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Overview

When an insurance agent opens a customer profile, the system should proactively generate an intelligent engagement plan — including prioritized recommendations, conversation openers, objection handling, and urgency drivers — by combining cloud-hosted product knowledge with real-time customer data from on-premises systems.

This is the flagship use case for Pattern 3 Hybrid Architecture, replacing the current reactive AI Rose MVP (brochure lookup only) with proactive agentic intelligence.

---

## Requirements

### REQ-NBE-001: Proactive Engagement Plan Generation
**Priority:** Must

When an agent opens a customer profile, the system must automatically generate a prioritized engagement plan without the agent having to ask a question.

**Acceptance Criteria:**
- Engagement plan is triggered on customer profile open
- Plan is displayed within 3 seconds of profile load
- Includes: top 3 recommended actions ranked by priority, conversation openers for each, objection handling tips, urgency drivers
- Agent can dismiss or regenerate the plan

---

### REQ-NBE-002: Parallel Data Retrieval
**Priority:** Must

The system must retrieve data from both cloud-hosted knowledge bases and on-premises customer systems in parallel to minimize latency.

**Acceptance Criteria:**
- Cloud RAG retrieval (product docs, UW guidelines, playbooks, competitor intel) executes in parallel with on-prem API calls (profile, policies, interactions, claims, life events)
- Combined retrieval completes within 120ms (target)
- Failure in one path does not block the other — graceful degradation with partial results

**Technical Notes:** _Cloud RAG via OpenSearch Serverless. On-prem APIs via AWS Direct Connect (1-5ms latency). Both assembled in the orchestrator._

---

### REQ-NBE-003: Context Assembly with Provenance
**Priority:** Must

The orchestrator must assemble retrieved data into a structured prompt for the LLM, with clear provenance labels so the model (and the agent) can distinguish where each piece of information came from.

**Acceptance Criteria:**
- RAG-sourced content labelled: `[PRODUCT]`, `[UNDERWRITING]`, `[PLAYBOOK]`, `[COMPETITOR]`
- API-sourced content labelled: `[PROFILE]`, `[PORTFOLIO]`, `[HISTORY]`, `[CLAIMS]`, `[LIFE EVENTS]`
- LLM prompt includes provenance labels
- Agent-facing output shows source tags on each recommendation

---

### REQ-NBE-004: Recommendation Intelligence
**Priority:** Must

The LLM must cross-reference product knowledge with the customer's specific situation to generate relevant, actionable recommendations — not generic suggestions.

**Acceptance Criteria:**
- Recommendations reference specific products applicable to the customer's profile
- Life events (marriage, new child, retirement approaching) are used as urgency drivers
- Existing coverage gaps are identified by comparing portfolio against needs
- Competitor product comparisons are included where relevant

---

### REQ-NBE-005: Handling Stale or Missing Data
**Priority:** Should

If certain data sources are unavailable or contain outdated information, the system must clearly indicate this rather than generating recommendations based on incomplete data.

**Acceptance Criteria:**
- If an API call fails, the engagement plan is still generated with available data
- Missing data sources are flagged (e.g., "Claims history unavailable — plan generated without claims context")
- Data freshness indicators shown where relevant (e.g., "Profile last updated: 3 months ago")

---

### REQ-NBE-006: Agent Feedback Loop
**Priority:** Could

Agents should be able to rate recommendations as helpful or not, to improve future suggestion quality.

**Acceptance Criteria:**
- Thumbs up/down on each recommendation
- Feedback stored for future model fine-tuning analysis
- Does not block or slow down the engagement plan flow

---

## Out of Scope (v1)
- Automated policy issuance from recommendations
- Real-time market data integration (e.g., investment-linked product pricing)
- Multi-customer batch engagement plans
- Customer self-service (agent-facing only)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft | Tech Architect |
