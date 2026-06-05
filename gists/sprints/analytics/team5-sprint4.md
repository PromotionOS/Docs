# Team 5 — Analytics Service — Sprint 4

> Type: Full Validation + Integration
> Duration: 2 hours
> Skills: sdd-spec-writer, analytics-service-guide, pr-reviewer

---

## Context

Sprints 1-3 fixed the divide-by-zero bug, added burn tracking, BudgetExhausted events, and lift/ROI calculation. Sprint 4 is integration and validation — all analytics scenarios pass with live event data flowing from Redemption Service.

---

## Sprint 4 Goal

All analytics scenarios pass. BudgetExhausted events flow correctly to Campaign Service triggering auto-pause. Service is demo-ready.

---

## Requirements

### Requirement 1 — Live Event Integration

Confirm live events are flowing:

1. Make a redemption via Redemption Service → confirm OfferRedeemed arrives
2. Confirm burnedAmount increments correctly after each redemption
3. Trigger budget exhaustion on a test campaign → confirm BudgetExhausted fires once
4. Confirm Campaign Service receives BudgetExhausted and pauses the campaign

### Requirement 2 — Full Scenario Pass

| Scenario | Expected |
|----------|----------|
| 8 | camp-001 at 95.0% → BudgetExhausted published |
| 18 | camp-012 lift:0.0, roi:0.0 |
| 19 | camp-004 lift:-20.0, roi:-0.40 |
| 20 | camp-003 roi:null, 200 response |
| 26 | camp-012 at exactly 95.0% boundary |
| 27 | BudgetExhausted fires exactly once |

### Requirement 3 — Hardening

1. All endpoints validate tenantId
2. Unknown campaignId returns 404 — never 500
3. OfferRedeemed for unknown campaign → log warning, do not crash
4. Concurrent OfferRedeemed events for same campaign — burn update must be atomic

### Requirement 4 — Demo Data Confirmation

Pre-load all analytics benchmarks from test-data.md:

1. camp-001: baselineSalesPerDay:1200, burnedAmount:47600, budgetBurnPercent:95.2
2. camp-002: baselineSalesPerDay:800, burnedAmount:12000, budgetBurnPercent:40.0
3. camp-003: baselineSalesPerDay:5000, burnedAmount:23000, roi:null
4. Confirm these are queryable via GET /analytics/campaigns/:id/report

---

## Sprint 4 ADR Topics

Record an ADR for:
- Atomic burn update strategy — how concurrent redemptions are handled safely
