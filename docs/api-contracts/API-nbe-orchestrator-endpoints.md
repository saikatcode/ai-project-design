# API Contract: NBE Orchestrator Endpoints

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-NBE-001, REQ-NBE-002, REQ-NBE-003, ARCH-nbe-pattern3-hybrid |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Base URL

```
https://api.internal.insurance.com/orchestrator/v1
```

---

## Endpoints

### POST /nbe/engagement-plan

Generates a proactive engagement plan when an agent opens a customer profile.

**Request:**
```json
{
  "agent_id": "A12345",
  "customer_id": "C98765",
  "trigger": "profile_open",
  "preferences": {
    "max_recommendations": 3,
    "include_competitor_comparison": true,
    "language": "en"
  }
}
```

**Response (200 OK):**
```json
{
  "customer_id": "C98765",
  "plan_id": "plan-uuid-v4",
  "generated_at": "2026-04-09T10:30:02Z",
  "recommendations": [
    {
      "rank": 1,
      "action": "Discuss critical illness coverage gap",
      "reasoning": "Customer has dependents but no CI coverage. Recent life event: second child born.",
      "conversation_opener": "Congratulations on the new addition to your family! Have you thought about how your family would be protected if...",
      "objection_handling": [
        {
          "objection": "I already have enough coverage",
          "response": "Your current coverage of $X covers Y, but with two dependents now..."
        }
      ],
      "urgency_drivers": [
        "New child born 2 months ago",
        "Premium increases at next age bracket in 4 months"
      ],
      "sources": [
        {"type": "PRODUCT", "reference": "CI Protect Plan v2.1"},
        {"type": "PROFILE", "reference": "Customer demographics"},
        {"type": "LIFE EVENTS", "reference": "Birth event - 2026-02-15"},
        {"type": "PORTFOLIO", "reference": "Existing policy P-44521"}
      ]
    }
  ],
  "data_gaps": [
    {
      "source": "CLAIMS",
      "status": "unavailable",
      "message": "Claims API timed out. Plan generated without claims context."
    }
  ],
  "metadata": {
    "processing_time_ms": 1850,
    "model_used": "anthropic.claude-3-sonnet",
    "rag_retrieval_ms": 48,
    "api_federation_ms": 72,
    "prompt_tokens": 3200,
    "completion_tokens": 890
  }
}
```

---

### POST /nbe/feedback

Records agent feedback on engagement plan quality.

**Request:**
```json
{
  "plan_id": "plan-uuid-v4",
  "agent_id": "A12345",
  "recommendation_rank": 1,
  "rating": "helpful",
  "comment": "Customer was receptive to the CI discussion"
}
```

**Response (202 Accepted):**
```json
{
  "feedback_id": "fb-uuid-v4",
  "status": "recorded"
}
```

---

## Authentication & Headers

Same as Chatbot API contract. See `API-chatbot-orchestrator-endpoints.md`.

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft — /nbe/engagement-plan endpoint | Tech Architect |
