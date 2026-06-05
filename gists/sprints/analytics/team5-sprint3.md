# Team 5 — Analytics Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, analytics-service-guide, adr-recorder, pr-reviewer, redemption-analytics-contract, analytics-campaign-contract

---

## Context

Sprint 1 fixed the divide-by-zero bug. Sprint 2 added real-time burn tracking and BudgetExhausted events. Sprint 3 adds lift calculation, ROI computation, and handling the new BudgetUpdated event from Campaign Service when a campaign is reactivated via budget top-up.

---

## Sprint 3 Goal

By end of sprint — lift and ROI are calculated correctly for all campaign types including zero-cost, negative lift, and zero lift scenarios. BudgetUpdated event resets the exhaustion flag for reactivated campaigns.

---

## Requirements

### Requirement 1 — Lift Calculation

Implement `LiftCalculator.calculate()`:

1. `lift = actualSalesPerDay - baselineSalesPerDay`
2. `liftPercentage = (lift / baselineSalesPerDay) × 100`
3. If `baselineSalesPerDay = 0` → liftPercentage is null (not divide-by-zero)
4. Negative lift is valid — do not clamp to zero
5. Zero lift (0.0) is valid — do not treat as null
6. `actualSalesPerDay` is updated from OfferRedeemed events (rolling average)

### Requirement 2 — ROI Calculation

Implement `LiftCalculator.calculateROI()`:

1. `incrementalMargin = lift × 0.30` (30% average margin — fixed for session)
2. `roi = incrementalMargin / totalFundingCost`
3. If `totalFundingCost = 0` → roi is null (100% Kroger-funded campaigns)
4. Negative ROI is valid
5. Zero ROI (0.0) is valid — zero lift means zero incremental margin means zero ROI

### Requirement 3 — BudgetUpdated Event Consumption

Handle new `BudgetUpdated` event from Campaign Service (Team 1 Sprint 3):

1. On `BudgetUpdated` with `reactivated: true`:
   - Update `totalAmount` on campaign metrics
   - Recalculate `budgetBurnPercent` with new totalAmount
   - Reset `budgetExhaustedEmitted = false`
   - This allows BudgetExhausted to be emitted again if burn hits 95% again
2. On `BudgetUpdated` with `reactivated: false`:
   - Update `totalAmount` only
   - Do not reset `budgetExhaustedEmitted`

### Requirement 4 — Full Analytics Report

Implement `GET /analytics/campaigns/:id/report`:

1. Return complete campaign analytics:
   - lift, liftPercentage, roi (all nullable)
   - budgetBurnPercent, burnedAmount, totalAmount
   - redemptionCount, totalFundingCost
   - budgetExhaustedEmitted flag
2. All scenarios 18, 19, 20 must return 200 (no 500 errors)

---

## Domain Rules

From ubiquitous-language.md:
- ROI is null when totalFundingCost is zero
- liftPercentage is null when baselineSalesPerDay is zero
- Negative and zero values are valid — never clamp

---

## Contracts Involved

`contract-redemption-analytics.md` — OfferRedeemed, ClaimSubmitted (consumed)
`contract-analytics-campaign.md` — BudgetExhausted (published)
New: `BudgetUpdated` event from Campaign Service — consume and handle

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 18 | GET /analytics/camp-012/report | lift:0.0, liftPercentage:0.0, roi:0.0 |
| 19 | GET /analytics/camp-004/report | lift:-20.0, liftPercentage:-5.0, roi:-0.40 |
| 20 | GET /analytics/camp-003/report | roi:null, liftPercentage:24.0, 200 response |
| — | BudgetUpdated reactivated:true for camp-006 | budgetExhaustedEmitted reset to false |
| — | GET /analytics/camp-001/report | lift:780.0, liftPercentage:65.0, roi:1.64 |

**Regression check — must still pass after Sprint 3 LiftCalculator rebuild:**
- Scenario 18: camp-012 → roi:0.0 (zero lift, zero ROI — not null)
- Scenario 19: camp-004 → roi:-0.40 (negative — not clamped to zero)
- Scenario 20: camp-003 → roi:null (zero-cost — not 500)

These three were fixed in Sprint 1. If Sprint 3 reimplementation breaks them — stop and fix before merging.

---

## Sprint 3 ADR Topics

Record an ADR for:
- Why ROI is null (not 0 and not Infinity) for zero-cost campaigns
- BudgetUpdated handling — why reactivated:false does not reset exhaustion flag
