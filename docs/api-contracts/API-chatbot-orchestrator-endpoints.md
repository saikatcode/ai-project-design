# API Contract: Chatbot Orchestrator Endpoints

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-CHAT-002, REQ-CHAT-003, REQ-CHAT-004, REQ-CHAT-005, ARCH-chatbot-orchestrator-flow |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Base URL

All endpoints are accessed via Axway API Gateway:
```
https://api.internal.insurance.com/orchestrator/v1
```

---

## Endpoints

### POST /chat/query

Handles a complex agent query that requires orchestration across RAG and on-prem APIs.

**Request:**
```json
{
  "session_id": "uuid-v4",
  "agent_id": "A12345",
  "customer_id": "C98765",
  "message": "Is this customer eligible for the critical illness rider?",
  "conversation_history": [
    {
      "role": "agent",
      "content": "Show me the customer's existing coverage",
      "timestamp": "2026-04-09T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "The customer currently has...",
      "timestamp": "2026-04-09T10:30:02Z"
    }
  ],
  "context": {
    "intent": "eligibility_check",
    "slots": {
      "product": "critical_illness_rider",
      "customer_id": "C98765"
    }
  }
}
```

**Response (200 OK):**
```json
{
  "session_id": "uuid-v4",
  "response": {
    "answer": "Based on the customer's profile and current medical declarations...",
    "sources": [
      {
        "type": "PRODUCT",
        "reference": "CI Rider Product Guide v3.2",
        "chunk_id": "chunk-1234",
        "relevance_score": 0.92
      },
      {
        "type": "UNDERWRITING",
        "reference": "UW Guidelines - CI Section 4.2",
        "chunk_id": "chunk-5678",
        "relevance_score": 0.88
      },
      {
        "type": "PROFILE",
        "reference": "Customer medical declarations",
        "data_timestamp": "2026-03-15T00:00:00Z"
      }
    ],
    "confidence": 0.85,
    "disclaimers": [
      "This assessment is based on available data. Final underwriting decision requires formal application."
    ]
  },
  "metadata": {
    "processing_time_ms": 2340,
    "model_used": "anthropic.claude-3-sonnet",
    "rag_chunks_retrieved": 5,
    "api_calls_made": 3
  }
}
```

**Error Responses:**

| Code | Scenario | Body |
|------|----------|------|
| 400 | Invalid request | `{"error": "INVALID_REQUEST", "message": "..."}` |
| 401 | Authentication failed | `{"error": "UNAUTHORIZED"}` |
| 408 | Timeout (>10s) | `{"error": "TIMEOUT", "partial_response": {...}}` |
| 500 | Internal error | `{"error": "INTERNAL_ERROR", "correlation_id": "..."}` |
| 503 | LLM unavailable | `{"error": "LLM_UNAVAILABLE", "fallback": "..."}` |

---

### POST /chat/feedback

Records agent feedback on a response for quality monitoring.

<!-- TODO: Define request/response after REQ-NBE-006 discussion -->

---

## Authentication

All requests must include:
- `X-API-Key`: Axway-issued API key
- `X-Correlation-ID`: UUID for request tracing
- `Authorization`: Bearer token (OAuth 2.0 client credentials)

---

## Rate Limits

| Tier | Requests/min | Burst |
|------|-------------|-------|
| Standard | 100 | 150 |
| Premium | 500 | 750 |

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft — /chat/query endpoint | Tech Architect |
