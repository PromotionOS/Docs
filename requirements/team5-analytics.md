# Team 5 — Analytics Service
## Type: Bug Fix via SDD

---

## Context

The Analytics Service is fully implemented. It tracks campaign performance in real time — lift, ROI, budget burn — by consuming domain events from the event bus. However, a bug has been reported that causes a service crash on specific campaigns.

**Your job:**
1. Understand the system using the analytics-service-guide skill
2. Reproduce the bug using the validation scenarios
3. Write an RCA using the rca-writer skill
4. Write a spec for the fix using the sdd-spec-writer skill
5. Implement the fix
6. Record an ADR using the adr-recorder skill
7. Validate against analytics benchmarks
8. Open a PR

---

## The Bug Report

**Reported by:** MX Team
**Severity:** Critical
**Description:** The analytics dashboard crashes with a 500 error when viewing performance for campaign `camp-003` (Spend $50 Get $10 Off). This campaign is 100% Kroger-funded — the vendor share is 0%. All other campaigns display correctly.

**Error observed:**
```
panic: runtime error: integer divide by zero
goroutine 1 [running]:
promotionos/analytics/service.(*LiftCalculatorImpl).Calculate(...)
    /internal/domain/service/lift_calculator.go:47
```

**Validation scenario that exposes this:**
- GET /analytics/campaigns/camp-003/report
- Expected: ROI calculated correctly for 100% Kroger-funded campaign
- Actual: 500 panic — divide by zero

---

## Business Rules You Must Enforce (after fix)

### Lift Calculation
1. `lift = actualSalesPerDay - baselineSalesPerDay`
2. `liftPercentage = (lift / baselineSalesPerDay) × 100`
3. If `baselineSalesPerDay` is zero — lift percentage is `null`, not a division error
4. Lift is recalculated every time an `OfferRedeemed` event is consumed

### ROI Calculation
5. `roi = incrementalMargin / totalFundingCost`
6. `incrementalMargin = lift × averageMarginPercent` (use 30% as default margin)
7. **If `totalFundingCost` is zero (100% Kroger-funded) — ROI is `null`, not a division error**
8. ROI is updated every time an `OfferRedeemed` event is consumed

### Budget Burn Tracking
9. Budget burn is updated in real time on every `OfferRedeemed` event
10. `budgetBurnPercent = (burnedAmount / totalAmount) × 100`
11. When `budgetBurnPercent >= 95.0` → publish `BudgetExhausted` event
12. `BudgetExhausted` must be published **exactly once** per campaign — not on every subsequent redemption

---

## Validation Scenarios to Pass

| Scenario | Input | Expected |
|----------|-------|----------|
| — | GET /analytics/campaigns/camp-003/report | 200, roi: null (not 500) |
| — | GET /analytics/campaigns/camp-001/report | roi: 1.64, budgetBurnPercent: 95.2 |
| — | GET /analytics/campaigns/camp-002/report | roi: 2.50, liftPercentage: 37.5 |
| 8 | camp-001 burn reaches 95% | BudgetExhausted published exactly once |
| — | camp-001 burn reaches 97% (subsequent redemptions) | BudgetExhausted NOT published again |

---

## Analytics Benchmarks to Validate Against

From test-data.md:

| Campaign | Expected Lift% | Expected ROI | Expected Burn% |
|----------|---------------|-------------|----------------|
| camp-001 | 65.0% | 1.64 | 95.2% |
| camp-002 | 37.5% | 2.50 | 40.0% |
| camp-003 | — | null | 23.0% |
| camp-005 | 64.0% | 2.24 | 95.3% |

---

## Events You Must Consume

- `OfferRedeemed` → update burn, recalculate lift and ROI
- `CampaignPublished` → initialise campaign metrics record
- `BudgetExhausted` (self-published guard) → do not republish

## Events You Must Publish

**BudgetExhausted**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "string",
  "totalBurned": "number",
  "occurredAt": "timestamp",
  "schemaVersion": 1
}
```

---

## Skills to Use

- `rca-writer` — write root cause analysis first
- `sdd-spec-writer` — write the fix spec
- `analytics-service-guide` — understand the codebase
- `adr-recorder` — record your null-guard decision
- `redemption-analytics-contract` — understand the OfferRedeemed event shape
- `analytics-campaign-contract` — understand the BudgetExhausted event you publish
