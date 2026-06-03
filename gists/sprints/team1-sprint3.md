# Team 1 — Campaign Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, campaign-service-guide, adr-recorder, pr-reviewer, campaign-eligibility-contract

---

## Context

Sprint 1 fixed the budget threshold bug. Sprint 2 added conflict resolution and campaign scheduling. Sprint 3 adds the funding validation API and campaign budget update capability — enabling MX teams to top up a campaign's budget without republishing.

---

## Sprint 3 Goal

By end of sprint — funding can be validated independently before campaign publish, and campaign budgets can be topped up on running campaigns without disrupting active offers.

---

## Requirements

### Requirement 1 — Funding Validation Endpoint

Implement `GET /campaigns/:id/validate-funding`:

1. Validate funding without publishing:
   - Funding source exists
   - vendorShare + krogerShare = 100%
   - vendorId is not empty
2. Return validation result: `{ valid: boolean, errors: [string] }`
3. Used by Frontend to show validation feedback before MX clicks Publish

### Requirement 2 — Budget Top-Up

Implement `PUT /campaigns/:id/budget`:

1. Allow MX team to increase a campaign's totalAmount while it is ACTIVE or PAUSED
2. Cannot decrease totalAmount below burnedAmount
3. If campaign was PAUSED due to BUDGET_EXHAUSTED and new totalAmount raises burn below 95% — reactivate campaign:
   - Set status back to ACTIVE
   - Emit `CampaignPublished` event (same event — Eligibility reloads rules)
   - Reset `budgetExhaustedEmitted` flag via `BudgetUpdated` event to Analytics
4. Publish `BudgetUpdated` event after any budget change

### Requirement 3 — BudgetUpdated Event

Define and publish new `BudgetUpdated` domain event:

```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "previousTotalAmount": "number",
    "newTotalAmount": "number",
    "burnedAmount": "number",
    "newBudgetBurnPercent": "number",
    "reactivated": "boolean"
  }
}
```

**Important:** This is a NEW event. Update `contract-analytics-campaign.md` skill with this event schema. Notify Team 5 — they need to consume this event to reset their `budgetExhaustedEmitted` flag when `reactivated: true`.

### Requirement 4 — Campaign Summary Endpoint

Implement `GET /campaigns/:id/summary`:

1. Return full campaign detail including current budget state
2. Used by Redemption Service to get vendorShare for claim calculation
3. Response includes: id, name, status, offer, funding (with vendorShare), budget (with burnedAmount)

---

## Domain Rules

From ubiquitous-language.md:
- Budget totalAmount cannot be decreased below burnedAmount
- A PAUSED campaign can be reactivated by budget top-up if burn drops below 95%
- BudgetUpdated is a new event — consuming teams must be notified

---

## Contracts Involved

**New:** `BudgetUpdated` event — notify Team 5 via contract skill update
`contract-campaign-eligibility.md` — CampaignPublished still used for reactivation

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| — | GET /campaigns/camp-010/validate-funding | valid:false, errors:[NO_FUNDING_SOURCE] |
| — | GET /campaigns/camp-001/validate-funding | valid:true |
| — | PUT /campaigns/camp-006/budget {totalAmount:15000} | camp-006 reactivated, CampaignPublished emitted |
| — | PUT /campaigns/camp-001/budget {totalAmount:30000} (below burnedAmount) | 400 BUDGET_BELOW_BURNED |
| — | GET /campaigns/camp-001/summary | includes vendorShare:60 |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Campaign reactivation trigger — threshold recalculation on budget top-up
- BudgetUpdated event design decision — why a new event vs reusing BudgetExhausted
