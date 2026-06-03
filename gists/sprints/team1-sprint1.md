# Team 1 — Campaign Service — Sprint 1

> Type: Bug Fix via SDD
> Duration: 2 hours
> Skills: rca-writer, sdd-spec-writer, campaign-service-guide, adr-recorder, pr-reviewer

---

## Context

The Campaign Service is fully implemented. One bug has been reported — campaigns are auto-pausing earlier than they should. Your job this sprint is to find it, document it, and fix it through the full SDD process.

The SDD process is the point of this sprint. Not the fix. Anyone can change `>` to `>=`. The discipline of RCA → Spec → ADR → Fix → Validate is what we are here to learn.

---

## Sprint 1 Focus

**One bug. Full process. No shortcuts.**

---

## The Bug Report

**Reported by:** Analytics Team
**Severity:** HIGH

**Description:**
Campaign `camp-001` (Weekend Mega Sale) auto-paused before its budget was exhausted. Analytics shows burn at 94.9% at time of pause. The campaign should have continued until 95.0%.

---

## Your Tasks — In This Order

### 1. Reproduce (15 min)
Before touching code — reproduce using test data.

Run scenario 8:
- camp-001: totalAmount $50,000, burnedAmount $47,500 (95.0%)
- Expected: CampaignPaused emitted
- Also test: burnedAmount $47,499 (94.998%) → should NOT pause

Record the exact wrong output. Do not guess.

### 2. Write RCA (20 min)
Use `rca-writer` skill.

Must answer:
- Exact file + method + line of wrong logic
- The wrong operator/expression
- Why it fails at the boundary
- What the correct logic is

### 3. Write Fix Spec (15 min)
Use `sdd-spec-writer` skill.

Must include:
- Domain rule: `budgetBurnPercent >= 95.0` triggers pause
- Acceptance criteria mapped to scenarios 8 and 26
- Edge case: exactly 95.0% must pause, 94.9% must not

### 4. Record ADR (10 min)
Use `adr-recorder` skill.

Cover: why `>= 95.0` not `> 94` or `> 95`.

### 5. Implement (30 min)
Fix exactly the line identified in RCA. Nothing else.

### 6. Validate (15 min)
- Scenario 8: camp-001 at 95.0% → CampaignPaused ✅
- Scenario 26: camp-012 at exactly 95.0% → CampaignPaused ✅
- camp-001 at 94.9% → stays ACTIVE ✅
- Scenario 27: BudgetExhausted fires exactly once ✅

### 7. Expose Campaign Summary Endpoint (10 min)
The pre-built service already has campaign data. Expose one read endpoint that Team 3 needs in Sprint 2:

`GET /campaigns/:id/summary` — return id, name, status, fundingVendorShare, fundingVendorId, budgetTotal, budgetBurned

This unblocks Team 3's claim deduction calculation. No business logic — pure data read from the existing campaign record.

### 8. PR (5 min)
Use `pr-reviewer` skill. PR description includes RCA + ADR links + summary endpoint added.

---

## Domain Rule

From ubiquitous-language.md:
> Budget exhaustion threshold is >= 95.0% — campaigns pause at exactly 95.0%, not before.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 8 | camp-001 burn: 95.0% | CampaignPaused emitted |
| 26 | camp-012 burn: exactly 95.0% | CampaignPaused emitted |
| 27 | camp-001 burn: 96% after pause | BudgetExhausted NOT re-emitted |
| — | camp-001 burn: 94.9% | Campaign stays ACTIVE |
