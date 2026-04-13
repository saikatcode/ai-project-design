# Requirements: Voice-to-Text Meeting Assistant

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | N/A (Source requirement) |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Overview

Insurance agents conduct face-to-face and phone meetings with clients. They need a mobile app that records these conversations, transcribes them accurately (including mixed-language scenarios), and generates structured summaries that are pushed to the Advisor Platform for follow-up actions.

---

## Requirements

### REQ-VOICE-001: Meeting Recording
**Priority:** Must

Agents must be able to record client meetings from their mobile device with a single tap.

**Acceptance Criteria:**
- One-tap start/stop recording in the Pega Cosmos mobile app
- Audio stored locally on device until upload completes
- Recording continues even if network connectivity drops (offline support)
- Clear visual indicator that recording is active

---

### REQ-VOICE-002: Transcription with Speaker Diarization
**Priority:** Must

The recorded audio must be transcribed into text with clear identification of who said what (agent vs. client).

**Acceptance Criteria:**
- Transcription accuracy ≥ 90% for English
- Speaker diarization correctly identifies at least 2 speakers
- Timestamps included for each segment
- Processing completes within 2x the audio duration

**Technical Notes:** _Audio flows: Pega App → Orchestrator → STT Service (AWS Transcribe or alternative). NOT directly from Pega to AWS._

---

### REQ-VOICE-003: Mixed Language Support
**Priority:** Should

Many client meetings in Malaysia/Singapore involve code-switching between English, Mandarin, and Malay. The system should handle these scenarios.

**Acceptance Criteria:**
- System transcribes mixed English-Mandarin conversations with ≥ 80% accuracy
- Malay segments are transcribed or translated to English
- Language segments are labelled in the transcript

---

### REQ-VOICE-004: AI-Powered Meeting Summary
**Priority:** Must

After transcription, the system must generate a structured summary of the meeting with key discussion points, action items, and follow-ups.

**Acceptance Criteria:**
- Summary includes: key topics discussed, client concerns/objections, product interest signals, agreed next steps, and action items
- Summary is generated within 60 seconds of transcription completing
- Agent can review and edit the summary before it is pushed to the Advisor Platform

---

### REQ-VOICE-005: Advisor Platform Integration
**Priority:** Must

Approved meeting summaries must be pushed to the Advisor Platform and linked to the relevant customer record.

**Acceptance Criteria:**
- Summary is pushed to Advisor Platform via API
- Linked to the correct customer profile
- Action items are created as tasks in the agent's workflow
- Audit trail maintained for compliance

---

### REQ-VOICE-006: Offline Support
**Priority:** Must

Agents often meet clients in areas with poor connectivity. The app must work offline and sync when connectivity is restored.

**Acceptance Criteria:**
- Recording works fully offline
- Completed recordings queued for upload when connectivity returns
- Pega case persistence ensures no data loss
- Agent can view pending uploads and their status

---

### REQ-VOICE-007: Scale and Cost
**Priority:** Must

The system must support approximately 10 million minutes of audio per month across all agents at an acceptable cost per minute.

**Acceptance Criteria:**
- STT solution cost is within approved budget (target: < $0.01 per minute)
- Architecture supports horizontal scaling
- Singapore data residency requirements are met

---

## Out of Scope (v1)
- Real-time live transcription during meetings (batch processing only for v1)
- Video recording
- Sentiment analysis on audio tone (text-based sentiment only)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial draft | Tech Architect |
