# SEQ-001: Meeting Summary Generation Flow

**Version**: 1.3
**Date**: 2026-04-21
**Status**: DRAFT

---

## Participants

| Alias | Full Name |
|---|---|
| Agent | Insurance Agent (end user) |
| Mobile | GP Mobile App |
| GP | GP Cloud (Backend) |
| S3 | S3 — GP AWS Account |
| Orch | Orchestrator (PCF Tanzu / Spring Boot) |
| AI | Group Data/AI API Layer |
| Transcribe | AWS Transcribe |
| LLM | LLM (AI Team hosted) |
| DB | GP Database |
| CDHAPI | CDH API |
| CDHDB | CDH Database |

---

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Agent
    participant Mobile as GP Mobile App
    participant GP as GP Cloud (Backend)
    participant S3 as S3 (GP AWS)
    participant Orch as Orchestrator
    participant AI as Group Data/AI API
    participant Transcribe as AWS Transcribe
    participant LLM as LLM
    participant DB as GP Database
    participant CDHAPI as CDH API
    participant CDHDB as CDH Database

    rect rgb(220, 235, 255)
        Note over Agent,Mobile: Phase 1 — Recording
        Agent->>Mobile: Start recording
        Agent->>Mobile: Stop recording
        Note over Mobile: Audio file saved to device memory
    end

    rect rgb(220, 255, 225)
        Note over Mobile,AI: Phase 2 — Upload & Transcription Trigger
        Mobile->>GP: POST /meeting/initiate {meetingId, agentId}
        GP->>S3: PUT audio file
        S3-->>GP: 200 OK · s3AudioUrl
        GP->>AI: POST /transcript {interactionId, s3AudioUrl}
        AI-->>GP: 202 Accepted
    end

    rect rgb(255, 248, 220)
        Note over AI,LLM: Phase 3 — Transcription & Snippet Generation (AI Team — async)
        AI->>Transcribe: StartTranscriptionJob {s3AudioUrl}
        loop Poll until COMPLETED
            AI->>Transcribe: GetTranscriptionJob {jobId}
            Transcribe-->>AI: status IN_PROGRESS
        end
        Transcribe-->>AI: status COMPLETED · raw items[] + speaker_labels[]
        Note over AI: Reconstruct diarized transcript from items[] + speaker_labels[]
        AI->>LLM: POST /llm/snippet {transcript, interactionId}
        LLM-->>AI: snippet response (SNIPPET blocks)
    end

    rect rgb(245, 225, 255)
        Note over AI,Mobile: Phase 4 — Result Delivery & Speaker Tagging
        AI->>GP: POST /callback/transcription {interactionId, transcript, snippet}
        GP->>DB: Save transcript + snippet keyed by interactionId
        DB-->>GP: OK
        GP->>Mobile: Push notification — "Snippet ready, please tag speakers"
        Agent->>Mobile: Open app · review snippet
        Agent->>Mobile: Assign customer name(s) to speaker label(s) (min. 1)
        Mobile->>GP: POST /meeting/tag-speakers {interactionId, speakerTags}
        GP->>DB: Save speaker tags
        DB-->>GP: OK
    end

    rect rgb(255, 255, 220)
        Note over GP,LLM: Phase 5 — Full Summary Generation
        GP->>Orch: POST /orchestrate/summary {interactionId, customerId, speakerTags}
        Orch->>GP: GET /enquiry/transcript {interactionId}
        GP-->>Orch: transcript · agentInfo
        Orch->>GP: GET /enquiry/customer-crm {customerId}
        GP-->>Orch: leads · opportunities · campaigns
        Orch->>CDHAPI: GET /customer-context {customerId}
        CDHAPI-->>Orch: customer profile · past summaries · products
        Note over Orch: Assemble full LLM request (FULL_SUMMARY)
        Orch->>AI: POST /llm/meeting-summary {FULL_SUMMARY, transcript, speakerTags, customerContext}
        AI->>LLM: Generate meeting summary
        LLM-->>AI: summary blocks + tasks[] + FLP/soft attributes
        AI-->>Orch: 200 OK · summary response
        Orch-->>GP: summary blocks · tasks · FLP attributes · soft attributes
        GP->>DB: Save draft summary + tasks + attributes
        DB-->>GP: OK
        GP->>Mobile: Push notification · render draft summary in UI
    end

    rect rgb(255, 235, 225)
        Note over Agent,CDHDB: Phase 6 — Review, Edit & Save to CDH
        loop Until agent saves
            Agent->>Mobile: Review and edit summary · tasks · FLP/soft attributes
        end
        Agent->>Mobile: Save
        Mobile->>GP: POST /meeting/summary/save {interactionId, editedSummary, tasks, FLPAttributes, softAttributes}
        GP->>CDHAPI: POST /cdh/meeting-output {interactionId, finalSummary, tasks, FLPAttributes, softAttributes}
        CDHAPI->>CDHDB: Persist all reviewed outputs
        CDHDB-->>CDHAPI: OK
        CDHAPI-->>GP: 200 OK
        GP->>DB: Delete draft
        DB-->>GP: OK
    end

    opt Agent requests sharable executive summary for customer
        Note over Agent,LLM: Phase 7 — Sharable Executive Summary (Optional)
        Agent->>Mobile: Request executive summary
        Mobile->>GP: POST /meeting/summary/share-request {interactionId}
        GP->>Orch: POST /orchestrate/sharable-summary {interactionId, customerId, editedSummaryText}
        Orch->>CDHAPI: GET /customer-profile {customerId}
        CDHAPI-->>Orch: customer name · key profile details
        Note over Orch: Assemble LLM request (SHARABLE_SUMMARY)
        Orch->>AI: POST /llm/meeting-summary {SHARABLE_SUMMARY, editedSummaryText, customerProfile}
        AI->>LLM: Generate executive summary
        LLM-->>AI: sharable summary blocks
        AI-->>Orch: 200 OK
        Orch-->>GP: sharable summary blocks
        GP->>Mobile: Render executive summary in UI
        Agent->>Mobile: Review · share with customer (WhatsApp / email)
        Mobile->>GP: POST /meeting/summary/share-confirm {interactionId, sharableSummary}
        GP->>CDHAPI: POST /cdh/sharable-summary {interactionId, sharableSummary}
        CDHAPI->>CDHDB: Persist sharable summary
        CDHDB-->>CDHAPI: OK
        CDHAPI-->>GP: 200 OK
    end
```

---

## Phase Summary

| Phase | Trigger | Owner | Async? |
|---|---|---|---|
| 1 — Recording | Agent action | Agent / Mobile | No |
| 2 — Upload & Trigger | Agent stops recording | GP Backend | No |
| 3 — Transcription + Snippet | Transcript API called | AI Team | Yes (polling) |
| 4 — Delivery & Tagging | AI team callback | GP Backend | Push notification |
| 5 — Full Summary | Agent submits speaker tags | GP → Orchestrator → AI | Near-real-time |
| 6 — Edit & CDH Sync | Agent saves final summary | Agent / GP → CDH | No |
| 7 — Sharable Summary | Agent requests (optional) | GP → Orchestrator → AI | No |

---

## Key Design Notes

- **Transcript reconstruction** is performed by the AI team inside their pipeline (steps 8–9). GP receives a clean diarized string — it has no dependency on AWS Transcribe's raw JSON schema.
- **Snippet is generated by the AI team** (step 10) as part of the same async pipeline, bundled with the transcript in the callback to GP.
- **Orchestrator owns context assembly** — GP triggers Orchestrator with minimal parameters (interactionId, customerId, speakerTags). Orchestrator fetches from two sources: CDH API (customer profile, past summaries, products) and GP enquiry APIs (transcript, agent info, leads, opportunities, campaigns).
- **Persistence is split across two paths**: (1) GP DB — draft summary, tasks, and attributes temporarily while agent reviews (deleted after CDH save confirms); (2) CDH via GP → CDH API — all reviewed outputs (final summary, tasks, FLP attributes, soft attributes, sharable summary), gated by agent save/confirm action (human in the loop).
- **Orchestrator has no persistence responsibility in this flow** — it is a pure computation layer. It assembles context, calls the AI team, and returns results to GP. Direct Orchestrator → CDH persistence is reserved for chatbot conversation management (e.g., conversation compaction summaries) which is out of scope for this diagram.
- **Orchestrator is bypassed for Phases 1–4** — the async transcription and snippet pipeline (AI team) does not involve the Orchestrator. GP handles the callback and temporary storage directly.
- **GP has no direct CDH DB access** — all CDH interactions go through CDH API.
- **Speaker tagging** happens after snippet delivery and before full summary generation. The snippet uses `[spk_0]`, `[spk_1]` labels; the agent maps these to real customer names.
- **CDH sync** occurs only after the agent explicitly saves the final (edited) summary — not on draft save.
- **Sharable summary** input is `editedSummaryText` — the agent's final reviewed version, not the raw LLM output.

---

## Related Documents

- [ADR-002: Transcript Reconstruction](../risks-and-decisions/ADR-002-transcript-reconstruction.md)
- [API Contract: POST /llm/meeting-summary](../api-contracts/API-llm-meeting-summary-endpoint.md)
