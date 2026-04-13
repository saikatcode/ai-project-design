# Architecture: Voice-to-Text Meeting Assistant Flow

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-VOICE-001, REQ-VOICE-002, REQ-VOICE-003, REQ-VOICE-004, REQ-VOICE-005, REQ-VOICE-006 |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## High-Level Flow

```mermaid
sequenceDiagram
    participant Agent as Agent Mobile (Pega Cosmos)
    participant Pega as Pega Case Manager
    participant Axway as Axway API Gateway
    participant Orch as Orchestrator
    participant STT as STT Service (AWS Transcribe)
    participant LLM as Bedrock Claude
    participant Advisor as Advisor Platform

    Agent->>Agent: Record meeting (local storage)
    Agent->>Pega: Upload audio (when online)
    Pega->>Pega: Create case with audio attachment
    Pega->>Axway: Submit transcription request
    Axway->>Orch: Route to orchestrator
    Orch->>STT: Send audio for transcription
    STT-->>Orch: Transcript with diarization + timestamps
    Orch->>Orch: Construct summarization prompt
    Orch->>LLM: Generate structured summary
    LLM-->>Orch: Meeting summary
    Orch-->>Axway: Return transcript + summary
    Axway-->>Pega: Update case with results
    Pega-->>Agent: Display summary for review
    Agent->>Pega: Approve/edit summary
    Pega->>Advisor: Push approved summary + action items
```

---

## Offline Flow

```mermaid
stateDiagram-v2
    [*] --> Recording: Agent taps Record
    Recording --> LocalStorage: Audio saved locally
    LocalStorage --> Queued: Recording complete
    Queued --> Uploading: Network available
    Queued --> Queued: No network (retry)
    Uploading --> CaseCreated: Pega case persistence
    CaseCreated --> Processing: Transcription + summarization
    Processing --> ReviewReady: Results available
    ReviewReady --> Approved: Agent reviews
    Approved --> Pushed: Sent to Advisor Platform
```

---

## STT Provider Evaluation

<!-- TODO: Populate with evaluation results -->

| Criteria | AWS Transcribe | Whisper | Deepgram | AssemblyAI |
|----------|---------------|---------|----------|------------|
| Cost per minute | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| Diarization quality | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| Chinese/Malay support | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| SG data residency | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| Scale (10M mins/mo) | _TBD_ | _TBD_ | _TBD_ | _TBD_ |

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft with sequence and offline flow | Tech Architect |
