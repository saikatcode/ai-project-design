# Test Data: NBE Engagement Plan Scenarios

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Derived From** | REQ-NBE-001, REQ-NBE-004, API-nbe-orchestrator-endpoints |
| **Last Updated** | 2026-04-09 |
| **Author** | Tech Architect |

---

## Test Scenario 1: New Parent with Coverage Gaps

**Description:** Customer recently had a child, has basic life coverage but no CI or education plan.

**Request Payload:**
```json
{
  "agent_id": "A-TEST-001",
  "customer_id": "C-TEST-NBE-001",
  "trigger": "profile_open",
  "preferences": {
    "max_recommendations": 3,
    "include_competitor_comparison": true,
    "language": "en"
  }
}
```

**Mock Customer Data:**
```json
{
  "profile": {
    "customer_id": "C-TEST-NBE-001",
    "name": "Test Customer NBE-1",
    "age": 35,
    "occupation": "Software Engineer",
    "annual_income": 120000,
    "dependents": [
      {"relation": "spouse", "age": 33},
      {"relation": "child", "age": 0, "dob": "2026-02-15"}
    ]
  },
  "portfolio": [
    {"policy_id": "P-20001", "product": "Basic Life 200", "sum_assured": 200000, "premium_monthly": 150}
  ],
  "interactions": [
    {"date": "2026-01-10", "type": "annual_review", "summary": "Discussed retirement planning, deferred action"},
    {"date": "2025-09-05", "type": "claim_inquiry", "summary": "Asked about hospitalization claim process"}
  ],
  "claims": [],
  "life_events": [
    {"event": "child_birth", "date": "2026-02-15", "details": "First child born"}
  ]
}
```

**Expected Recommendations:**
1. CI coverage (urgency: new dependent, no existing CI)
2. Education savings plan (urgency: newborn, compounding benefit of early start)
3. Increase life coverage (current $200k insufficient for family of 3 at income level)

---

## Test Scenario 2: Pre-Retirement Customer

**Description:** Customer approaching retirement, has multiple policies, recent hospitalization claim.

**Mock Customer Data:**
```json
{
  "profile": {
    "customer_id": "C-TEST-NBE-002",
    "age": 58,
    "occupation": "Retired (pending)",
    "dependents": [{"relation": "spouse", "age": 56}]
  },
  "portfolio": [
    {"policy_id": "P-30001", "product": "Whole Life 500", "sum_assured": 500000},
    {"policy_id": "P-30002", "product": "Medical Shield Gold", "status": "active"},
    {"policy_id": "P-30003", "product": "ILP Growth Fund", "fund_value": 180000}
  ],
  "claims": [
    {"claim_id": "CL-5001", "type": "hospitalization", "date": "2026-03-01", "amount": 15000, "status": "settled"}
  ],
  "life_events": [
    {"event": "retirement_approaching", "estimated_date": "2027-01-01"}
  ]
}
```

**Expected Behavior:**
- Recommendations focus on retirement readiness and medical coverage adequacy
- Recent hospitalization claim should influence medical coverage recommendations
- ILP fund rebalancing discussion (risk appetite shift pre-retirement)
- No hard-sell on new products — focus on optimizing existing portfolio

---

## Test Scenario 3: Partial Data — Claims API Unavailable

**Setup:** Mock claims API returns 504 timeout.

**Expected Response:**
- Engagement plan still generated with available data
- `data_gaps` array includes: `{"source": "CLAIMS", "status": "unavailable", "message": "..."}`
- Recommendations that would require claims data are flagged with lower confidence

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial 3 test scenarios | Tech Architect |
