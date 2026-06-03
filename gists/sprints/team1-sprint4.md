# Team 1 — Campaign Service — Sprint 4

> Type: Build via SDD + Full Validation
> Duration: 2 hours
> Skills: sdd-spec-writer, campaign-service-guide, adr-recorder, pr-reviewer

---

## Context

Sprints 1-3 fixed the budget bug, added conflict resolution, scheduling, and budget top-up. Sprint 4 is the final integration sprint — validate all scenarios, harden the service, and ensure the Campaign Service is demo-ready.

---

## Sprint 4 Goal

All Campaign Service scenarios pass. Service is stable, deployed, and ready for the demo.

---

## Requirements

### Requirement 1 — Campaign List Endpoint

Implement `GET /campaigns?tenantId=X&status=Y`:

1. Return all campaigns for a tenant
2. Optional `status` filter: ACTIVE, PAUSED, SCHEDULED, DRAFT, EXPIRED
3. Default: return all statuses
4. Sorted by: ACTIVE first, then SCHEDULED, then PAUSED, then DRAFT, then EXPIRED
5. Each campaign includes: id, name, status, offerType, budgetBurnPercent, dateRange

### Requirement 2 — Hardening

Add defensive checks across all endpoints:

1. All endpoints must return 400 if `tenantId` is missing
2. All endpoints must return 403 if `tenantId` does not match the resource's tenantId
3. All date inputs must be validated — endDate must be after startDate
4. Budget totalAmount must be > 0

### Requirement 3 — Full Scenario Validation Pass

Run and confirm ALL Campaign Service scenarios:

| Scenario | Expected |
|----------|----------|
| 8 | Budget at 95% → CampaignPaused |
| 9 | No funding → 400 NO_FUNDING_SOURCE |
| 10 | UPC overlap → 409 UPC_OVERLAP |
| 26 | Budget at exactly 95.0% boundary |
| 27 | BudgetExhausted fires exactly once |
| — | SCHEDULED → ACTIVE on startDate |
| — | ACTIVE → EXPIRED on endDate |
| — | Budget top-up reactivates PAUSED campaign |

---

## Sprint 4 ADR Topics

Record an ADR for:
- Campaign list default sort order — business justification for ACTIVE first
