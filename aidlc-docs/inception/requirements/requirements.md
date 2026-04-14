# Requirements — Advisor 2.0 AI Features
**Company**: Great Eastern Insurance (GE)
**Project**: Advisor 2.0 — AI-powered features for Insurance Agents/Distributors
**Version**: 0.2 (Working Draft)
**Date**: 2026-04-14
**Status**: Draft — context not 100% final; subject to revision

## Source Legend
| Tag | Source |
|-----|--------|
| [BIZ] | Business requirements document (Advisor 2.0 JIRA stories) |
| [DESIGN] | Technical design document (ai_project_design_consolidated.md) |
| [VERIFIED] | Confirmed via requirements verification questions |
| [SUGGESTED] | Suggested addition/edit — requires user review and approval |

---

## 1. Project Overview

Great Eastern Insurance is building AI-powered features into **Advisor 2.0**, the iPad/mobile application used by Insurance Agents and Distributors across Singapore and Malaysia. The initiative is a greenfield build on top of the existing GP (advisor platform) infrastructure.

The **Alpha Release** will be delivered as a controlled pilot with a limited agent group in one market, before broader rollout. [VERIFIED Q2]

---

## 2. Use Cases in Scope (Alpha Release)

| ID | Use Case | Description |
|----|----------|-------------|
| UC-01 | Meeting Transcription & Summary | Record, transcribe, and AI-generate structured meeting summaries from agent-customer interactions |
| UC-02 | Meeting Preparation Notes | AI-generate pre-meeting notes tailored to customer profile and history |
| UC-03 | Agent Knowledge Chatbot (ROSE) | Intent-driven conversational assistant for product knowledge, policy servicing, sales support, and compliance guidance |
| UC-04 | FLP Attribute Extraction | Extract up to 90 predefined Financial Life Plan attributes from AI meeting summary; agent reviews and saves to FLP system for customer FLP creation |
| UC-05 | Soft Attribute Extraction & Engagement Ideas | Extract up to 40 predefined soft attributes (hobbies, interests, etc.) from meeting transcript per customer; tag to customer profile; use downstream for engagement ideas and corporate-sponsored event matching |

All five use cases are in scope for the Alpha Release. [VERIFIED Q1 + user addition 2026-04-14]

---

## 3. Functional Requirements — UC-01: Meeting Transcription & Summary

### 3.1 Transcription Entry Points [BIZ AD-223]
The agent must be able to begin a transcription from the following entry points:
- Landing page → Shortcuts → Transcribe
- Landing page → Today's appointments → Transcribe (per appointment card)
- Landing page → Today's appointments → Prepare for meeting → Transcribe
- Landing page → Today's appointments → View all → Calendar view → Transcribe (per card)
- Landing page → Today's appointments → View all → Calendar view → Prepare for meeting → Transcribe (per card)
- Landing page → Navigation bar (customer profile) → Search customer → Customer profile → Transcribe

### 3.2 Consent and Privacy [BIZ AD-224, AD-225]
- A consent screen must be displayed before every transcription session
- Agent must acknowledge via checkbox; Start Transcription button is disabled until checkbox is selected
- A privacy content link must be available; content opens in a browser
- Agent can exit consent screen without starting transcription

### 3.3 Transcription Modes [BIZ AD-283]
Three distinct modes must be supported, each using a different backend prompt:

| Mode | Trigger | Customer Context | Prompt |
|------|---------|-----------------|--------|
| Standard (with customer) | Appointment card entry points | Auto-preserved from appointment | Meeting summary prompt |
| Standard (no customer) | Shortcuts / Customer profile | None at start; agent tags speakers pre-summary via snippet screen | Meeting summary prompt |
| Voice recap | Shortcuts / Customer profile + recap checkbox | Optional | Voice recap prompt (agent-only) |

### 3.4 Recording Controls [BIZ AD-226]
During an active recording session:
- Recording time elapsed must be displayed
- Agent can: Pause, Resume (after pause), Stop (with confirmation), Trash (with confirmation)
- On Trash confirmation: recording and notes are discarded, no transcription triggered
- On Stop confirmation: transcription, translation, and snippet generation are triggered sequentially in the backend (summarisation is triggered later, only after the agent completes customer tagging — see Section 3.6)
- For recordings exceeding 60 minutes: agent is prompted every 60 minutes to confirm the meeting is still ongoing
- If agent does not respond within 15 minutes of the prompt, the recording is automatically stopped
- Maximum audio file size: ≤ 30MB [BIZ @tech]

### 3.5 Notes, Files, and Photos During Recording [BIZ AD-232, AD-233, AD-234]
- Agent can take text notes during recording (non-recap mode only)
- Agent can upload JPEG, PNG, and PDF files
- Agent can capture photos
- Text/written notes taken during recording ARE sent to `POST /llm/query` as part of the summary generation input
- Uploaded files (JPEG, PNG, PDF) and captured photos are stored alongside the meeting summary (Customer 360 — Notes & Interactions tab, iPad only) but are NOT sent to the LLM
- If upload fails, an error message must prompt the agent to retry

### 3.6 Async Processing Flow [BIZ AD-243, AD-241] [DESIGN 2.10] [CORRECTED]
The end-to-end flow from recording completion to agent review is as follows:

```
Step 1: Agent stops recording in Pega
         |
         v
Step 2: ASR / Transcription Service processes audio (async)
         |
         v
Step 3: Translation (if needed — multilingual / code-switching)
         |
         v
Step 4: Orchestrator calls AI Team: POST /llm/query (intent: snippet)
        Generates distinct speech snippets per identified speaker
        (spk_0, spk_1, spk_2, ... — includes agent and customers)
         |
         v
Step 5: Push notification sent to agent
         |
         v
Step 6: Agent opens snippet/tagging screen in Pega
        - Sees speech snippets for each non-agent speaker
        - Tags each speaker (spk_0, spk_1...) to the correct customer
        - Agent's own speaker is excluded from tagging
        - At least one customer must be tagged to proceed
         |
         v
Step 7: Agent confirms tagging
        Orchestrator calls AI Team: POST /llm/query (intent: summary)
        with transcript + speaker-to-customer mapping
         |
         v
Step 8: Agent sees Meeting Details screen — all outputs together:
         - Meeting Summary (Overview, Discussion, Action Items)
         - FLP Attributes (up to 90, per customer)
         - Soft Attributes (up to 40, per customer)
         - Action Items / To-Do List
         |
         v
Step 9: Agent edits and saves all outputs to database
         |
         v (optional)
Step 10: Agent triggers shareable summary creation
         Orchestrator calls AI Team: POST /llm/query (intent: shareable_summary)
         (based on agent's edited and saved summary/attributes/to-do)
         |
         v
Step 11: Agent shares shareable summary via WhatsApp / Telegram
```

**Key implications:**
- Customer tagging occurs **before** summary generation — it is an interactive mid-flow step
- The push notification after Step 4 signals the agent to tag, not to view the summary
- FLP attributes, soft attributes, and action items are all returned as part of (or alongside) the summary response in Step 7–8
- Shareable summary is generated from the **agent-edited** version, not the raw AI output
- Turnaround time SLA (Steps 2–5): to be agreed with AI team [VERIFIED Q3]
- While Steps 2–4 are processing, the Meeting Summary tab shows a "summary will be available soon" message [BIZ AD-243]

### 3.7 Customer Tagging [BIZ AD-242] [CORRECTED — occurs BEFORE summary generation]
Tagging is an interactive step triggered by the push notification after `POST /llm/query` (intent: snippet) completes (Step 5–6 in Section 3.6). Summary generation does not begin until the agent completes tagging.

**Snippet screen behaviour:**
- The agent is shown distinct speech snippets for each identified speaker (spk_0, spk_1, spk_2, ...)
- Snippets are generated via `POST /llm/query` (intent: snippet) — content is from that speaker's distinct speech segments
- The agent's own speaker track is excluded from the tagging screen
- Snippets must not include anything said by the agent/advisor [BIZ AD-242]
- Snippet length: ~TBD words (open point — see OP-11)

**Tagging rules:**
- For appointment-based entry points: the customer linked to the appointment is pre-tagged to spk_0 (or the primary identified non-agent speaker)
- For no-context entry points: agent must search and tag each speaker manually
- Each speaker can be mapped to an existing customer via typeahead search (3 character minimum) or a new customer can be created (name + phone mandatory)
- At least one customer must be tagged before tagging can be confirmed and summary generation triggered
- Agent can edit/change customer mapping before confirming

**After confirmation:**
- Speaker-to-customer mapping is submitted to the Orchestrator
- Orchestrator calls `POST /llm/query` (intent: summary) with transcript + speaker mapping
- [SUGGESTED]: If the agent closes the app before completing tagging, the snippet data and partial tags must be preserved for re-entry (see OP-23)

### 3.8 Meeting Summary Content [BIZ AD-244] [CORRECTED]
Once `POST /llm/query` (intent: summary) completes, the agent sees **all the following outputs together on a single Meeting Details screen**:

| Output | Format | Notes |
|--------|--------|-------|
| Overview | Text | Purpose, type of review, attendees, conversation stage, tone, sentiments, overall insights |
| Discussion | Text / bullets | Key discussion points |
| Action Items | Bullets, split by customer and advisor | Only explicitly mentioned items — no inference |
| FLP Attributes | Categorised list per customer | Up to 90 attributes — see UC-04 (Section 6) |
| Soft Attributes | Categorised list per customer | Up to 40 attributes — see UC-05 (Section 7) |

Tone: Warm, friendly, professional, not sales-pushy
Guardrails: No product push, follow GE policy
Multiple customers: Summary is unified; FLP attributes, soft attributes, and action items are split per customer (e.g., "Customer 1 — xx")

[CONFIRMED]: `POST /llm/query` (intent: summary) returns all outputs — meeting summary, FLP attributes, soft attributes, and action items — in a single response payload. No separate parallel calls needed. (Closes OP-21)

### 3.9 Summary Edit, Recap, and Share [BIZ AD-245, AD-246, AD-247] [CORRECTED]

**Edit:**
- Agent can edit the meeting summary, FLP attributes, soft attributes, and action items
- Editing is locked after agent confirms save [BIZ AD-245]
- Agent is prompted to save if they navigate away without saving (once per day)

**Shareable Summary:**
- After editing and saving, agent may optionally request a customer-shareable summary
- Triggered by agent CTA → Orchestrator calls AI Team: **POST /llm/query** (intent: shareable_summary)
- Input to the API: the agent's edited and saved summary, attributes, and to-do — **not the raw AI output**
- Generated shareable summary format: WhatsApp message style
  - Casual greeting
  - Short recap of discussed points
  - Action items
  - Tone: Warm, friendly, professional
  - Guardrail: Exclude agent-only information (soft attributes, advisor-only reference numbers)

**Share:**
- Agent can share the generated shareable summary via WhatsApp or Telegram (native app share)
- The message opens as a draft in the selected app for the agent to review before sending
- Agent can also copy text to clipboard [BIZ AD-247]

### 3.10 Client Attributes Extraction [BIZ AD-249, AD-250, AD-252]
Post-summary, two types of attributes are extracted and presented for agent review. These are covered as dedicated use cases:

| Attribute Type | Dedicated Use Case | Detail |
|----------------|-------------------|--------|
| FLP (Financial Planning) — 90 attributes, 14/15 categories | **UC-04** | See Section 6 |
| Soft / Relationship Intelligence — 40 attributes, 6 categories | **UC-05** | See Section 7 |

- Both are returned as part of the `POST /llm/query` (intent: summary) response and displayed together with the meeting summary
- Agent reviews, edits, and confirms all outputs before saving
- FLP attributes → FLP system; Soft attributes → GP Customer Profile

### 3.11 Action Items [BIZ AD-255, AD-256, AD-257]
- Action items extracted from meeting summary are displayed in the Action Items tab
- Where deadline is detected in transcript, it is used; otherwise default deadline applies:
  - Weekday (Mon–Thu, Sun): meeting date + 1 day
  - Friday: meeting date + 3 days
  - Saturday: meeting date + 2 days
- Agent can add, edit, and delete action items
- Action items sync to the global Tasks list (Landing page)

---

## 4. Functional Requirements — UC-02: Meeting Preparation Notes

### 4.1 Entry Points [BIZ AD-237]
Meeting prep is triggered from:
- Landing page → Today's appointments → Prepare for meeting (via card)
- Landing page → Today's appointments → View all → Calendar → Prepare for meeting (via card)
- Meeting Details → Meeting Prep tab → CTA "Prepare for meeting" (empty state)
- Entry points are no longer available once a meeting summary has been generated for that meeting

### 4.2 Customer Scenarios [BIZ AD-238, AD-239, AD-240]
The Orchestrator must generate different prep content based on the customer scenario:

**Scenario A — New Customer** [BIZ AD-238]
Sections: Quick check-in, Brief intro, Meeting framing, Personal & life context discovery, Financial & insurance baseline, Goals & priorities discovery, Solution alignment, Alignment & reflection, Close & next steps
Format: Bullet points | Tone: Warm, friendly, professional
Guardrails: No product push, follow GE policy. No Personal Insights or Customer Context sections.

**Scenario B — Existing Customer with Previous Meetings** [BIZ AD-239]
Sections: Conversational openers (personal signals from last meeting, local news/trends), Conversation starter (life event check-in, recap of previous discussion, products mentioned, customer needs, agreed items, open items), Discussion on key focus area, Close & next steps
Additional data: Date of last meeting from AI note-taking app
Format: Bullet points + task list (see Section 4.5) | Tone: Warm, friendly, professional
Guardrails: No product push, follow GE policy

**Scenario C — Existing Customer, No Previous Meetings Captured** [BIZ AD-240]
Sections: Conversational openers, Conversation starter (life event check-in, any new responsibilities, expectation re-alignment, known servicing/underwriting info), Discussion on key focus area, Close & next steps
Data: Use relevant data from policy system and customer database
Format: Bullet points + task list (see Section 4.5) | Tone: Warm, friendly, professional
Guardrails: No product push, follow GE policy

### 4.3 Manual Notes [BIZ AD-281, AD-282]
- Agent can add manual notes to supplement AI-generated prep (via separate button from AI generation)
- Text area: multiline, max 255 characters, non-rich text
- Manual notes are displayed below AI-generated prep
- Editable until AI summary is generated for that meeting; editing is locked thereafter

### 4.4 Promos [BIZ AD-285] [VERIFIED Q14]
- Relevant promos are displayed alongside AI-generated meeting prep
- Orchestrator fetches and includes promo data as part of the meeting prep generation call
- [SUGGESTED]: Promo retrieval should be treated as a non-blocking enrichment — prep notes should be returned even if promo data is unavailable

### 4.5 Task List Output [CORRECTED]
The task list is an output of **Meeting Summary** (`POST /llm/meeting-summary`, subIntent: `FULL_SUMMARY`) — not of Meeting Prep. Tasks are derived from action items agreed during the meeting and are propagated to the agent's dashboard/Tasks system after the summary is generated.

Meeting Prep (`POST /llm/meeting-prep`) does **not** produce tasks. It produces a structured conversation guide only.

**Task list format** (JSON structure within the `POST /llm/meeting-summary` FULL_SUMMARY response payload):
```json
{
  "tasks": [
    {
      "taskHeader": "string — short description of the advisor action item",
      "dueDate": "ISO 8601 date string — e.g. 2026-05-01 | null"
    }
  ]
}
```

**Rules:**
- `tasks[]` contains advisor-only action items (not customer action items — those are shown in the UI via the ACTION_LIST block in `contentBlocks`)
- Due date logic: weekday+1, Friday+3, Saturday+2 unless a specific date is determinable from the transcript (see Section 3.11)
- `tasks[]` is produced in the same `FULL_SUMMARY` response alongside the meeting summary, FLP attributes, soft attributes, and action items
- Tasks are consumed by a downstream Tasks/dashboard service to surface advisor to-dos
- OP-24 closed — the earlier question about Scenario A (new customer meeting prep) task list is no longer applicable

---

## 5. Functional Requirements — UC-03: Agent Knowledge Chatbot (ROSE)

### 5.1 Intent-Based Routing [DESIGN 2.3]
- Fixed-intent journeys for meeting prep and meeting summary (not chatbot routing)
- Registry-driven intent/sub-intent resolution for chatbot
- Deterministic classification where practical; LLM-assisted classification for ambiguous requests
- Default fallback/denial for out-of-scope or unsupported requests

### 5.2 Supported Intents (Initial Registry) [DESIGN 7.2]

| Category | Intent | Sub-intent | Description |
|----------|--------|-----------|-------------|
| Product Info | PRODUCT_INFO | FEATURE_EXPLANATION | Explain product/rider/feature |
| Product Info | PRODUCT_INFO | PRODUCT_COMPARISON | Compare products/riders/plans |
| Policy Service | POLICY_SERVICE | POLICY_STATUS | Policy status or servicing guidance |
| Policy Service | POLICY_SERVICE | SERVICING_STEPS | How-to steps for service requests |
| Claims | CLAIMS | CLAIM_PROCESS | Claims eligibility, process, documents |
| Customer Meeting | CUSTOMER_MEETING | MEETING_PREP_LIGHT | Ad hoc quick prep in chat |
| Customer Meeting | CUSTOMER_MEETING | NEXT_BEST_ACTION | Next talking points or actions |
| Knowledge Search | KNOWLEDGE_SEARCH | FAQ_LOOKUP | FAQ from knowledge docs |
| Knowledge Search | KNOWLEDGE_SEARCH | DOCUMENT_SUMMARY | Summarise retrieved document |
| Sales Support | SALES_SUPPORT | OBJECTION_HANDLING | Responses to customer objections |
| Sales Support | SALES_SUPPORT | RECOMMENDATION_SUPPORT | Candidate solutions based on needs |
| Compliance | COMPLIANCE | ALLOWED_TO_SAY | Compliant phrasing or restrictions |
| Invalid | INVALID | OUT_OF_SCOPE | Unsupported/unsafe/irrelevant |

### 5.3 Multi-Turn Conversation [DESIGN 2.6]
- Chatbot supports multi-turn conversations
- Conversation continuity managed via `conversationId`
- Recent turns passed as context to each LLM call; older turns summarised

### 5.4 Knowledge Retrieval [DESIGN 2.7, 2.8]
- Semantic retrieval over enterprise knowledge documents via the AI team's Retrieve API
- Retrieval and LLM are separate service concerns — Orchestrator calls each independently
- Citations/source links must be included in chatbot responses where documents are retrieved
- Knowledge documents sourced from SharePoint document library; ingested and embedded by AI team

### 5.5 Fallback and Denial [DESIGN 7.3]
- Out-of-scope, missing-data, compliance-sensitive, or unsafe requests return a polite denial
- Denial must: acknowledge limitation, explain what is supported, invite rephrasing where appropriate

---

## 6. Functional Requirements — UC-04: FLP Attribute Extraction

### 6.1 Overview [BIZ AD-249, AD-250]
The AI system must automatically extract Financial Life Plan (FLP) attributes from the AI-generated meeting summary and present them to the agent for review and confirmation before saving to the FLP system.

### 6.2 Attribute Taxonomy
- Total predefined FLP attributes: **90** [VERIFIED]
- Organised into predefined categories: up to **14 for SG**, **15 for MY** [BIZ AD-250]
- Attribute list is predefined and fixed — [SUGGESTED: confirm whether business can extend the taxonomy over time; see OP-15]

### 6.3 Extraction Source and Scope
- Attributes are extracted from the **AI-generated meeting summary** (not directly from the raw transcript)
- For multi-customer meetings: attributes are extracted and presented **per customer**
- Only attributes with identifiable evidence in the summary should be extracted — no inference beyond what is stated

### 6.4 Agent Review Workflow [BIZ AD-249, AD-250]
- Agent reviews extracted attributes organised by category per customer
- Agent may:
  - Accept/confirm an extracted attribute
  - Deselect/remove an attribute before saving
  - Manually add attributes not extracted by AI
  - Add or remove entire categories
  - Search for specific categories and attributes
- Only categories with extracted or manually added attributes are displayed
- Agent can check/uncheck individual attributes before final save
- Confirmed attributes are saved to the **FLP system**

### 6.5 Storage and Downstream Use
- Confirmed FLP attributes populate the corresponding fields when creating or updating a customer's FLP
- [SUGGESTED]: Define conflict resolution when AI-extracted attributes contradict existing FLP data (see OP-16)

---

## 7. Functional Requirements — UC-05: Soft Attribute Extraction & Engagement Ideas

### 7.1 Overview [BIZ AD-252, AD-213]
The AI system must extract relationship intelligence (soft attributes) from the meeting transcript for each customer, tag them to the customer profile, and use them downstream for engagement idea generation and corporate-sponsored event matching.

### 7.2 Attribute Taxonomy
- Total predefined soft attributes: **40** [VERIFIED]
- Organised into up to **6 predefined categories** (e.g., hobbies, interests, lifestyle, family context, life stage, personal milestones) [BIZ AD-252]
- Attribute list is predefined — [SUGGESTED: confirm whether business can extend the taxonomy; see OP-17]

### 7.3 Extraction Source and Scope
- Attributes are extracted from the **meeting transcript** (not limited to the summary)
- For multi-customer meetings: attributes are extracted **per customer** and attributed to the correct identified speaker
- Only attributes that are explicitly mentioned or clearly inferable from the transcript — no fabrication

### 7.4 Agent Review Workflow [BIZ AD-252]
- Agent reviews extracted soft attributes per customer, organised by category
- Agent may accept, edit, add, or remove attributes and categories
- Agent can check/uncheck attributes before final save
- Confirmed soft attributes are saved to **GP Customer Profile**

### 7.5 Downstream: Engagement Ideas [BIZ AD-213] [VERIFIED]
- Confirmed soft attributes are used to surface engagement ideas for the agent
- Engagement ideas are displayed on the Landing page → Customers to Engage carousel
- [SUGGESTED]: Clarify whether engagement idea generation is AI-powered (LLM-ranked) or rule/tag-based matching (see OP-18)

### 7.6 Downstream: Corporate-Sponsored Event Matching [VERIFIED]
- Confirmed soft attributes are used to match relevant customers with upcoming corporate-sponsored events
- Matched customers are surfaced to the agent for proactive engagement outreach
- [SUGGESTED]: Clarify whether event-to-customer matching is AI-ranked or rule/tag-based (see OP-19)
- [SUGGESTED]: Confirm whether the Engagement Ideas and Event Matching features are in scope for the Alpha Release or a subsequent release (see OP-20)

---

## 8. Integration Requirements

### 8.1 Call Flow [VERIFIED Q7, Q8]
```
Pega Mobile App
      |
      v (all calls via Axway T2)
Enterprise API Gateway (Axway T2)
      |
      v
Orchestrator (Python / FastAPI, AWS)
      |
      +---> GP / CRM / Policy Systems (direct API calls) [VERIFIED Q13]
      +---> AI Team: POST /llm/query       (all LLM operations — snippet, summary, shareable summary, meeting prep, chatbot; differentiated by intent identifier in request)
      +---> AI Team: POST /retrieve        (semantic retrieval for chatbot RAG)
      +---> AI Team: ASR / Transcription Service
      +---> Promo / Campaign System [VERIFIED Q14]
```

### 8.2 Business-Facing API Endpoints [DESIGN 2.1]
| Endpoint | Use Case |
|----------|---------|
| POST /meeting-prep/generate | UC-02 — Meeting Prep |
| POST /meeting-summary/generate | UC-01 — Meeting Summary (submit audio / trigger flow) |
| POST /chat/query | UC-03 — Chatbot |

### 8.3 Internal AI Team Endpoints [DESIGN 2.1] [CONFIRMED] [UPDATED]
LLM operations are split across two endpoints by operation family, with operations within each endpoint distinguished by a `subIntent` identifier. (Closes OP-22, OP-26)

| Endpoint | SubIntent | When Called | Purpose |
|----------|-----------|-------------|---------|
| POST /llm/meeting-summary | SNIPPET | Step 4 of UC-01 flow | Generate per-speaker speech snippets for agent tagging |
| POST /llm/meeting-summary | FULL_SUMMARY | Step 7 of UC-01 flow | Generate meeting summary + FLP attributes + soft attributes + action items (single response) |
| POST /llm/meeting-summary | SHARABLE_SUMMARY | Step 10 of UC-01 flow (optional) | Generate customer-shareable WhatsApp-format summary from agent-edited output |
| POST /llm/meeting-prep | NEW_CUSTOMER | UC-02 | Generate meeting prep guide — new customer, no CRM record |
| POST /llm/meeting-prep | EXISTING_WITH_MEETINGS | UC-02 | Generate meeting prep guide — existing customer with prior meeting notes |
| POST /llm/meeting-prep | EXISTING_WITHOUT_MEETINGS | UC-02 | Generate meeting prep guide — existing customer, no meeting notes captured |
| POST /llm/query | chatbot | UC-03 | Chatbot intent response (single endpoint retained for UC-03) |
| POST /retrieve | — | UC-03 | Semantic chunk retrieval for chatbot RAG (separate endpoint, not LLM) |

### 8.4 Correlation Identifiers [DESIGN 2.6]
All API requests must include:
- `requestId` — unique identifier per API call
- `interactionId` — correlates all service calls within one user interaction
- `conversationId` — multi-turn conversation continuity (chatbot only)
- `sessionId` — optional, if UI/session lifecycle requires separate tracking

### 8.5 Enterprise System Integration [VERIFIED Q13]
The Orchestrator queries the following enterprise systems directly:
- GP (existing advisor system): customer profile, past meeting summaries, calendar data
- CRM / Policy systems: policy data, servicing history
- FLP: financial planning attribute definitions
- Promo/Campaign system: relevant promotions for meeting prep

[SUGGESTED]: Network connectivity between the Orchestrator (AWS private VPC) and on-premises GP/CRM/policy systems must be established via an approved secure channel (e.g., AWS Direct Connect or Site-to-Site VPN). This is an open architecture decision.

---

## 9. Non-Functional Requirements

### 7.1 Availability [VERIFIED Q4]
- Target: 24/7 high availability for all AI features
- AI services (Orchestrator, ASR, LLM, Retrieve) must be available outside business hours
- Degraded-mode behaviour (e.g., partial outage of one AI team API) must be defined [SUGGESTED]

### 7.2 Performance [VERIFIED Q3]
- Meeting prep response time: [TBD — to be agreed with AI team]
- Meeting summary turnaround (async, push notification): [TBD — to be agreed with AI team]
- Chatbot response time per turn: [TBD — to be agreed with AI team]
- [SUGGESTED]: Latency targets should be agreed as part of the AI team SLA definition, covering P50, P90, and P99 percentiles per endpoint

### 7.3 Language Support [VERIFIED Q5, Q6]
- UI and AI-generated content must support: English, Bahasa Melayu, Mandarin Chinese
- ASR / transcription must handle code-switching (mixed languages within a single recording)
- [SUGGESTED]: Define fallback behaviour when transcription language detection is inconclusive

### 7.4 Scalability [DESIGN 2.9]
- Model tier routing: FAST | STANDARD | DEEP_THINK based on intent complexity
- Token caps per request configurable per intent in the Intent Registry
- Retrieval caps: topK and chunk limits configurable per use case
- Caching: applicable to stable knowledge responses; must NOT be applied to user-specific or sensitive outputs

### 7.5 Observability [DESIGN 9.1]
- Structured logging with `interactionId` end-to-end across Pega, Orchestrator, Retrieve, and LLM
- Cost tracking per use case and model tier
- [SUGGESTED]: Monitoring dashboard covering: latency per endpoint, error rates, token consumption, and ASR turnaround time

---

## 10. Regulatory and Compliance Requirements [VERIFIED Q9]

### 8.1 Regulatory Scope
- MAS (Monetary Authority of Singapore) AI and data guidelines apply — SG market
- BNM (Bank Negara Malaysia) AI and data guidelines apply — MY market
- Specific guideline requirements to be confirmed with the compliance team

### 8.2 Data Privacy
- PDPA (Personal Data Protection Act) Singapore and Malaysia apply
- Audio recordings, transcripts, and customer data must be handled per PDPA requirements
- Customer consent is obtained at the point of transcription (consent screen — AD-224)

### 8.3 Data Residency [VERIFIED Q10]
- All data processing must occur within Singapore using AWS services hosted in a private VPC
- LLM inference uses a self-hosted model (AI team responsibility) — no data leaves the private environment
- Audio files, transcripts, and prompts must not be sent to external third-party LLM providers

### 8.4 Content Compliance [BIZ guardrails]
- AI-generated content must follow GE's content policy at all times
- No product recommendations that constitute regulated financial advice
- Action items must not be inferred — only explicitly stated items in the transcript
- Short recap must exclude agent-only information not intended for the customer

---

## 11. Security Requirements

Security Baseline extension is **ENABLED** (blocking constraints). The following rules apply:

| Rule | Requirement |
|------|------------|
| SECURITY-01 | Encryption at rest and in transit (TLS 1.2+) for all data stores |
| SECURITY-02 | Access logging on API gateway (Axway T2) and all network intermediaries |
| SECURITY-03 | Structured application-level logging with correlation IDs; no PII or secrets in logs |
| SECURITY-04 | HTTP security headers on all HTML-serving endpoints |
| SECURITY-05 | Input validation on all Orchestrator API parameters |
| SECURITY-06 | Least-privilege IAM roles for Orchestrator and all downstream service identities |
| SECURITY-07 | Restrictive network configuration — private VPC, no public ingress except via gateway |
| SECURITY-08 | Application-level access control — deny by default, IDOR prevention, JWT validation |
| SECURITY-09 | Security hardening — no default credentials, no stack traces in production responses |
| SECURITY-10 | Dependency pinning, vulnerability scanning, SBOM for production |
| SECURITY-11 | Secure design — rate limiting on public-facing endpoints, defence in depth |
| SECURITY-12 | Authentication and credential management — no hardcoded secrets, MFA for admin accounts |
| SECURITY-13 | Software and data integrity — SRI for external scripts, CI/CD access controls |
| SECURITY-14 | Security event alerting — auth failures, access violations, log retention ≥ 90 days |
| SECURITY-15 | Exception handling — fail closed, no unhandled exceptions, generic user-facing errors |

---

## 12. Intent Registry Governance [VERIFIED Q12]

The Intent Registry maps each intent/sub-intent to: tools, data sources, model profile, prompt templates, output schema, and fallback behaviour.

| Aspect | Owner |
|--------|-------|
| Prompt templates and model selection | AI team |
| Intent, sub-intent definition, and output structure | Business |
| Registry storage, deployment, and runtime execution | Orchestration team (user's team) |

The registry is stored and deployed as part of the Orchestrator service. Changes require input from the owning team per aspect above.

---

## 13. Open Points

| ID | Topic | Status |
|----|-------|--------|
| OP-01 | Transcription + summarisation turnaround SLA | To be agreed with AI team |
| OP-02 | Chatbot response time target per turn | To be agreed with AI team |
| OP-03 | Meeting prep generation latency target | To be agreed with AI team |
| OP-04 | Specific MAS and BNM AI guideline requirements | To be confirmed with compliance team |
| OP-05 | Audio file size limit (currently ≤ 30MB) | @tech — to be confirmed |
| OP-06 | Browser behaviour for privacy link on iOS/Android | @tech — to be confirmed |
| OP-07 | Sleep mode and phone disruption handling during recording | @tech — to be confirmed |
| OP-08 | Network connectivity from AWS VPC to on-premises GP/CRM systems | Architecture decision pending |
| OP-09 | Degraded-mode behaviour when one AI team API is unavailable | Design decision pending |
| OP-10 | Fallback behaviour when transcript language is inconclusive | Design decision pending |
| OP-11 | Speaker snippet length for customer tagging screen | TBD — business decision |
| OP-12 | Upload file size limit for notes attachments | @tech — to be confirmed |
| OP-13 | WhatsApp / Telegram share implementation on iOS/Android (native share) | @tech — to be confirmed |
| OP-14 | Customer creation flow scope (new customer during tagging) | Design detail pending |
| OP-15 | FLP attribute taxonomy governance — who can add/modify the 90 attributes over time? | Business decision pending |
| OP-16 | Conflict resolution when AI-extracted FLP attributes contradict existing FLP data | Design decision pending |
| OP-17 | Soft attribute taxonomy governance — who can add/modify the 40 attributes over time? | Business decision pending |
| OP-18 | Engagement idea generation mechanism — AI-powered (LLM-ranked) or rule/tag-based? | Architecture decision pending |
| OP-19 | Corporate-sponsored event matching mechanism — AI-ranked or rule/tag-based? | Architecture decision pending |
| OP-20 | Scope of Engagement Ideas and Event Matching for Alpha Release vs subsequent release | Scope decision pending |
| OP-21 | ~~Does `/llm/summary` return summary + FLP attributes + soft attributes + action items in a single response, or does the Orchestrator aggregate from separate calls?~~ | **CLOSED** — `POST /llm/query` (intent: summary) returns all outputs in a single response payload |
| OP-22 | ~~Are `/llm/summary` and `/llm/query` the same generic endpoint with different intents, or distinct endpoints?~~ | **CLOSED** — Single endpoint `POST /llm/query`; operations distinguished by intent identifier in request |
| OP-23 | If agent closes app before completing tagging, must snippet data and partial tags be preserved for re-entry? | Design decision pending |
| OP-24 | ~~Should Meeting Prep (Scenario A — new customer) also output a task list alongside prep notes?~~ | **Closed** — Tasks are output of Meeting Summary (FULL_SUMMARY), not of Meeting Prep. Meeting Prep never produces tasks. |
| OP-25 | **Transcript reconstruction**: AWS Transcribe does not produce a pre-formatted diarized transcript string. It returns word-level `items[]` tokens and a separate `speaker_labels.segments[]` with timestamp ranges. The Orchestrator must merge these into a single diarized text (e.g., `[spk_0] text \n [spk_1] text`) before calling `POST /llm/meeting-summary`. Design decisions required: (1) agreed output format with AI team, (2) error handling for poor diarization quality, (3) Orchestrator component ownership and structure. Candidate for ADR — see doc/risks-and-decisions/ADR-002-transcript-reconstruction.md. | Design decision pending |
| OP-26 | ~~Single vs split LLM endpoints — keep `POST /llm/query` for all intents, or split by operation family?~~ | **CLOSED** — Confirmed split: `POST /llm/meeting-summary` (UC-01) and `POST /llm/meeting-prep` (UC-02). UC-03 chatbot remains on `POST /llm/query`. |
| OP-28 | **Past meeting summary scope for meeting prep**: When passing `pastMeetingSummaries.summaryText` in `POST /llm/meeting-prep` requests, what portion of the prior FULL_SUMMARY should be included? Options: (A) overview paragraph only; (B) overview + discussion points; (C) full summary text. Trade-off is context richness vs token cost, particularly for customers with many prior meetings. Agree with AI team based on prompt requirements. | Pending AI team guidance |
| OP-27 | **Sub-section nesting in contentBlocks**: Current SECTION block supports one nesting level only (SECTION → leaf children). If LLM output requires Header → SubHeader → multiple content blocks under the same subheader, the schema cannot represent this cleanly. Two candidate solutions: (A) allow two-level SECTION nesting; (B) introduce a `GROUP` block type (subheader container, title + leaf children only, nestable inside SECTION but not recursive). Most likely to surface in meeting prep for existing customers. Decision should be deferred until AI team output patterns are observed in practice. | Candidate — assess against actual LLM output before extending schema |

---

## 14. Suggested Additions (Requires Review)

The following items are not in the business requirements or design doc but are recommended based on analysis. Review and approve/reject each:

| ID | Suggestion | Rationale |
|----|-----------|-----------|
| SUG-01 | Define P50/P90/P99 latency targets for each endpoint | Needed for capacity planning and SLA with AI team |
| SUG-02 | Promo retrieval as non-blocking enrichment | Meeting prep should degrade gracefully if promo API is slow |
| SUG-03 | Degraded-mode definition (e.g., chatbot unavailable, summary delayed) | Agents need a clear experience when AI services are partially unavailable |
| SUG-04 | Language detection fallback strategy | Code-switching transcripts need a defined handling path when detection is inconclusive |
| SUG-05 | Caching strategy for stable knowledge responses | Product FAQs change infrequently; caching reduces LLM cost and latency |
| SUG-06 | Audit trail for AI-generated content edits | MAS/BNM compliance may require an audit of what the agent edited before saving |
