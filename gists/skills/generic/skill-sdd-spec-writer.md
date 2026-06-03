# Skill — SDD Spec Writer

> Type: Generic Utility
> Used by: Every team, every sprint
> Purpose: Transform requirements into a precise, implementation-ready spec

---

## What This Skill Does

This skill guides you through writing a Spec from a set of requirements before writing any code. A Spec is not a design doc. It is not pseudocode. It is a precise, verifiable contract between the requirement and the implementation.

**The rule:** No implementation without a Spec. No Spec without requirements. No merge without validation passing against the Spec's acceptance criteria.

---

## When to Use

Use this skill immediately after reading your sprint requirements and before opening your IDE.

---

## The Spec Structure

Every Spec you write must follow this exact structure:

---

```
# Spec — [Feature Name]

## 1. Context
Which bounded context does this belong to?
Which sprint is this?
Which team is writing this?

## 2. Requirements Reference
Link to or quote the exact requirements this Spec addresses.
Do not paraphrase. Copy the requirement exactly.

## 3. Ubiquitous Language
List every domain term from the Ubiquitous Language doc that this Spec uses.
If you use a term not in the Ubiquitous Language — stop. Add it or use the correct term.

## 4. What This Feature Does
One paragraph. Plain English. No technical jargon.
A non-engineer should be able to read this and understand what the feature does.

## 5. Domain Rules
List every business rule this feature must enforce.
Number them. Be precise. Use exact values from test data where applicable.

Example:
1. A Campaign cannot be published without a confirmed Funding source
2. vendorShare + krogerShare must equal exactly 100%
3. Budget exhaustion threshold is >= 95.0% (not > 94%)

## 6. Acceptance Criteria
Map directly to validation scenarios from test-data.md.
Each criterion must be binary — it either passes or it does not.

- [ ] Scenario 1: cust-002 + camp-002 → eligible: true, discount: 4.99
- [ ] Scenario 9: camp-010 publish → 400 NO_FUNDING_SOURCE
- [ ] Scenario 26: camp-012 at 95.0% burn → CampaignPaused emitted

## 7. Edge Cases
What must NOT happen. Failure modes. Boundary conditions.
For every edge case — state what the correct behaviour is.

Example:
- Budget at 94.9% → Campaign must NOT pause
- Budget at 95.0% → Campaign MUST pause
- BudgetExhausted must be emitted exactly once — not on every subsequent redemption

## 8. Contract Impact
Does this feature publish or consume any events or APIs defined in the contracts?
If yes — list which contracts are involved.
If you are changing a contract — STOP. Update the contract skill and notify the consuming team first.

## 9. Out of Scope
What are you explicitly NOT building in this sprint?
This prevents scope creep and keeps the team focused.

## 10. ADR Reference
Link to the ADR once it is written (after Spec review, before implementation).
```

---

## How to Use This Skill with Claude

1. Paste your sprint requirements into Claude
2. Say: "Use the sdd-spec-writer skill to write a Spec for these requirements"
3. Claude will produce a draft Spec following the structure above
4. Review every section — especially Domain Rules and Acceptance Criteria
5. Correct anything that does not match the Ubiquitous Language
6. Get a cross-team review on Section 8 (Contract Impact) before proceeding

---

## Common Mistakes to Avoid

| Mistake | Why it matters |
|---------|---------------|
| Writing implementation details in the Spec | Spec defines WHAT, not HOW |
| Using synonyms instead of Ubiquitous Language terms | Creates ambiguity in code |
| Skipping edge cases | Edge cases are where bugs live |
| Acceptance criteria not mapped to test data | Cannot validate without real data |
| No contract impact assessment | Breaks downstream services |
| Writing Spec after starting code | Defeats the purpose of SDD |

---

## Review Checklist

Before moving to ADR — verify your Spec passes this checklist:

- [ ] Every domain term matches the Ubiquitous Language doc exactly
- [ ] Every acceptance criterion maps to a named validation scenario
- [ ] Every edge case has a defined expected behaviour
- [ ] Contract impact section is complete
- [ ] A non-engineer can read Section 4 and understand the feature
- [ ] Out of scope is clearly defined
