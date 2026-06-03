# Team 5 — Analytics Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, analytics-service-guide, adr-recorder, pr-reviewer, redemption-analytics-contract, analytics-campaign-contract

---

## Context

Sprint 1 fixed the divide-by-zero bug. Analytics Service now correctly handles zero-cost campaigns returning null ROI. Sprint 2 adds real-time budget burn tracking and the BudgetExhausted event that triggers campaign auto-pause.

---

## Sprint 2 Goal

By end of sprint — budget burn is updated in real time on every OfferRedeemed event, and BudgetExhausted is published exactly once when a campaign hits 95.0% burn.

---

## Requirements

### Requirement 1 — Real-Time Budget Burn Tracking

Implement `BudgetBurnTracker.update()`:

1. On every `OfferRedeemed` event — increment `burnedAmount` by `discountApplied`
2. Recalculate `budgetBurnPercent = (burnedAmount / totalAmount) × 100`
3. Round to 2 decimal places
4. Persist updated metrics immediately — do not batch
5. Update `redemptionCount` by 1 on every OfferRedeemed

### Requirement 2 — BudgetExhausted Event

Implement `BudgetBurnTracker.checkThreshold()`:

1. After every burn update — check if `budgetBurnPercent >= 95.0`
2. If threshold reached AND `budgetExhaustedEmitted` flag is false:
   - Publish `BudgetExhausted` event
   - Set `budgetExhaustedEmitted = true` on campaign metrics record
3. If threshold reached AND `budgetExhaustedEmitted` is already true → do NOT republish
4. Event must match exact schema in `contract-analytics-campaign.md`
5. `schemaVersion: 1`

### Requirement 3 — Campaign Metrics Initialisation

On `CampaignPublished` event:

1. Create a campaign metrics record with:
   - `campaignId`, `tenantId`
   - `burnedAmount: 0`
   - `redemptionCount: 0`
   - `budgetBurnPercent: 0`
   - `budgetExhaustedEmitted: false`
   - `baselineSalesPerDay` from test data benchmarks
2. If metrics record already exists — do not overwrite (idempotent)

### Requirement 4 — GET /analytics/campaigns/:id/burn

Implement burn endpoint:

1. Return current burn state: totalAmount, burnedAmount, budgetBurnPercent, redemptionCount
2. Return `budgetExhaustedEmitted` flag so Frontend knows if alert should show
3. Used by Frontend for real-time burn chart

---

## Domain Rules

From ubiquitous-language.md:
- Budget burn threshold is >= 95.0% (not > 94%)
- BudgetExhausted is emitted exactly once per Campaign
- burnedAmount accumulates from discountApplied across all redemptions
- budgetBurnPercent = (burnedAmount / totalAmount) × 100, rounded to 2 decimal places

---

## Contracts Involved

`contract-redemption-analytics.md` — OfferRedeemed event (consumed)
`contract-analytics-campaign.md` — BudgetExhausted event (published)

The BudgetExhausted event triggers Campaign Service to pause the campaign.
Any deviation from the contract schema will break the auto-pause flow.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 8 | camp-001 burn reaches 95.0% | BudgetExhausted published |
| 26 | camp-012 burn at exactly 95.0% | BudgetExhausted published |
| 27 | camp-001 subsequent redemption after exhaustion | BudgetExhausted NOT re-published |
| — | camp-002 burn at 40.0% | BudgetExhausted NOT published |
| — | GET /analytics/camp-001/burn | burnedAmount:47600, budgetBurnPercent:95.2 |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Exactly-once BudgetExhausted semantics — how the flag approach guarantees single emission
- Whether to use optimistic or pessimistic locking for concurrent burn updates
