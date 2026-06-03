# Skill — ADR Recorder

> Type: Generic Utility
> Used by: Every team, every sprint
> Purpose: Record every significant domain or architectural decision before merging

---

## What This Skill Does

An Architectural Decision Record (ADR) captures a significant decision made during development — what was decided, why, what alternatives were considered, and what the consequences are. ADRs are the institutional memory of the system. They explain WHY the code is written the way it is.

**The rule:** Every significant domain decision must have an ADR. No merge without one.

---

## When to Write an ADR

Write an ADR whenever you make a decision that:
- Affects a domain rule or business logic
- Changes how bounded contexts communicate
- Chooses between two or more valid technical approaches
- Introduces a constraint that future engineers need to understand
- Fixes a bug in a non-obvious way

**You do NOT need an ADR for:**
- Trivial implementation details (variable names, formatting)
- Framework configuration
- Decisions fully dictated by the requirements

---

## The ADR Structure

```
# ADR-[number] — [Decision Title]

## Status
PROPOSED | ACCEPTED | SUPERSEDED | DEPRECATED

## Date
YYYY-MM-DD

## Context
What is the problem or situation that requires a decision?
What constraints exist?
What is the pressure that forces us to decide now?

Write 2-5 sentences. Be specific. Reference the requirement or scenario that triggered this decision.

## Decision
What did we decide?

State the decision clearly and directly. One paragraph maximum.
Use Ubiquitous Language terms exactly.

## Domain Rules Applied
Which domain rules from the Ubiquitous Language or requirements drove this decision?
List them explicitly.

Example:
- Budget exhaustion threshold is >= 95.0% (from ubiquitous-language.md)
- BudgetExhausted must be emitted exactly once per Campaign
- Idempotency is enforced per Tenant — same key from different Tenants is not a duplicate

## Consequences

### Positive
What does this decision enable?
What problems does it solve?

### Negative
What does this decision constrain?
What technical debt does it introduce?
What future work does it make harder?

### Neutral
What changes as a result of this decision that is neither good nor bad?

## Alternatives Considered
List every alternative you evaluated seriously.
For each — why was it rejected?

Format:
- **Alternative A:** [description] — Rejected because [reason]
- **Alternative B:** [description] — Rejected because [reason]

## Validation
Which test data scenarios validate that this decision is correctly implemented?

- Scenario N: [description] → [expected outcome]

## Related
- Links to related ADRs
- Links to contracts affected
- Links to specs this ADR supports
```

---

## ADR Numbering

ADRs are numbered sequentially per service. Team 1 starts at ADR-001, Team 2 starts at ADR-001, etc. They are stored in the team's sprint gist comments.

Format: `ADR-[team number]-[sequence]`
Example: `ADR-2-001` = Team 2's first ADR

---

## How to Use This Skill with Claude

1. After writing your Spec and getting cross-team review
2. Say: "Use the adr-recorder skill to write an ADR for this decision: [describe the decision]"
3. Claude will produce a draft ADR
4. Fill in the Context section with your specific situation
5. Review Consequences carefully — this is where most teams cut corners
6. Record the ADR before writing any implementation code

---

## Example ADR (Reference)

```
# ADR-1-001 — Budget Threshold Uses >= Not >

## Status
ACCEPTED

## Date
2026-06-10

## Context
The Campaign Service must auto-pause a Campaign when its budget burn reaches
the exhaustion threshold. The threshold is defined as 95.0% in the Ubiquitous
Language. A bug was found where the original implementation used > 94 which
caused campaigns to pause one redemption early at 94.9%.

## Decision
The threshold check uses >= 95.0 (greater than or equal to).
A campaign with budgetBurnPercent of exactly 95.0 must pause.
A campaign with budgetBurnPercent of 94.9 must not pause.

## Domain Rules Applied
- Budget exhaustion threshold is >= 95.0% (ubiquitous-language.md)
- BudgetExhausted must be emitted exactly once per Campaign

## Consequences

### Positive
- Campaigns run until their full budget is consumed up to the threshold
- MX team gets maximum value from every campaign

### Negative
- None identified

### Neutral
- Floating point comparison requires explicit precision handling (2 decimal places)

## Alternatives Considered
- **> 94:** Rejected — causes premature pause at 94.9%, loses ~0.1% of budget
- **> 95:** Rejected — allows campaigns to run past the threshold to 95.something%

## Validation
- Scenario 26: camp-012 at exactly 95.0% → CampaignPaused emitted ✅
- Scenario 8: camp-001 at 95.2% → CampaignPaused emitted ✅
- camp-001 at 94.9% → Campaign stays ACTIVE ✅
```

---

## Review Checklist

Before merging — verify your ADR:

- [ ] Status is set to ACCEPTED
- [ ] Context explains WHY this decision was needed now
- [ ] Decision uses Ubiquitous Language terms exactly
- [ ] Domain rules are explicitly cited
- [ ] At least two alternatives were considered and rejected with reasons
- [ ] Validation scenarios are named and expected outcomes stated
- [ ] Consequences (negative) are honest — not left blank
