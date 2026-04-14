# API Contract: POST /llm/meeting-prep

**Version**: 1.0 — Draft for Review
**Owner**: Orchestration Team
**AI Team Endpoint**: Provided by AI Team; Orchestrator is the sole caller
**Scope**: Meeting Preparation Notes (UC-02)
**Last Updated**: 2026-04-14

---

## 1. Overview

`POST /llm/meeting-prep` generates AI-assisted meeting preparation notes for an upcoming customer meeting. Operations are distinguished by `subIntent`, which reflects how much customer history is available.

**Caller flow**: Pega → Axway T2 → Orchestrator → `POST /llm/meeting-prep` (AI Team)

### SubIntent Registry

| `subIntent` | Scenario | Customer data available | UC |
|---|---|---|---|
| `NEW_CUSTOMER` | First meeting — no customer record in database | None | UC-02 |
| `EXISTING_WITH_MEETINGS` | Returning customer with prior meeting notes captured | Profile + past meeting summaries | UC-02 |
| `EXISTING_WITHOUT_MEETINGS` | Returning customer but no prior meeting notes captured | Profile only | UC-02 |

**Trigger**: Agent taps "Prepare for meeting" in the Advisor app before an upcoming appointment. Orchestrator determines subIntent based on CRM lookup results.

---

## 2. Master Request Contract

```json
{
  "requestId": "string — unique ID for this API call",
  "interactionId": "string — correlates all service calls within one user interaction",
  "timestamp": "ISO 8601 datetime with timezone",

  "actor": "DISTRIBUTOR | CUSTOMER_SUPPORT | CUSTOMER",
  "domain": "SALES | RECRUITMENT | SERVICE | CLAIMS | UNDERWRITING",

  "intent": "MEETING_PREP_NOTES",
  "subIntent": "NEW_CUSTOMER | EXISTING_WITH_MEETINGS | EXISTING_WITHOUT_MEETINGS",

  "modelConfig": {
    "processingMode": "FAST | STANDARD | DEEP_THINK",
    "maxOutputTokens": "number"
  },

  "prompt": {
    "templateId": "string — references prompt template in Intent Registry",
    "templateVersion": "string — version for traceability and rollback",
    "variables": {
      "key": "value — key-value pairs substituted into the template at runtime (e.g. market, agentName, numCustomers)"
    },
    "systemInstruction": "string | null — optional override; use only when dynamic context cannot be expressed via variables",
    "developerInstruction": "string | null — optional override",
    "userInstruction": "string | null — optional override"
  },

  "customerContext": [
    {
      "customerId": "string | null — null for NEW_CUSTOMER",
      "customerName": "string | null — null for NEW_CUSTOMER",
      "profile": {
        "age": "number | null",
        "occupation": "string | null",
        "familyStatus": "string | null",
        "existingPolicies": ["string"]
      },
      "pastMeetingSummaries": [
        {
          "meetingId": "string",
          "meetingDate": "ISO 8601 date",
          "summaryText": "string"
        }
      ]
    }
  ],

  "agentContext": {
    "agentId": "string",
    "agentName": "string"
  },

  "meetingPrepConfig": {
    "promoData": [
      {
        "promoId": "string",
        "title": "string",
        "description": "string"
      }
    ]
  },

  "outputSchema": {
    "schemaVersion": "string",
    "allowedBlockTypes": ["SECTION", "TEXT", "BULLET_LIST"],
    "responseFormat": "JSON"
  },

  "requestMetadata": {
    "userId": "string — agent user ID",
    "locale": "en-SG | en-MY | ms-MY | zh-SG",
    "clientApp": "ADVISOR_APP_IOS | ADVISOR_APP_ANDROID"
  }
}
```

> **Note**: `meetingContext` and `grounding` are not applicable to this endpoint and must not be sent.

### Field Applicability by SubIntent

| Field | `NEW_CUSTOMER` | `EXISTING_WITH_MEETINGS` | `EXISTING_WITHOUT_MEETINGS` |
|---|---|---|---|
| `actor` | Required | Required | Required |
| `domain` | Required | Required | Required |
| `prompt.variables` | Required | Required | Required |
| `customerContext` | Empty array `[]` | Required | Required |
| `customerContext.profile` | Not sent | Required | Optional |
| `customerContext.pastMeetingSummaries` | Not sent | Required | Not sent |
| `agentContext` | Required | Required | Required |
| `meetingPrepConfig` | Required | Required | Required |
| `meetingPrepConfig.promoData` | Required (generic promos) | Required (contextualised promos) | Required (contextualised promos) |

---

## 3. Master Response Contract

```json
{
  "requestId": "string — echoes request ID",
  "interactionId": "string — echoes interaction correlation ID",
  "timestamp": "ISO 8601 datetime with timezone",
  "status": "SUCCESS | PARTIAL_SUCCESS | FAILED",
  "intent": "MEETING_PREP_NOTES",
  "subIntent": "string — echoes resolved subIntent",

  "response": {
    "title": "string — top-level label for this response",
    "contentBlocks": [
      {
        "blockId": "string — unique block identifier within this response",
        "type": "SECTION | TEXT | BULLET_LIST",
        "title": "string — section heading",
        "body": "string | null — populated for TEXT type only; markdown supported",
        "data": "object | null — populated for BULLET_LIST leaf type",
        "childBlocks": "array | null — populated for SECTION type only; children cannot be SECTION"
      }
    ],
    "tasks": null
  },

  "usage": {
    "model": "string",
    "inputTokens": "number",
    "outputTokens": "number"
  },

  "error": {
    "code": "string",
    "message": "string",
    "retryable": "boolean"
  }
}
```

> **Note**: `tasks` is always `null` for `POST /llm/meeting-prep`. Agent dashboard tasks are generated by `POST /llm/meeting-summary` (`FULL_SUMMARY`) from agreed action items captured during the meeting.

### Block Type Schemas (Meeting Prep)

**`TEXT`**
```json
{ "body": "string — markdown supported" }
```
Used for: Suggested openers, framing statements, narrative context.

**`BULLET_LIST`**
```json
{ "items": [{ "text": "string" }] }
```
Used for: Conversation starters, discussion questions, recommended topics, closing lines.

**`SECTION`** (container — one level only)
```json
{
  "title": "string — numbered section header (e.g. '1. Quick Check-In')",
  "childBlocks": [
    { "type": "TEXT | BULLET_LIST", "title": "...", "body/data": "..." }
  ]
}
```
Used for: Each numbered section of the meeting prep guide. Children cannot be `SECTION`.

> **Note**: `ACTION_LIST` and `KEY_VALUE` blocks are not applicable to this endpoint.

---

## 4. Sample — `NEW_CUSTOMER`

**Scenario**: Agent has an upcoming first meeting with a customer not yet in the database. Agent taps "Prepare for meeting". Orchestrator finds no customer record, sets `subIntent = NEW_CUSTOMER`, fetches active promos, and calls `/llm/meeting-prep`.

### Request

```json
{
  "requestId": "req-20260415-001",
  "interactionId": "int-prep-2001",
  "timestamp": "2026-04-15T08:30:00+08:00",
  "actor": "DISTRIBUTOR",
  "domain": "SALES",
  "intent": "MEETING_PREP_NOTES",
  "subIntent": "NEW_CUSTOMER",
  "modelConfig": {
    "processingMode": "STANDARD",
    "maxOutputTokens": 1500
  },
  "prompt": {
    "templateId": "tpl-meetingprep-new-v1",
    "templateVersion": "1.0.0",
    "variables": {
      "market": "SG",
      "agentName": "Nisha"
    },
    "systemInstruction": null,
    "developerInstruction": null,
    "userInstruction": null
  },
  "customerContext": [],
  "agentContext": {
    "agentId": "agt-101",
    "agentName": "Nisha"
  },
  "meetingPrepConfig": {
    "promoData": [
      {
        "promoId": "promo-2026-q2-001",
        "title": "Great Eastern Spring Saver",
        "description": "Limited-time endowment plan with guaranteed returns. Applicable for customers aged 25–55."
      }
    ]
  },
  "outputSchema": {
    "schemaVersion": "1.0",
    "allowedBlockTypes": ["SECTION", "TEXT", "BULLET_LIST"],
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
  "requestId": "req-20260415-001",
  "interactionId": "int-prep-2001",
  "timestamp": "2026-04-15T08:30:12+08:00",
  "status": "SUCCESS",
  "intent": "MEETING_PREP_NOTES",
  "subIntent": "NEW_CUSTOMER",
  "response": {
    "title": "First Meeting Guide — New Customer",
    "contentBlocks": [
      {
        "blockId": "blk-001",
        "type": "SECTION",
        "title": "1. Quick Check-In",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "TEXT",
            "title": "Simple Intro",
            "body": "> \"I usually help people organise their finances and make sense of things — more like a sounding board than anything.\""
          },
          {
            "type": "BULLET_LIST",
            "title": "Conversation Starters (local + relatable)",
            "data": {
              "items": [
                { "text": "\"Any travel plans coming up? Seems like everyone going Japan these days 😄\"" },
                { "text": "\"Have you been following the cost of living stuff? Everything getting pricier.\"" },
                { "text": "\"Weekends nowadays you more chill or still quite packed?\"" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-002",
        "type": "SECTION",
        "title": "2. Frame the Conversation",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "What to say",
            "data": {
              "items": [
                { "text": "\"Today is really just a casual chat to understand your situation.\"" },
                { "text": "\"No pressure to make any decisions.\"" },
                { "text": "\"If anything is useful, we can always explore further next time.\"" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-003",
        "type": "SECTION",
        "title": "3. Personal & Life Context",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Family Situation",
            "data": { "items": [{ "text": "\"Do you usually plan things just for yourself or also for family?\"" }] }
          },
          {
            "type": "BULLET_LIST",
            "title": "Life Stage",
            "data": { "items": [{ "text": "\"Are you more in the building phase now or already settling down?\"" }] }
          },
          {
            "type": "BULLET_LIST",
            "title": "Responsibilities",
            "data": { "items": [{ "text": "\"These days biggest commitments usually what — work, home, family?\"" }] }
          }
        ]
      },
      {
        "blockId": "blk-004",
        "type": "SECTION",
        "title": "4. Goals & Priorities",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Questions to ask",
            "data": {
              "items": [
                { "text": "\"Anything you're looking forward to in the next couple of years?\"" },
                { "text": "\"Longer term, what would you ideally want life to look like?\"" },
                { "text": "\"For you now, is it more about growing, protecting, or keeping things flexible?\"" },
                { "text": "\"Anything that worries you a bit financially these days?\"" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-005",
        "type": "SECTION",
        "title": "5. Financial & Insurance Baseline",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Questions to ask",
            "data": {
              "items": [
                { "text": "\"Have you set up anything before? Like hospital or other plans?\"" },
                { "text": "\"Are you someone who saves and invests regularly or more flexible?\"" },
                { "text": "\"Do you feel quite clear about how everything is structured currently?\"" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-006",
        "type": "SECTION",
        "title": "6. Next Steps",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Closing lines",
            "data": {
              "items": [
                { "text": "\"If you're open, next time I can help you organise everything clearly — no rush.\"" },
                { "text": "\"Maybe we catch up again next week or the week after, whichever works for you.\"" }
              ]
            }
          }
        ]
      }
    ],
    "tasks": null
  },
  "usage": {
    "model": "ge-llm-v2",
    "inputTokens": 420,
    "outputTokens": 890
  },
  "error": null
}
```

> `tasks: null` — meeting prep does not generate dashboard tasks. Tasks are created from the `FULL_SUMMARY` call after the meeting.

---

## 5. Sample — `EXISTING_WITH_MEETINGS`

**Scenario**: Agent is preparing to meet an existing customer (John) who has prior meeting summaries on record. Orchestrator fetches profile + past summaries from CRM, retrieves contextualised promos, and calls `/llm/meeting-prep`.

### Request

```json
{
  "requestId": "req-20260415-002",
  "interactionId": "int-prep-2002",
  "timestamp": "2026-04-15T09:00:00+08:00",
  "actor": "DISTRIBUTOR",
  "domain": "SALES",
  "intent": "MEETING_PREP_NOTES",
  "subIntent": "EXISTING_WITH_MEETINGS",
  "modelConfig": {
    "processingMode": "STANDARD",
    "maxOutputTokens": 2000
  },
  "prompt": {
    "templateId": "tpl-meetingprep-existing-with-v1",
    "templateVersion": "1.0.0",
    "variables": {
      "market": "SG",
      "agentName": "Nisha",
      "numCustomers": 1
    },
    "systemInstruction": null,
    "developerInstruction": null,
    "userInstruction": null
  },
  "customerContext": [
    {
      "customerId": "cust-john-002",
      "customerName": "John",
      "profile": {
        "age": 42,
        "occupation": "Tech Product Manager",
        "familyStatus": "Married, 1 daughter (17 years old)",
        "existingPolicies": ["Term life ~$500k", "CI (older plan)", "Hospital plan"]
      },
      "pastMeetingSummaries": [
        {
          "meetingId": "mtg-0998",
          "meetingDate": "2026-01-10",
          "summaryText": "Discussed John's upcoming condo upgrade consideration, daughter's university timeline (~1 year), and concern about CI coverage adequacy under his older plan. Agreed to revisit CI and retirement planning at next meeting."
        }
      ]
    }
  ],
  "agentContext": {
    "agentId": "agt-101",
    "agentName": "Nisha"
  },
  "meetingPrepConfig": {
    "promoData": [
      {
        "promoId": "promo-2026-q2-002",
        "title": "Great Eastern CI Booster",
        "description": "Top-up CI plan for existing policyholders. Applicable for customers aged 35–55 with existing CI gaps."
      },
      {
        "promoId": "promo-2026-q2-003",
        "title": "GE EduSave Plus",
        "description": "Education savings endowment. Suitable for parents with children 3–5 years from university."
      }
    ]
  },
  "outputSchema": {
    "schemaVersion": "1.0",
    "allowedBlockTypes": ["SECTION", "TEXT", "BULLET_LIST"],
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
  "requestId": "req-20260415-002",
  "interactionId": "int-prep-2002",
  "timestamp": "2026-04-15T09:00:14+08:00",
  "status": "SUCCESS",
  "intent": "MEETING_PREP_NOTES",
  "subIntent": "EXISTING_WITH_MEETINGS",
  "response": {
    "title": "Meeting Prep — John",
    "contentBlocks": [
      {
        "blockId": "blk-001",
        "type": "SECTION",
        "title": "1. Quick Reconnect",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "TEXT",
            "title": "Opening",
            "body": "> \"John, good to see you again. Last time we spoke was January — quite a bit has probably happened since then.\""
          },
          {
            "type": "BULLET_LIST",
            "title": "Warm-up prompts",
            "data": {
              "items": [
                { "text": "\"How's the family? Your daughter's exams coming up soon?\"" },
                { "text": "\"Any updates on the condo plans you mentioned last time?\"" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-002",
        "type": "SECTION",
        "title": "2. Recap from Last Meeting",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Key items discussed in January",
            "data": {
              "items": [
                { "text": "Potential condo upgrade — check if still on the cards and updated timeline." },
                { "text": "Daughter's university in ~1 year — EduSave or savings position to confirm." },
                { "text": "CI coverage concern (older plan) — agreed to revisit adequacy this meeting." }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-003",
        "type": "SECTION",
        "title": "3. Conversation Focus Areas",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Suggested discussion points",
            "data": {
              "items": [
                { "text": "CI gap: Is the older plan still sufficient given his current income and commitments?" },
                { "text": "University funding: Confirm readiness — lump sum, education plan, or CPF?" },
                { "text": "Retirement runway: At 42, how is he thinking about the next 15–20 years?" },
                { "text": "Condo upgrade impact: Would this change his liquidity position or CPF availability?" }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-004",
        "type": "SECTION",
        "title": "4. Relevant Promotions",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Products worth mentioning",
            "data": {
              "items": [
                { "text": "GE CI Booster — top-up for existing CI gaps; well-suited to John's age and existing coverage situation." },
                { "text": "GE EduSave Plus — education endowment; timing aligns with daughter's university in ~1 year (check if still relevant)." }
              ]
            }
          }
        ]
      },
      {
        "blockId": "blk-005",
        "type": "SECTION",
        "title": "5. Next Steps",
        "body": null,
        "data": null,
        "childBlocks": [
          {
            "type": "BULLET_LIST",
            "title": "Suggested close",
            "data": {
              "items": [
                { "text": "\"If the CI numbers look off, I can run a quick comparison for you before the next visit.\"" },
                { "text": "\"For the education piece — let's work out the numbers while there's still time.\"" }
              ]
            }
          }
        ]
      }
    ],
    "tasks": null
  },
  "usage": {
    "model": "ge-llm-v2",
    "inputTokens": 680,
    "outputTokens": 1100
  },
  "error": null
}
```

> `tasks: null` — meeting prep does not generate dashboard tasks. Tasks are created from the `FULL_SUMMARY` call after the meeting.

**`EXISTING_WITHOUT_MEETINGS` variant**: Request omits `pastMeetingSummaries`. Response `contentBlocks` do not include a "Recap from Last Meeting" section. The prep guide covers discovery-style questions appropriate for a customer with a profile but no recorded history.

---

## 6. Open Points

| ID | Question | Status |
|---|---|---|
| OP-24 | ~~Should `NEW_CUSTOMER` also return a `tasks` array?~~ | **Closed** — Tasks are output of `POST /llm/meeting-summary` (FULL_SUMMARY), not of meeting prep. `tasks` is always null on this endpoint. |
| OP-26 | ~~Single vs split endpoints~~ | **Closed** — confirmed split: `/llm/meeting-summary` (UC-01) and `/llm/meeting-prep` (UC-02) |
| OP-27 | **Sub-section nesting in contentBlocks**: The current SECTION block supports one level of nesting only (SECTION → leaf children). If LLM output requires a Header → SubHeader → content pattern where a single SubHeader contains more than one content block (e.g., a TEXT explanation + BULLET_LIST of questions under the same subheader), the current schema cannot represent this cleanly. Two options: (A) allow two-level SECTION nesting (SECTION children may include SECTION, with leaf-only grandchildren); (B) introduce a new `GROUP` block type as a subheader container (title + leaf childBlocks only), nestable inside SECTION but not recursive. Most likely to surface in `EXISTING_WITH_MEETINGS` / `EXISTING_WITHOUT_MEETINGS` prep output where contextualised topic sections need both a narrative and discussion bullets. Confirm with AI team whether their output naturally produces this pattern before extending the schema. | Candidate — assess against actual LLM output |

---

*Samples are based on business-provided input/output scenarios from the Sample Input Output CSV. Validate field names, data types, and prompt template IDs with the AI team before finalising.*
