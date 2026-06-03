# Team 1 — Campaign Service
## Type: Bug Fix via SDD

---

## Context

The Campaign Service is fully implemented. It manages the lifecycle of promotions — from draft through publish to pause. However, a bug has been reported in production.

**Your job:**
1. Understand the system using the campaign-service-guide skill
2. Reproduce the bug using the validation scenarios
3. Write an RCA using the rca-writer skill
4. Write a spec for the fix using the sdd-spec-writer skill
5. Implement the fix
6. Record an ADR using the adr-recorder skill
7. Validate against test data
8. Open a PR

---

## The Bug Report

**Reported by:** Analytics Team
**Severity:** High
**Description:** Campaign `camp-001` (Weekend Mega Sale) is auto-pausing before its budget is actually exhausted. The analytics dashboard shows budget burn at 94.9% but the campaign has already been paused. MX team is losing revenue because active campaigns are stopping too early.

**Validation scenario that exposes this:**
- Scenario 8: Budget hits 95% → campaign auto-pauses
- Expected: campaign pauses at exactly 95% burn
- Actual: campaign pauses at 94.9% burn

---

## Business Rules You Must Enforce (after fix)

1. A campaign must auto-pause when budget burn reaches **exactly 95% or above**
2. A campaign with no funding source cannot be published — return error `NO_FUNDING_SOURCE`
3. A campaign cannot be published if its UPCs overlap with another active campaign on the same tenant — return error `UPC_OVERLAP`
4. Funding split (vendorShare + krogerShare) must equal exactly 100%
5. A campaign's date range end must be after its start date

---

## Validation Scenarios to Pass

| Scenario | Input | Expected |
|----------|-------|----------|
| 8 | camp-001 budgetBurnPercent: 95.0 | CampaignPaused event emitted |
| 8 | camp-001 budgetBurnPercent: 94.9 | Campaign stays ACTIVE |
| 9 | camp-010 (no funding) → publish | 400 NO_FUNDING_SOURCE |
| 10 | camp-009 (UPC overlap with camp-001) → publish | 409 UPC_OVERLAP |

---

## Contract Responsibility

If your fix changes the `CampaignPaused` or `BudgetExhausted` event schema — update the `campaign-eligibility-contract` skill and notify Team 2.

---

## Skills to Use

- `rca-writer` — write the root cause analysis first
- `sdd-spec-writer` — write the fix spec
- `campaign-service-guide` — understand the codebase
- `adr-recorder` — record your decision
- `campaign-eligibility-contract` — check if your fix impacts downstream
