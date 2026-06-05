# Team 2 — Eligibility Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, eligibility-service-guide, adr-recorder, pr-reviewer, campaign-eligibility-contract, catalog-eligibility-contract

---

## Context

Sprint 1 built core eligibility. Sprint 2 added stack validation and full ACL. Sprint 3 adds the eligibility audit log — a record of every eligibility decision made — and handles the campaign reactivation event from Campaign Service (BudgetUpdated with reactivated:true).

---

## Sprint 3 Goal

By end of sprint — every eligibility decision is recorded for audit, campaign reactivation correctly reloads rules, and the full eligibility check handles all 30 validation scenarios.

---

## Requirements

### Requirement 1 — Eligibility Audit Log

Implement eligibility decision recording:

1. After every `POST /eligibility/check` — record the decision:
   - customerId, campaignId, tenantId
   - eligible: boolean
   - reason (if ineligible)
   - evaluatedAt timestamp
   - cartTotal, cartUPCs provided
2. Store in `eligibility_audit_log` table
3. Do NOT return audit log in the check response — store asynchronously
4. Audit log is append-only — never update or delete records

### Requirement 2 — Campaign Reactivation Handling

Handle `CampaignPublished` for reactivated campaigns:

1. When CampaignPublished arrives for a campaign that already has rules loaded:
   - Replace existing rules with the new event's rules (idempotent reload)
   - Do not error — treat as a rule refresh
2. This handles the budget top-up reactivation flow from Team 1's Sprint 3

### Requirement 3 — GET /eligibility/audit

Implement audit log endpoint:

1. `GET /eligibility/audit?campaignId=X&tenantId=Y&from=date&to=date`
2. Return paginated list of eligibility decisions
3. Useful for MX team to understand why customers are qualifying or not qualifying

### Requirement 4 — Full Scenario Validation

Run and confirm ALL Sprint 1 + Sprint 2 scenarios still pass after Sprint 3 changes:

| All passing scenarios | Count |
|----------------------|-------|
| Scenarios 1-6 | Basic eligibility |
| Scenarios 11-16 | Status filter, geo, stack |
| Scenarios 30 | Multi-segment |

---

## Domain Rules

From ubiquitous-language.md:
- Eligibility audit log is append-only
- CampaignPublished for an existing campaign = rule reload (idempotent)
- Every eligibility decision must be recorded regardless of outcome

---

## Contracts Involved

`contract-campaign-eligibility.md` — CampaignPublished (now also handles reactivation)
No schema changes — existing handling extended to cover reload case.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| All Sprint 1+2 | — | All still passing |
| — | POST /eligibility/check for cust-001+camp-001 | decision recorded in audit log |
| — | GET /eligibility/audit?campaignId=camp-001 | returns decisions with eligible:true for cust-001 |
| — | CampaignPublished for already-loaded campaign | rules reloaded, no error |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Audit log write approach — synchronous vs asynchronous
- Idempotent rule reload strategy for reactivated campaigns
