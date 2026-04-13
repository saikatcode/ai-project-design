# API Contract: Voice-to-Text Orchestrator Endpoints

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-VOICE-002, REQ-VOICE-004, REQ-VOICE-005, ARCH-voice-meeting-flow |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Base URL

```
https://api.internal.insurance.com/orchestrator/v1
```

---

## Endpoints

### POST /voice/transcribe

Submits a recorded audio file for transcription and summarization.

**Request:**
```
Content-Type: multipart/form-data

Fields:
  - audio_file: (binary) audio file (WAV, MP3, M4A)
  - agent_id: "A12345"
  - customer_id: "C98765"
  - meeting_type: "client_review" | "prospecting" | "claims_discussion"
  - language_hint: "en" | "en-zh" | "en-ms" (optional, for mixed-language)
```

**Response (202 Accepted):**
```json
{
  "job_id": "job-uuid-v4",
  "status": "processing",
  "estimated_completion_seconds": 120
}
```

---

### GET /voice/transcribe/{job_id}

Polls for transcription/summarization status and results.

**Response (200 OK — completed):**
```json
{
  "job_id": "job-uuid-v4",
  "status": "completed",
  "transcript": {
    "segments": [
      {
        "speaker": "Agent",
        "text": "Thank you for coming in today. Let's review your current coverage.",
        "start_time": 0.0,
        "end_time": 4.2,
        "language": "en",
        "confidence": 0.94
      },
      {
        "speaker": "Client",
        "text": "Yes, I wanted to discuss...",
        "start_time": 4.5,
        "end_time": 8.1,
        "language": "en",
        "confidence": 0.91
      }
    ],
    "duration_seconds": 1823,
    "speaker_count": 2
  },
  "summary": {
    "key_topics": ["Coverage review", "CI rider interest", "Budget concerns"],
    "client_concerns": ["Monthly premium affordability", "Existing coverage overlap"],
    "product_interests": ["Critical illness rider", "Investment-linked plan"],
    "agreed_next_steps": ["Send CI rider illustration", "Schedule follow-up in 2 weeks"],
    "action_items": [
      {
        "owner": "agent",
        "action": "Prepare CI rider illustration for customer",
        "due": "2026-04-12"
      }
    ]
  },
  "metadata": {
    "transcription_time_ms": 45000,
    "summarization_time_ms": 3200,
    "stt_provider": "aws_transcribe",
    "model_used": "anthropic.claude-3-sonnet"
  }
}
```

**Response (200 OK — processing):**
```json
{
  "job_id": "job-uuid-v4",
  "status": "processing",
  "progress_percent": 65
}
```

---

### POST /voice/summary/{job_id}/approve

Agent approves the summary and triggers push to Advisor Platform.

**Request:**
```json
{
  "agent_id": "A12345",
  "edits": {
    "action_items": [
      {
        "owner": "agent",
        "action": "Prepare CI rider AND ILP illustration",
        "due": "2026-04-11"
      }
    ]
  }
}
```

**Response (200 OK):**
```json
{
  "status": "approved_and_pushed",
  "advisor_platform_reference": "AP-REF-12345"
}
```

---

## Authentication & Headers

Same as Chatbot API contract. See `API-chatbot-orchestrator-endpoints.md`.

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft — transcribe, poll, approve endpoints | Tech Architect |
