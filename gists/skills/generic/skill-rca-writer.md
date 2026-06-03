# Skill — RCA Writer

> Type: Generic Utility
> Used by: Teams with pre-built services containing bugs (Team 1, Team 5)
> Purpose: Systematically find, document, and fix bugs through root cause analysis

---

## What This Skill Does

A Root Cause Analysis (RCA) is a structured investigation of a bug. It answers four questions:
1. What is the bug?
2. Why was it introduced?
3. What is the correct behaviour?
4. How do we fix it?

**The rule:** No bug fix without an RCA. No RCA without first reproducing the bug against test data. No fix without a Spec written after the RCA.

---

## The RCA Process

### Step 1 — Reproduce First
Before reading any code, reproduce the bug using the validation scenarios.

- Run the failing scenario
- Confirm the wrong output
- Record the exact wrong output (not what you expected — what actually happened)

### Step 2 — Isolate
Narrow down where in the code the wrong behaviour occurs.

- Which method is producing the wrong result?
- What inputs cause it to fail?
- What inputs do NOT cause it to fail? (boundaries)

### Step 3 — Root Cause
Find the exact line or logic that is wrong.

- Is it a wrong operator? (`>` vs `>=`)
- Is it a missing null check?
- Is it a wrong formula?
- Is it a missing edge case?

### Step 4 — Write RCA
Document what you found using the RCA structure below.

### Step 5 — Write Fix Spec
Use the `sdd-spec-writer` skill to write a Spec for the fix. The Spec's acceptance criteria are the failing scenarios that must now pass.

### Step 6 — Implement + Validate
Fix the code. Run all scenarios — both the ones that were failing AND the ones that were passing (regression check).

---

## The RCA Structure

```
# RCA — [Bug Title]

## Bug Report Reference
What was reported? Who reported it? When?

## Severity
CRITICAL | HIGH | MEDIUM | LOW

## Reproduction Steps
Exact steps to reproduce the bug using test data.

1. Call [endpoint] with [exact input from test-data.md]
2. Observe [exact wrong output]

Example:
1. GET /analytics/campaigns/camp-003/report
2. Response: 500 Internal Server Error
   Error: panic: runtime error: integer divide by zero

## Expected Behaviour
What should have happened according to the domain rules and Ubiquitous Language?

Reference the exact domain rule being violated:
"ROI is null when totalFundingCost is zero" (ubiquitous-language.md)

## Actual Behaviour
What actually happened? Be precise.
Copy the exact error message or wrong value.

## Root Cause

### Where
File: [file path]
Method: [method name]
Line: [line number if known]

### What
Describe the exact wrong logic in plain English.

Example:
The ROI calculation divides incrementalMargin by totalFundingCost without
checking if totalFundingCost is zero. For 100% Kroger-funded campaigns
(camp-003, camp-004) where vendorShare is 0, totalFundingCost accumulates
as 0.00, causing a divide-by-zero panic at runtime.

### Why It Was Introduced
Was it an oversight? A wrong assumption? A missing requirement?
Be honest — this is for learning, not blame.

Example:
The zero-cost case was not considered during implementation. The formula
assumes all campaigns have vendor funding. Kroger-funded campaigns were
added to test data after the initial implementation.

## Impact
Which validation scenarios fail because of this bug?
Which campaigns in test data are affected?
Are any other methods affected by the same root cause?

## Fix Description
What is the correct logic?

Example:
Add a null guard before the division:
- If totalFundingCost == 0 → return null for ROI
- Do not throw, do not return 0, do not return Infinity

## Fix Spec Reference
Link to the Spec written for this fix (written after this RCA using sdd-spec-writer skill).

## Regression Risk
What existing passing scenarios could break if the fix is wrong?
List them. Run them after fixing.

## Lessons Learned
What process, pattern, or check would have caught this earlier?

Example:
- Zero-value boundary cases should always be included in test data
- Division operations should always have null guards reviewed in PR
```

---

## How to Use This Skill with Claude

1. Run the failing validation scenario and collect the exact error
2. Say: "Use the rca-writer skill. The bug is: [describe what's happening]. The failing scenario is: [scenario number and description]"
3. Claude will guide you through the RCA process
4. Fill in the Root Cause section yourself — do not let Claude guess at the root cause without you reading the code
5. Once RCA is complete, use `sdd-spec-writer` to write the fix Spec

---

## Common RCA Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|-------------|---------------|
| "The code was wrong" | Not a root cause — what specifically was wrong and why? |
| Fixing before reproducing | You might fix the wrong thing |
| No regression check | Fix introduces new bug |
| Blaming the test data | Test data is real-world — the code must handle it |
| Skipping RCA for "obvious" bugs | Obvious bugs often have non-obvious causes |

---

## Review Checklist

- [ ] Bug reproduced exactly using named test data scenario
- [ ] Root cause identifies the exact file, method, and wrong logic
- [ ] Why it was introduced is answered honestly
- [ ] Impact lists all affected scenarios
- [ ] Fix description states the correct logic precisely
- [ ] Regression scenarios identified and listed
- [ ] Fix Spec written before implementation begins
