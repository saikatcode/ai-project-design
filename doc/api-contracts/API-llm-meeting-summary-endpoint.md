# API Contract: POST /llm/meeting-summary

**Version**: 1.0 — Draft for Review
**Owner**: Orchestration Team
**AI Team Endpoint**: Provided by AI Team; Orchestrator is the sole caller
**Scope**: Meeting Transcription & Summary (UC-01)
**Last Updated**: 2026-04-14

---

## 1. Overview

`POST /llm/meeting-summary` handles all LLM operations related to meeting audio processing and summarisation. Operations are distinguished by `subIntent` in the request. The Orchestrator resolves subIntent before calling this endpoint and assembles all context.

**Caller flow**: Pega → Axway T2 → Orchestrator → AWS API Gateway → `POST /llm/meeting-summary` (AI Team)

> **Note**: Two API gateways are in the call chain with different roles. Axway T2 is the enterprise API gateway for inbound calls (Pega → Orchestrator). AWS API Gateway is the AI team's outbound boundary (Orchestrator → LLM services) — it authenticates and routes calls into the AI team's private VPC. Authentication mechanism for AWS API Gateway calls (API key, SigV4, or custom authorizer) to be confirmed with AI team.

### SubIntent Registry

| `subIntent` | Operation | UC |
|---|---|---|
| `SNIPPET` | Generate per-speaker snippets for agent tagging screen | UC-01 |
| `FULL_SUMMARY` | Generate meeting summary + FLP attributes + soft attributes + action items | UC-01 |
| `SHARABLE_SUMMARY` | Generate customer-shareable WhatsApp-format recap from agent-edited output | UC-01 |

**Flow order**: SNIPPET (post-transcription, pre-tagging) → FULL_SUMMARY (post-tagging) → SHARABLE_SUMMARY (agent-triggered, post-edit)

---

## 2. Master Request Contract

```json
{
  "requestId": "string — unique ID for this API call",
  "interactionId": "string — correlates all service calls within one user interaction",
  "timestamp": "ISO 8601 datetime with timezone",

  "actor": "DISTRIBUTOR | CUSTOMER_SUPPORT | CUSTOMER",
  "domain": "SALES | RECRUITMENT | SERVICE | CLAIMS | UNDERWRITING",

  "intent": "MEETING_SUMMARY",
  "subIntent": "SNIPPET | FULL_SUMMARY | SHARABLE_SUMMARY",

  "userInstruction": "string | null — free-text instruction from the end user; null for all UC-01 fixed-intent flows",

  "meetingContext": {
    "meetingId": "string",
    "meetingType": "VOICE_RECAP | STANDARD_TRANSCRIPT | VOICE_RECAP_WITH_NOTES — identifies input source type for prompt routing",
    "transcript": "string | null — reconstructed diarized transcript produced by Orchestrator from AWS Transcribe output (see OP-25); code-switching supported; null for SHARABLE_SUMMARY",
    "writtenNotes": "string | null — advisor text notes included in LLM input for FULL_SUMMARY (VOICE_RECAP_WITH_NOTES) only; null for all other subIntents",
    "language": "en | ms | zh | mixed",
    "agentSpeakerId": "string | null — speaker ID of the advisor (e.g. spk_0); required for STANDARD_TRANSCRIPT to exclude advisor speech from snippet generation; null for VOICE_RECAP",
    "speakerMapping": [
      {
        "speakerId": "string — e.g. spk_1, spk_2",
        "customerId": "string — confirmed by agent tagging step; null if not yet tagged"
      }
    ],
    "editedSummaryText": "string | null — agent-reviewed and edited meeting summary text; populated for SHARABLE_SUMMARY only; null for all other subIntents"
  },

  "customerContext": [
    {
      "customerId": "string",
      "customerName": "string",
      "speakerId": "string | null — links to speakerMapping entry",
      "profile": "null — not used for any MEETING_SUMMARY subIntent (confirmed with AI team)"
    }
  ],

  "agentContext": {
    "agentId": "string",
    "agentName": "string"
  },

  "outputSchema": {
    "schemaVersion": "string",
    "allowedBlockTypes": ["TEXT", "BULLET_LIST", "KEY_VALUE", "SNIPPET", "SECTION"],
    "responseFormat": "JSON"
  },

  "requestMetadata": {
    "userId": "string — agent user ID",
    "locale": "en-SG | en-MY | ms-MY | zh-SG",
    "clientApp": "ADVISOR_APP_IOS | ADVISOR_APP_ANDROID"
  }
}
```

### Field Applicability by SubIntent

| Field | `SNIPPET` | `FULL_SUMMARY` | `SHARABLE_SUMMARY` |
|---|---|---|---|
| `actor` | Required | Required | Required |
| `domain` | Required | Required | Required |
| `userInstruction` | null | null | null |
| `meetingContext` | Required | Required | Required (meetingId + speakerMapping only) |
| `meetingContext.meetingType` | Required | Required | Not sent |
| `meetingContext.transcript` | Required | Required | Not sent |
| `meetingContext.writtenNotes` | Not sent | Optional (VOICE_RECAP_WITH_NOTES only) | Not sent |
| `meetingContext.agentSpeakerId` | Required (STANDARD_TRANSCRIPT only) | Not sent | Not sent |
| `meetingContext.speakerMapping` | Not sent (pre-tagging) | Required | Required |
| `customerContext` | Not sent | Required | Required |
| `customerContext.profile` | Not sent | Not sent | Not sent |
| `agentContext` | Required | Required | Required |
| `meetingContext.editedSummaryText` | Not sent | Not sent | Required |

---

## 3. Master Response Contract

```json
{
  "requestId": "string — echoes request ID",
  "interactionId": "string — echoes interaction correlation ID",
  "timestamp": "ISO 8601 datetime with timezone",
  "status": "SUCCESS | PARTIAL_SUCCESS | FAILED",
  "intent": "MEETING_SUMMARY",
  "subIntent": "string — echoes resolved subIntent",

  "response": {
    "title": "string — top-level label for this response",
    "contentBlocks": [
      {
        "blockId": "string — unique block identifier within this response",
        "type": "TEXT | BULLET_LIST | KEY_VALUE | SNIPPET | SECTION",
        "title": "string — section heading",
        "body": "string | null — populated for TEXT type only; markdown supported",
        "data": "object | null — populated for BULLET_LIST, KEY_VALUE, SNIPPET leaf types",
        "childBlocks": "array | null — populated for SECTION type only; children cannot be SECTION"
      }
    ],
    "tasks": [
      {
        "taskHeader": "string — short description of the advisor action item",
        "dueDate": "ISO 8601 date | null"
      }
    ]
  },

  "usage": {
    "inputTokens": "number",
    "outputTokens": "number"
  },

  "debug": {
    "promptTemplateId": "string — template resolved and used by AI team",
    "promptTemplateVersion": "string — version of the template used",
    "model": "string — LLM model identifier used for this call"
  },

  "error": {
    "code": "string",
    "message": "string",
    "retryable": "boolean"
  }
}
```

> **`tasks[]` population by subIntent**: Populated for `FULL_SUMMARY` only (advisor-only action items for dashboard/Tasks system). `null` for `SNIPPET` (pre-tagging, no summary yet) and `SHARABLE_SUMMARY` (downstream formatting step, no new tasks generated).

### Block Type Schemas

**`TEXT`**
```json
{ "body": "string — markdown supported" }
```
Used for: Overview narrative, shareable summary WhatsApp message.

**`BULLET_LIST`**
```json
{ "items": [{ "text": "string" }] }
```
Used for: Discussion points, all-action-items display list (inside SECTION "Action Items").

**`KEY_VALUE`**
```json
{
  "items": [
    {
      "category": "string — must match taxonomy category name exactly (e.g. Income, CPF OA Balance)",
      "label": "string — attribute label",
      "value": "string — extracted value; sparse — only attributes mentioned in transcript are returned"
    }
  ]
}
```
Used for: FLP attributes (up to 90, from taxonomy), soft attributes (up to 40, from taxonomy).

**`SNIPPET`**
```json
{
  "speakerId": "string — e.g. spk_1",
  "customerRef": "string — e.g. Customer 1",
  "snippet": "string — third-person description unique to this speaker"
}
```
Used for: Per-speaker snippets on the agent tagging screen.

**`SECTION`** (container — one level only)
```json
{
  "title": "string — section header",
  "childBlocks": [
    { "type": "BULLET_LIST | KEY_VALUE | TEXT | SNIPPET", "title": "...", "data": "..." }
  ]
}
```
Used for: Action items grouping (all items as BULLET_LIST), attribute grouping per customer (one KEY_VALUE child per customer). Children cannot be `SECTION`.

### `BULLET_LIST` vs `tasks[]` — action items pattern

For `FULL_SUMMARY`, action items are represented in two shapes for different consumers:

| | `BULLET_LIST` in SECTION "Action Items" | `tasks[]` array |
|---|---|---|
| Purpose | UI display of all action items (customer + advisor) inline on the meeting summary screen | Structured feed to the Tasks/dashboard system — advisor to-dos only |
| Scope | All parties — customer items prefixed with customer name, advisor items prefixed with "Advisor" | Advisor-only items |
| Format | SECTION "Action Items" → child BULLET_LIST (inside `contentBlocks`) | Top-level `response.tasks[]` |
| Consumers | Advisor App UI | Tasks/dashboard service |

---

## 4. Sample — `SNIPPET`

**Scenario**: Voice recap (single customer, Marcus). ASR complete. Agent has not yet tagged speakers. Orchestrator calls `/llm/meeting-summary` to generate per-speaker snippets for the tagging screen.

### Request

```json
{
  "requestId": "req-20260414-001",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T19:45:00+08:00",
  "actor": "DISTRIBUTOR",
  "domain": "SALES",
  "intent": "MEETING_SUMMARY",
  "subIntent": "SNIPPET",
  "userInstruction": null,
  "meetingContext": {
    "meetingId": "mtg-1001",
    "meetingType": "VOICE_RECAP",
    "transcript": "Okay ah, just met Marcus just now at Tanjong Pagar, quick voice note so I don't forget. Uhh so he just changed job recently, like about 3 months ago. From banking go to fintech startup, so now his pay structure a bit different. Last time he was earning about 9.5k plus bonus maybe 3 months, now base around 8k but got equity la… so I think he also not very sure how stable long term. Then biggest thing is he just bought a condo last year, loan quite big sia… about 1.1 mil. Monthly like 4.2k, mostly using CPF. His OA now left about 60k only, SA around 110k which is actually quite okay. Savings wise got about 70k cash, investments around 180k… he doing DCA like 1.5k every month into ETF and some tech stocks. So actually financially quite decent la, just that his protection side really quite weak. He only got hospital plan with rider, AIA one, then term plan only 300k. No CI at all… which I think quite dangerous especially now he in startup, income not stable. Also ah, he quite active — gym and recently do Muay Thai. He say last time got ankle injury but never claim anything because no personal accident. So that one quite obvious gap. Then personal side… he got girlfriend, about 1 year already, quite serious. He mention maybe 2–3 years will get married, then maybe kids few years later.",
    "writtenNotes": null,
    "language": "mixed",
    "agentSpeakerId": null,
    "speakerMapping": []
  },
  "agentContext": {
    "agentId": "agt-101",
    "agentName": "Nisha"
  },
  "outputSchema": {
    "schemaVersion": "1.0",
    "allowedBlockTypes": ["SNIPPET"],
    "responseFormat": "JSON"
  },
  "requestMetadata": {
    "userId": "agt-101",
    "locale": "en-SG",
    "clientApp": "ADVISOR_APP_IOS"
  }
}
```

### Response

```json
{
  "requestId": "req-20260414-001",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T19:45:08+08:00",
  "status": "SUCCESS",
  "intent": "MEETING_SUMMARY",
  "subIntent": "SNIPPET",
  "response": {
    "title": "Speaker Snippets",
    "contentBlocks": [
      {
        "blockId": "blk-001",
        "type": "SNIPPET",
        "title": "Customer 1",
        "body": null,
        "data": {
          "speakerId": "spk_1",
          "customerRef": "Customer 1",
          "snippet": "Recently transitioned from banking to a fintech startup (~3 months), with a shift to lower base pay (~8k) plus equity and some uncertainty around income stability. Holds a large condo loan (~$1.1M, ~$4.2k/month, mainly CPF), maintains ~70k cash and ~180k in investments with regular $1.5k monthly DCA, but only has basic hospitalisation and ~300k term coverage with no CI or accident protection. Leads an active lifestyle (gym, Muay Thai) with prior minor injury, and is in a serious relationship (~1 year) with plans for marriage in 2–3 years and children thereafter."
        },
        "childBlocks": null
      }
    ],
    "tasks": null
  },
  "usage": {
    "inputTokens": 312,
    "outputTokens": 98
  },
  "debug": {
    "promptTemplateId": "tpl-snippet-v1",
    "promptTemplateVersion": "1.0.0",
    "model": "ge-llm-v2"
  },
  "error": null
}
```

> `tasks: null` — no tasks generated at SNIPPET stage; summary has not yet been produced.

**Two-customer variant** (John and Sarah, voice recap): `contentBlocks` contains two SNIPPET blocks with `speakerId: "spk_1"` and `speakerId: "spk_2"` respectively:

```json
[
  {
    "blockId": "blk-001", "type": "SNIPPET", "title": "Customer 1",
    "data": {
      "speakerId": "spk_1", "customerRef": "Customer 1",
      "snippet": "Recently moved into a tech product manager role earning ~9k with reduced bonus upside, has lower CPF OA due to housing usage, actively cycles every weekend (East Coast) with a past minor wrist injury, holds ~500k term coverage with weaker/older CI, and is planning ahead for daughter's university while considering a future condo upgrade."
    }
  },
  {
    "blockId": "blk-002", "type": "SNIPPET", "title": "Customer 2",
    "data": {
      "speakerId": "spk_2", "customerRef": "Customer 2",
      "snippet": "MOE teacher earning ~5.5k with potential increase to ~6.5k from upcoming HOD role, has stronger CPF balances, more conservative lifestyle (occasionally joins cycling), relies partly on basic group insurance, and is closely involved in planning and supporting daughter's upcoming university education."
    }
  }
]
```

---

## 5. Sample — `FULL_SUMMARY`

**Scenario**: Standard transcript (one customer, Marcus). Agent has completed speaker tagging (spk_1 → Marcus). Orchestrator calls `/llm/meeting-summary` for the full summary output. FLP and soft attribute taxonomies are managed within the AI team's prompt template.

### Request

```json
{
  "requestId": "req-20260414-002",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T19:52:00+08:00",
  "actor": "DISTRIBUTOR",
  "domain": "SALES",
  "intent": "MEETING_SUMMARY",
  "subIntent": "FULL_SUMMARY",
  "userInstruction": null,
  "meetingContext": {
    "meetingId": "mtg-1001",
    "meetingType": "STANDARD_TRANSCRIPT",
    "transcript": "[spk_0] Hey Marcus, long time no see. You still working around Tanjong Pagar area ah?\n[spk_1] Ya still around here, but office now hybrid already. Today I just come out for meeting.\n[spk_0] Shiok lah, hybrid life. I got your iced long black, no sugar right?\n[spk_1] Wah thanks, you remember ah.\n[spk_1] Ya actually I just switched job three months ago. Still finance, but now I moved to a fintech startup.\n[spk_1] Ya exactly. My base now is about 8k, slightly lower than before, but got equity component.\n[spk_1] Ya lor. Last time I was drawing about 9.5k with bonus maybe 3 months. Now bonus not guaranteed.\n[spk_1] I just bought a resale condo last year. About 1.1 million loan. Monthly repayment around 4.2k.\n[spk_1] Mostly CPF OA. My OA now left about 60k. SA around 110k.\n[spk_1] Ya I'm putting about $1.5k monthly into ETFs and some tech stocks. Around 180k total.\n[spk_1] About 70k liquid.\n[spk_1] Ya I go gym like 3 times a week. Recently also started doing Muay Thai.\n[spk_1] Ya lor, last time I injured my ankle slightly already. No leh, I don't have personal accident.\n[spk_1] I only have hospital plan with AIA, with rider. Got one term plan, about 300k coverage. Don't have CI.\n[spk_1] I'm attached now. We've been together about 1 year plus. Ya quite serious. We're thinking maybe in 2-3 years settle down. Maybe 3-4 years later for kids.\n[spk_0] Next step, can you send me your existing policy details, latest CPF summary, rough breakdown of your monthly expenses.\n[spk_1] Okay can.\n[spk_0] Then I prepare a proper proposal for you. Thursday evening works?",
    "writtenNotes": null,
    "language": "mixed",
    "agentSpeakerId": "spk_0",
    "speakerMapping": [
      { "speakerId": "spk_1", "customerId": "cust-marcus-001" }
    ]
  },
  "customerContext": [
    {
      "customerId": "cust-marcus-001",
      "customerName": "Marcus",
      "speakerId": "spk_1",
      "profile": null
    }
  ],
  "agentContext": {
    "agentId": "agt-101",
    "agentName": "Nisha"
  },
  "outputSchema": {
    "schemaVersion": "1.0",
    "allowedBlockTypes": ["TEXT", "BULLET_LIST", "KEY_VALUE", "SECTION"],
    "responseFormat": "JSON"
  },
  "requestMetadata": {
    "userId": "agt-101",
    "locale": "en-SG",
    "clientApp": "ADVISOR_APP_IOS"
  }
}
```

### Response

```json
{
  "requestId": "req-20260414-002",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T19:52:34+08:00",
  "status": "SUCCESS",
  "intent": "MEETING_SUMMARY",
  "subIntent": "FULL_SUMMARY",
  "response": {
    "title": "Meeting Summary — Marcus",
    "contentBlocks": [
      {
        "blockId": "blk-001",
        "type": "TEXT",
        "title": "Overview",
        "body": "This was a post-career transition review with Marcus to understand his updated financial situation and assess whether his current plans are still adequate.\n\n- **Purpose of Meeting**: Review of Marcus's financial and protection needs following a career change and recent property purchase.\n- **Type of Review**: Financial and insurance coverage assessment.\n- **Who Attended**: Marcus and the advisor.\n- **Current Stage of Conversation**: Initial discussion on gaps in protection and financial planning.\n- **Tone and Sentiments**: Open and concerned about cash flow and stability.\n- **Overall Insights**: Marcus is financially stable but has significant gaps in protection, especially given his new income structure and upcoming responsibilities.",
        "data": null,
        "childBlocks": null
      },
      {
        "blockId": "blk-002",
        "type": "BULLET_LIST",
        "title": "Discussion Points",
        "body": null,
        "data": {
          "items": [
            { "text": "Career change from banking to fintech startup (3 months ago)." },
            { "text": "Income structure changed: Base salary ~8k with equity, previously 9.5k plus bonus." },
            { "text": "Recent condo purchase: loan ~$1.1M, repayment ~$4.2k/month via CPF." },
            { "text": "CPF balances: OA ~60k, SA ~110k." },
            { "text": "Savings: ~70k cash; investments ~180k (DCA $1.5k/month into ETFs and tech stocks)." },
            { "text": "Protection gaps: Hospital plan with rider (AIA), term plan ~$300k only. No CI, no personal accident coverage." },
            { "text": "Active lifestyle: Gym and Muay Thai. Previous ankle injury — no personal accident coverage." },
            { "text": "Personal life: Serious relationship (~1 year); plans to marry in 2–3 years, children ~3–4 years later." }
          ]
        },
        "childBlocks": null
      },
      {
        "blockId": "blk-003",
        "type": "SECTION",
        "title": "Action Items",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "All Action Items",
            "data": {
              "items": [
                { "text": "Customer — Marcus: Send existing policy details to advisor." },
                { "text": "Customer — Marcus: Share latest CPF summary." },
                { "text": "Customer — Marcus: Provide rough breakdown of monthly expenses." },
                { "text": "Advisor: Perform proper financial calculations based on received documents." },
                { "text": "Advisor: Prepare and present proposal at next meeting (Thursday evening)." }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-004",
        "type": "SECTION",
        "title": "FLP Attributes",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "KEY_VALUE",
            "title": "Marcus",
            "data": {
              "items": [
                { "category": "Income", "label": "Current Monthly Income", "value": "~8k/month + equity component" },
                { "category": "Income", "label": "Previous Monthly Income", "value": "~9.5k/month + ~3 months annual bonus" },
                { "category": "CPF OA Balance", "label": "CPF OA", "value": "~60k" },
                { "category": "CPF SA Balance", "label": "CPF SA", "value": "~110k" },
                { "category": "Investments", "label": "Investment Portfolio", "value": "~180k (ETFs + tech stocks; $1.5k/month DCA)" },
                { "category": "Capital Assets", "label": "Residential Property", "value": "Resale condo; loan ~$1.1M; repayment ~$4.2k/month via CPF" },
                { "category": "Cash Savings", "label": "Liquid Savings", "value": "~70k" },
                { "category": "Insurance Coverage", "label": "Existing Policies", "value": "Hospital plan with rider (AIA); Term life ~$300k; No CI; No personal accident" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-005",
        "type": "SECTION",
        "title": "Soft Attributes",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "KEY_VALUE",
            "title": "Marcus",
            "data": {
              "items": [
                { "category": "Career Change", "label": "Recent Job Change", "value": "Banking to fintech startup (~3 months ago); income less stable; equity component" },
                { "category": "Sports and Fitness Preferences", "label": "Physical Activities", "value": "Gym (3x/week); Muay Thai (recent); previous ankle injury" },
                { "category": "Marital or Relationship Dynamics", "label": "Relationship Status", "value": "Serious relationship (~1 year); plans to marry in 2–3 years; children ~3–4 years later" }
              ]
            }
          }
        ]
      }
    ],
    "tasks": [
      {
        "taskHeader": "Perform proper financial calculations based on received documents from Marcus",
        "dueDate": null
      },
      {
        "taskHeader": "Prepare and present proposal to Marcus (Thursday evening meeting)",
        "dueDate": "2026-04-17"
      }
    ]
  },
  "usage": {
    "inputTokens": 1240,
    "outputTokens": 680
  },
  "debug": {
    "promptTemplateId": "tpl-full-summary-v1",
    "promptTemplateVersion": "1.0.0",
    "model": "ge-llm-v2"
  },
  "error": null
}
```

> `tasks[]` contains advisor-only action items in a structured format for the dashboard/Tasks system. Customer action items (e.g., "Customer — Marcus: Send policy details") appear in the BULLET_LIST within `contentBlocks` but are NOT included in `tasks[]`.

**Multi-customer note**: When `numCustomers` > 1, each SECTION "FLP Attributes" and "Soft Attributes" contains one `KEY_VALUE` child block per customer (e.g., "John" and "Sarah"). The SECTION "Action Items" BULLET_LIST prefixes each item with the customer name (e.g., `"Customer — John: ..."`, `"Customer — Sarah: ..."`). The FULL_SUMMARY Overview TEXT block is unified across all customers.

**Voice recap + notes variant**: When `meetingType = VOICE_RECAP_WITH_NOTES`, `meetingContext.writtenNotes` is populated alongside `transcript`. The Orchestrator includes both in the prompt context. Response structure is identical.

---

## 6. Sample — `SHARABLE_SUMMARY`

**Scenario**: Agent has reviewed, edited, and saved the meeting summary for Marcus. Agent taps "Create shareable summary". Orchestrator passes the agent-edited summary text in `meetingContext.editedSummaryText` and calls `/llm/meeting-summary`.

### Request

```json
{
  "requestId": "req-20260414-003",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T20:10:00+08:00",
  "actor": "DISTRIBUTOR",
  "domain": "SALES",
  "intent": "MEETING_SUMMARY",
  "subIntent": "SHARABLE_SUMMARY",
  "userInstruction": null,
  "meetingContext": {
    "meetingId": "mtg-1001",
    "speakerMapping": [
      { "speakerId": "spk_1", "customerId": "cust-marcus-001" }
    ],
    "editedSummaryText": "Post-career transition review with Marcus. Discussed: recent job change (banking to fintech startup), condo loan ($1.1M), protection gaps (no CI, no personal accident). Action items: Marcus to send policy details, CPF summary, expense breakdown. Advisor to prepare proposal. Next meeting: Thursday evening."
  },
  "customerContext": [
    {
      "customerId": "cust-marcus-001",
      "customerName": "Marcus",
      "speakerId": "spk_1",
      "profile": null
    }
  ],
  "agentContext": {
    "agentId": "agt-101",
    "agentName": "Nisha"
  },
  "outputSchema": {
    "schemaVersion": "1.0",
    "allowedBlockTypes": ["TEXT"],
    "responseFormat": "JSON"
  },
  "requestMetadata": {
    "userId": "agt-101",
    "locale": "en-SG",
    "clientApp": "ADVISOR_APP_IOS"
  }
}
```

### Response

```json
{
  "requestId": "req-20260414-003",
  "interactionId": "int-mtg-1001",
  "timestamp": "2026-04-14T20:10:05+08:00",
  "status": "SUCCESS",
  "intent": "MEETING_SUMMARY",
  "subIntent": "SHARABLE_SUMMARY",
  "response": {
    "title": "Shareable Summary — Marcus",
    "contentBlocks": [
      {
        "blockId": "blk-001",
        "type": "TEXT",
        "title": "WhatsApp Message",
        "body": "Hey Marcus, great catching up with you today.\n\nJust to recap, we went through your recent job change, your current housing commitments, and how your overall financial position has evolved. We also discussed your existing coverage and highlighted a few areas that may need a closer review, especially around protection for income, health, and future responsibilities.\n\nAs a next step, please send me:\n- Your existing policy details\n- Your CPF summary\n- A rough breakdown of your monthly expenses\n\nOnce I have that, I'll put together a proper review and proposal for us to go through next. Looking forward to seeing you on Thursday — feel free to ping me if anything comes up before then!",
        "data": null,
        "childBlocks": null
      }
    ],
    "tasks": null
  },
  "usage": {
    "inputTokens": 280,
    "outputTokens": 145
  },
  "debug": {
    "promptTemplateId": "tpl-sharable-summary-v1",
    "promptTemplateVersion": "1.0.0",
    "model": "ge-llm-v2"
  },
  "error": null
}
```

> `tasks: null` — SHARABLE_SUMMARY is a downstream formatting step triggered after the agent has already saved the summary. Tasks were already generated and propagated to the dashboard during the FULL_SUMMARY call. No new tasks are produced here.

---

## 7. Open Points

| ID | Question | Status |
|---|---|---|
| OP-25 | **Transcript reconstruction**: Orchestrator must reconstruct a diarized text transcript from the AWS Transcribe response (merging `items[]` word-level tokens with `speaker_labels.segments[]` timestamp ranges) before calling `/llm/meeting-summary`. Need to: (1) agree reconstruction output format with AI team (e.g., `[spk_0] text \n [spk_1] text`), (2) define error handling for poor diarization quality, (3) design and build as a dedicated Orchestrator component. Candidate for ADR. | Design decision pending |
| OP-11 | Snippet length — approximate word/character limit per speaker snippet | Pending AI team guidance |
| OP-27 | **Sub-section nesting in contentBlocks**: The current SECTION block supports one level of nesting only (SECTION → leaf children). If LLM output requires a Header → SubHeader → content pattern where a single SubHeader contains more than one content block, the current schema cannot represent this cleanly. Two options: (A) allow two-level SECTION nesting; (B) introduce a new `GROUP` block type as a subheader-level container. Most likely to surface in meeting prep (`POST /llm/meeting-prep`) but may also apply to future FULL_SUMMARY sections. Confirm with AI team whether their output naturally produces this pattern before extending the schema. | Candidate — assess against actual LLM output |
| OP-26 | ~~Single vs split endpoints~~ | **Closed** — confirmed split: `/llm/meeting-summary` (UC-01) and `/llm/meeting-prep` (UC-02) |

---

*Samples are based on business-provided input/output scenarios from the Sample Input Output CSV. Validate field names, data types, and prompt template IDs with the AI team before finalising.*
