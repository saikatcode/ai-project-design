# ADR-002: Transcript Reconstruction — Orchestrator Component Design

**Status**: DRAFT — Pending decision
**Date**: 2026-04-14
**Deciders**: Orchestration team + AI team
**Related Open Point**: OP-25

---

## Context

AWS Transcribe does not return a pre-formatted diarized transcript string ready for LLM consumption. Its output is split across two structures:

- **`items[]`** — word-level tokens, each with `start_time`, `end_time`, `content`, and `type` (pronunciation or punctuation)
- **`speaker_labels.segments[]`** — per-speaker time ranges with `start_time`, `end_time`, and `speaker_label` (e.g., `spk_0`, `spk_1`)

Before calling `POST /llm/meeting-summary`, the Orchestrator must reconstruct a coherent, diarized text transcript by merging these two structures.

---

## Decision Drivers

1. The AI team's LLM prompt templates expect a clean diarized transcript as a single string input.
2. The reconstruction logic is deterministic and non-LLM — it belongs in the Orchestration layer, not the AI layer.
3. Output format must be agreed with the AI team so prompt templates can reliably parse speaker turns.
4. The component must handle degraded AWS Transcribe output gracefully (poor diarization, overlapping segments, missing speaker labels).

---

## Options Considered

### Option A — Inline reconstruction in Orchestrator request handler
Embed the merge logic directly in the meeting summary API handler as a pre-processing step.

**Pros**: Simple, no additional component.
**Cons**: Couples transcript parsing to the API layer; hard to test in isolation; reuse is difficult if other intents need the same transcript.

---

### Option B — Dedicated `TranscriptReconstructionService` within Orchestrator
Create a single-responsibility internal service/module that accepts raw AWS Transcribe output and returns a diarized transcript string.

**Pros**: Testable in isolation; reusable; output format can be versioned independently; clear boundary for AI team collaboration.
**Cons**: Slightly more code structure upfront.

---

### Option C — Preprocessing step handled by AI team
AI team receives raw AWS Transcribe JSON and performs reconstruction on their side before LLM prompt construction.

**Pros**: Orchestrator stays thin.
**Cons**: Increases AI team dependency; leaks infrastructure concerns into the LLM layer; Orchestrator loses visibility over transcript quality.

---

## Decision

**[PENDING]** — To be decided jointly with AI team.

Preliminary recommendation: **Option B** — a dedicated `TranscriptReconstructionService` within the Orchestrator. This keeps the AI team's contract clean (they receive a formatted string), makes the reconstruction testable, and gives the Orchestration team full control over output format and error handling.

---

## Consequences

### If Option B is adopted:
- A `TranscriptReconstructionService` must be designed and built as part of the Orchestrator codebase
- Output format (e.g., `[spk_0] utterance \n [spk_1] utterance`) must be agreed with the AI team and locked into `POST /llm/meeting-summary` contract
- Error handling cases must be defined: missing speaker labels, single-speaker output, segment overlap

### Agreed output format (to be confirmed with AI team)

Proposed diarized transcript format for the `meetingContext.transcript` field:

```
[spk_0] Hey Marcus, long time no see.
[spk_1] Ya still around here, but office now hybrid already.
[spk_0] Shiok lah, hybrid life.
```

- One line per speaker turn
- Speaker label in square brackets at the start of each line
- Consecutive words from the same speaker merged into a single line
- Punctuation preserved from AWS Transcribe `items[]`

### Error handling cases (to be defined):
| Scenario | Proposed handling |
|---|---|
| No speaker labels returned (single speaker) | Wrap entire transcript as `[spk_0]`; set `agentSpeakerId: null`; set `meetingType: VOICE_RECAP` |
| Speaker label missing for a word token | Assign to previous speaker segment if within time overlap tolerance; otherwise omit |
| Segment overlap (two speakers same timestamp) | Prioritise segment with higher confidence score; flag in Orchestrator logs |
| AWS Transcribe job failed or empty output | Return error to Pega UI; do not call LLM |

---

## References

- OP-25 in `aidlc-docs/inception/requirements/requirements.md`
- `POST /llm/meeting-summary` contract: `doc/api-contracts/API-llm-meeting-summary-endpoint.md`
- AWS Transcribe diarization documentation: `TranscriptEvent.Transcript.Results[].Alternatives[].Items[]` + `speaker_labels`
