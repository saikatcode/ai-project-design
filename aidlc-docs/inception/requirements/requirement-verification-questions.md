# Requirements Verification Questions (Revised)

I have now read both your technical design document and the Advisor 2.0 business requirements.
The questions below reflect only what is **still unclear or missing** after reading both documents.

Pre-answered items from the business requirements are noted below for transparency.

---

## Pre-answered by Business Requirements (no action needed)

| Question | What was answered |
|----------|------------------|
| Async vs sync transcription | Confirmed async — recording → backend transcription + summarisation → push notification |
| Meeting prep scenarios | Three confirmed: new customer / existing with meetings / existing without meetings |
| Transcription modes | Three confirmed: no customer context / with customer context / voice recap |
| SG vs MY differentiation | Confirmed — 14 FLP attribute categories for SG, 15 for MY |
| Guardrails | Confirmed — no product push, follow GE policy, no inferred action items |
| Short recap sharing | Confirmed — WhatsApp / Telegram via native share |
| Action items default deadline logic | Confirmed — weekday+1, Friday+3, Saturday+2 |

---

## Still Open — Please Answer

When done, let me know in the chat.

---

## Section A — Scope and Release

## Question 1
The business requirements cover Meeting Transcription/Summary and Meeting Prep. Is the **Agent Chatbot** (ROSE, as mentioned in the nav bar) also in scope for this first release, or is it a separate initiative?

A) Chatbot is in scope for the same first release — design all three together
B) Chatbot is a separate release — focus only on Meeting Transcription/Summary and Meeting Prep for now
C) Chatbot design should be captured in parallel but not built in the first release
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 2
Will this first release be a **controlled pilot** or a **full rollout** to all agents across SG and MY?

A) Controlled pilot — limited group of agents, one market first
B) Simultaneous rollout to all agents in both SG and MY
C) SG first, then MY — phased by market
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Section B — Performance and Availability

## Question 3
The business requirements show a push notification when the summary is ready, implying agents will wait. What is the **expected turnaround time** for transcription + summarisation to complete (from when the agent stops recording)?

A) Under 2 minutes — near-real-time
B) 2–5 minutes — acceptable wait with push notification
C) Up to 15 minutes — acceptable for longer meetings
D) No target set yet — to be agreed with the AI team
E) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 4
What is the **availability expectation** for the AI services (transcription, summarisation, meeting prep)?

A) Must be available during business hours only — agents work set hours
B) High availability 24/7 — agents may work evenings and weekends
C) Best-effort for now — formal SLA to be agreed with the AI team later
D) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Section C — Language

## Question 5
What **language(s)** must the app and AI-generated content support?

A) English only — all agents and customers interact in English
B) English + Bahasa Melayu — primarily for MY market
C) English + Mandarin Chinese — primarily for SG market
D) All three: English, Malay, Mandarin
E) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 6
Can meeting **transcripts contain mixed languages** (e.g., agent speaks English, customer responds in Mandarin or Malay)?

A) Yes — code-switching is common, transcription must handle it
B) No — meetings are conducted in a single language
C) Possible but rare — treat as best-effort for now
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Section D — Security and Integration

## Question 7
How will the **Pega app authenticate to the Orchestrator** when calling the AI business APIs (meeting prep, meeting summary)?

A) Service account / OAuth2 client credentials (service-to-service, no user token propagation)
B) The agent's SSO / JWT token is forwarded from Pega to the Orchestrator
C) API key per environment (shared secret)
D) Via enterprise API gateway only — Pega never calls Orchestrator directly
E) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 8
Will the Orchestrator be called **directly by Pega**, or routed through an **enterprise API gateway** (Axway T2 was mentioned in your design doc)?

A) Direct — Pega calls the Orchestrator service-to-service
B) All calls go through an enterprise API gateway (e.g., Axway T2)
C) API gateway for external/public endpoints; direct for internal ones
D) Not yet decided
E) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Section E — Regulatory and Data

## Question 9
Are there **regulatory compliance requirements** from MAS (Singapore) or BNM (Malaysia) that apply specifically to AI-generated content in the app (e.g., AI-generated financial guidance disclosures, mandatory disclaimers)?

A) Yes — specific MAS and/or BNM AI guidelines must be followed
B) General PDPA (data privacy) compliance only — no AI-specific regulations identified yet
C) Currently under review with the compliance team — not yet confirmed
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10
Are there **data residency restrictions** on where the audio recordings, transcripts, and prompts can be processed or stored?

A) Yes — must remain within Singapore (in-country processing)
B) Yes — must remain within Malaysia (in-country processing)
C) Regional cloud is acceptable — SG or MY cloud regions of approved cloud providers
D) No specific restrictions confirmed yet
E) Other (please describe after [Answer]: tag below)

[Answer]: A: We will use AWS services hosted in private VPC and for LLM, we will use self hosted model. Moreover, the LLM layer is responsibility of AI team, not in my direct responsibility.

---

## Section F — Orchestration and Governance

## Question 11
What is the **confirmed or preferred technology stack** for the Orchestrator service?

A) Java / Spring Boot (as noted in your design doc)
B) Node.js / TypeScript
C) Python (FastAPI or equivalent)
D) Not yet decided — pending technical evaluation
E) Other (please describe after [Answer]: tag below)

[Answer]: C: Python deployed on AWS is preferred.

---

## Question 12
Who **owns and governs the Intent Registry** (the configuration that maps intents to tools, prompts, and model profiles)?

A) Your team (UI/Orchestration) owns it — stored and deployed with the Orchestrator
B) The external AI/data team owns it
C) Jointly governed — shared config repo with approval gates
D) Not yet decided
E) Other (please describe after [Answer]: tag below)

[Answer]: C: Stored & Deployed by with the Orchestrator, but different mappings are defined by different teams eg. Prompt and model selection to be defined by AI team, intent, subintent and output structure to be defined by business.

---

## Section G — Meeting Prep Data Sources

## Question 13
The business requirements reference **customer profile, past meeting summaries, and policy data** for meeting prep generation. Which systems will the Orchestrator directly query for this data?

A) Orchestrator queries Pega directly — Pega aggregates and passes all context in the request
B) Orchestrator queries GP / CRM / policy systems directly via their APIs
C) Pega assembles the context payload and passes it to the Orchestrator — Orchestrator does not reach back to enterprise systems
D) Not yet decided — data sourcing strategy to be defined
E) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 14
The business requirements mention **relevant promos** (AD-285) displayed alongside meeting prep. Where do these come from, and should the Orchestrator serve them or does Pega source them independently?

A) Pega sources promos directly from the backend — the Orchestrator is not involved
B) Orchestrator fetches and includes promo data as part of meeting prep generation
C) Promos are a separate Pega component — displayed alongside but not part of the AI-generated prep
D) Not yet decided
E) Other (please describe after [Answer]: tag below)

[Answer]: B

---

*Please answer all questions and let me know in chat when done.*
*If unsure about any answer, pick the closest option and add a note after [Answer]:*
