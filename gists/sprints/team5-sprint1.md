# Team 5 — Analytics Service — Sprint 1

> Type: Bug Fix via SDD
> Duration: 2 hours
> Skills: rca-writer, sdd-spec-writer, analytics-service-guide, adr-recorder, pr-reviewer

---

## Context

The Analytics Service is fully implemented but crashes with a 500 error on specific campaigns. Your job is to find the bug, document it properly, and fix it through the full SDD process.

The process matters as much as the fix. RCA first. Spec second. Code third.

---

## Sprint 1 Focus

**One bug. Full RCA → Spec → ADR → Fix → Validate process.**

---

## The Bug Report

**Reported by:** MX Team
**Severity:** CRITICAL

**Description:**
Analytics dashboard crashes with 500 Internal Server Error for `camp-003` (Spend $50 Get $10 Off) and `camp-004` (Platinum Wine). All other campaigns work fine.

**Error in logs:**
```
panic: runtime error: integer divide by zero
goroutine 1 [running]:
promotionos/analytics/internal/domain/service.(*LiftCalculatorImpl).Calculate(...)
```

---

## Your Tasks — In This Order

### 1. Reproduce (15 min)
Before reading code:

- `GET /analytics/campaigns/camp-003/report` → record error
- `GET /analytics/campaigns/camp-004/report` → record error
- `GET /analytics/campaigns/camp-001/report` → confirm this works

What is different about camp-003 and camp-004?
Look at their funding in test-data.md before reading any code.

### 2. Write RCA (20 min)
Use `rca-writer` skill.

Must identify:
- Exact file + method + wrong expression
- Which specific input value triggers the crash
- Why camp-003/004 trigger it but camp-001 does not

### 3. Write Fix Spec (15 min)
Use `sdd-spec-writer` skill.

Must include:
- Domain rule: ROI is null when totalFundingCost is zero
- Acceptance criteria for scenarios 18, 19, 20
- Edge cases: zero lift, negative lift, null ROI are all valid non-error states

### 4. Record ADR (10 min)
Use `adr-recorder` skill.

Cover: why ROI is null (not 0, not Infinity) for zero-cost campaigns.

### 5. Implement (30 min)
Add null guard before division. Nothing else.

### 6. Validate (15 min)
- Scenario 20: camp-003 → roi: null, 200 response ✅
- Scenario 19: camp-004 → roi: -0.40, 200 response ✅
- Scenario 18: camp-012 → roi: 0.0, 200 response ✅
- camp-001 → roi: 1.64 still works ✅

### 7. PR (15 min)
Use `pr-reviewer` skill. Include RCA + ADR links.

---

## Domain Rule

From ubiquitous-language.md:
> ROI is null when totalFundingCost is zero. This applies to 100% Kroger-funded campaigns where vendorShare is 0.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 18 | GET /analytics/camp-012/report | roi:0.0, lift:0.0, 200 |
| 19 | GET /analytics/camp-004/report | roi:-0.40, lift:-5.0%, 200 |
| 20 | GET /analytics/camp-003/report | roi:null, 200 (no 500) |
| — | GET /analytics/camp-001/report | roi:1.64, still works |
