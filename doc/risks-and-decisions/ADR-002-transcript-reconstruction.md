# ADR-002: Transcript Reconstruction — Where and How

**Status**: DECIDED
**Date**: 2026-04-20
**Deciders**: Orchestration team
**Related Open Point**: OP-25

---

## Context

AWS Transcribe does not return a pre-formatted diarized transcript string ready for LLM consumption. Its output is split across two structures:

- **`items[]`** — word-level tokens, each with `start_time`, `end_time`, `content`, and `type` (pronunciation or punctuation)
- **`speaker_labels.segments[]`** — per-speaker time ranges with `start_time`, `end_time`, and `speaker_label` (e.g., `spk_0`, `spk_1`)

Before calling `POST /llm/meeting-summary`, the system must reconstruct a coherent diarized text transcript by merging these two structures.

A separate infrastructure decision has also been made: the async AWS Transcribe job lifecycle (audio upload to S3, job submission, completion notification, result retrieval) will be handled by a dedicated **AWS Lambda function**, not by the Orchestrator service directly. The Orchestrator is hosted on PCF Tanzu / Spring Boot and does not poll AWS.

---

## Decision Drivers

1. The AI team's LLM prompt templates expect a clean diarized transcript as a single string input.
2. The reconstruction logic is deterministic and non-LLM — it belongs in the Orchestration layer, not the AI layer.
3. Output format must be agreed with the AI team so prompt templates can reliably parse speaker turns.
4. The Lambda already has the raw AWS Transcribe output in hand — it is the natural place to perform reconstruction before notifying the Orchestrator.
5. The Orchestrator (PCF Tanzu) should not need to handle AWS Transcribe JSON internals — keeping it AWS-agnostic simplifies the Spring Boot service.

---

## Options Considered

### Option A — Reconstruction inline in Orchestrator request handler
Lambda sends raw AWS Transcribe JSON to the Orchestrator. Orchestrator merges `items[]` and `speaker_labels.segments[]` inline before calling the LLM.

**Pros**: All logic in one place (Orchestrator codebase).
**Cons**: Orchestrator must parse AWS Transcribe-specific JSON schema; couples Spring Boot service to AWS Transcribe internals; harder to test in isolation; Lambda callback payload would be large (full raw Transcribe JSON).

---

### Option B — Dedicated `TranscriptReconstructionService` within Orchestrator
Same as Option A but extraction into a Spring component with a clear interface.

**Pros**: Better internal structure than Option A; testable in isolation.
**Cons**: Same fundamental problem — Orchestrator still receives and understands raw AWS Transcribe JSON. Reconstruction logic is physically in PCF Tanzu but conceptually belongs with the AWS layer.

---

### Option C — Reconstruction handled by AI team
AI team receives raw AWS Transcribe JSON and reconstructs before LLM prompt construction.

**Pros**: Orchestrator and Lambda stay thin.
**Cons**: Leaks AWS infrastructure concerns into the LLM layer; AI team should not need to know about AWS Transcribe output format; violates the clean `meetingContext.transcript` contract already agreed.

---

### Option D — Reconstruction inside the Lambda (CHOSEN)
Lambda handles the full AWS lifecycle AND transcript reconstruction:
1. Receives audio upload trigger
2. Submits AWS Transcribe job
3. Waits for completion via EventBridge / SNS notification
4. Retrieves raw Transcribe output
5. Merges `items[]` with `speaker_labels.segments[]` into a diarized text string
6. Sends the clean reconstructed transcript to the Orchestrator via webhook callback

Orchestrator receives a ready-to-use `transcript` string and places it directly into `meetingContext.transcript` in the LLM request.

**Pros**: Lambda already has the raw output — no redundant data transfer; Orchestrator stays AWS-agnostic; reconstruction logic lives closest to where the data originates; Lambda is independently testable and deployable; callback payload is small (plain text string + meetingId).
**Cons**: Reconstruction logic is in a Lambda (Python/Node) rather than the main Java codebase — separate test suite needed.

---

## Decision

**Option D** — transcript reconstruction inside the AWS Lambda function.

The Lambda owns the complete AWS Transcribe lifecycle end-to-end and delivers a clean diarized string to the Orchestrator. The Orchestrator treats `transcript` as an opaque string input — it has no dependency on AWS Transcribe's JSON schema.

---

## Consequences

### Lambda responsibilities
- Upload audio to S3
- Submit and monitor AWS Transcribe job
- Reconstruct diarized transcript from `items[]` + `speaker_labels.segments[]`
- POST clean transcript to Orchestrator callback endpoint with `meetingId` and `transcript` string

### Orchestrator responsibilities
- Expose a callback endpoint to receive the reconstructed transcript from Lambda
- Store transcript against `meetingId` pending the agent completing speaker tagging
- On FULL_SUMMARY trigger (post-tagging), retrieve stored transcript and include in LLM request

### Agreed transcript output format (to be confirmed with AI team)

Proposed diarized transcript format for `meetingContext.transcript`:

```
[spk_0] Hey Marcus, long time no see.
[spk_1] Ya still around here, but office now hybrid already.
[spk_0] Shiok lah, hybrid life.
```

- One line per speaker turn
- Speaker label in square brackets at the start of each line
- Consecutive words from the same speaker merged into a single turn
- Punctuation preserved from AWS Transcribe `items[]`

### Error handling

| Scenario | Handling |
|---|---|
| No speaker labels returned (single speaker) | Lambda wraps entire transcript as `[spk_0]`; sets a `singleSpeaker: true` flag in callback; Orchestrator sets `meetingType: VOICE_RECAP`, `agentSpeakerId: null` |
| Speaker label missing for a word token | Assign to previous speaker segment if within time overlap tolerance; otherwise omit token |
| Segment overlap (two speakers, same timestamp) | Prioritise segment with higher confidence score; log warning in Lambda |
| AWS Transcribe job failed or empty output | Lambda POSTs error status to Orchestrator callback; Orchestrator returns error to Pega UI; LLM is not called |
| Orchestrator callback unreachable | Lambda retries with exponential backoff (3 attempts); logs failure to CloudWatch |

---

## References

- OP-25 in `aidlc-docs/inception/requirements/requirements.md`
- `POST /llm/meeting-summary` contract: `doc/api-contracts/API-llm-meeting-summary-endpoint.md`
- AWS Transcribe diarization: `TranscriptEvent.Transcript.Results[].Alternatives[].Items[]` + `speaker_labels`
