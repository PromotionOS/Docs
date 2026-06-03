# PromotionOS — Session Guide

> For: Facilitator
> Purpose: How to run the 4 sprint sessions
> Each sprint: 2 hours

---

## Overview

PromotionOS is built across 4 sprints of 2 hours each. Each sprint is a focused development session where teams implement their bounded context using Spec Driven Development (SDD), Domain Driven Design (DDD), and Claude as a force multiplier.

**The goal is not just to ship code. The goal is to demonstrate how Claude changes the way engineers think, spec, and build.**

---

## Before Every Sprint

### 15 minutes before start

- [ ] Confirm all services are deployed and returning 501s (or pre-built for Team 1 + Team 5)
- [ ] Confirm test data is seeded in all DBs
- [ ] Confirm Railway + Vercel are healthy
- [ ] Confirm all skills are accessible in Claude
- [ ] Prepare secret gist links for this sprint's requirements — do NOT share until sprint starts
- [ ] Have the validation scenario list ready for reference

### Room setup

- [ ] Each team has access to their service repo
- [ ] Each team has Claude Code or Claude.ai open
- [ ] Each team has the ubiquitous language doc open
- [ ] Each team has the relevant contract docs open

---

## Sprint Opening (5 minutes)

Say this at the start of every sprint:

> "Welcome to Sprint [N]. You have 2 hours. Your job is to implement your bounded context using SDD. Every feature starts with a Spec. Every decision gets an ADR. Every bug gets an RCA before a fix. Claude is your pair programmer — not your author. You write the requirements into a spec, Claude helps you implement. The spec is yours. The decisions are yours. Claude is the multiplier.
>
> Your sprint requirements are in the secret gist I am sharing now. Read them fully before touching any code. Use the sdd-spec-writer skill first. You have 15 minutes to read and 20 minutes to write your spec. After that — build."

Share the secret gist links now.

---

## Sprint Timeline

```
0:00 - 0:15   Read requirements from secret gist
0:15 - 0:35   Write spec using sdd-spec-writer skill
0:35 - 0:45   Cross-team contract review + ADR
0:45 - 0:50   Record ADR using adr-recorder skill
0:50 - 1:40   Implement using codebase skill + Claude
1:40 - 1:50   Validate against test data scenarios
1:50 - 2:00   PR → GitHub Actions → deploy → confirm live
```

---

## Facilitator Interventions

### If a team skips the spec and goes straight to code

Stop them immediately.

> "No code without a spec. Open the sdd-spec-writer skill and write the spec first. I know it feels slower. It is not. You will write better code faster with a spec than without one."

### If a team uses wrong domain terms

> "Check the ubiquitous language doc. What is the correct term? Change it in your spec before you implement. The term in your spec becomes the variable name in your code."

### If a team is blocked by another team's contract

> "Use the contract skill. It tells you exactly what the other team's service sends you. You do not need to wait for them — the contract is the agreement."

### If a team changes a contract without notifying the other team

> "Stop. Update the contract skill first. Bump the schema version. Tell the consuming team now. Then continue."

### If a team finishes early

Give them an additional requirement:
> "Write a second ADR for a decision you made that you did not document. Or add a validation test for an edge case not in your spec."

### If a team is falling behind

At the 1:30 mark — if they have not started validation:
> "Stop implementing new features. Validate what you have against the test data. A working partial implementation is better than a broken full implementation."

---

## Sprint Closing (5 minutes)

At 1:55:

> "Wrap up. Open a PR if you haven't. Check GitHub Actions. In 5 minutes we do a quick round — each team states: what they built, which scenarios pass, and one thing Claude helped them do that they couldn't have done as fast alone."

Quick round (30 seconds per team):
- What did you build?
- Which validation scenarios pass?
- One Claude multiplier moment

---

## Sprint 1 Specific Notes

**Teams:** Eligibility (Team 2) + Catalog & Customer (Team 4)
**Watch for:** Team 4 publishing CatalogItemExcluded before Team 2 is ready to consume it. Remind both teams to check the `catalog-eligibility-contract` skill together at the 0:35 mark.

**Key teaching moment:** When Team 4 changes their event schema mid-sprint — pause both teams and walk through the contract skill update process publicly. This is the session's most important demonstration.

---

## Sprint 2 Specific Notes

**Team:** Redemption (Team 3)
**Watch for:** Team 3 not calling Eligibility Service before confirming redemption. Remind them the contract requires this call.
**Key teaching moment:** The idempotency scenario — show that the same key from two different tenants must succeed. This is where tenant isolation understanding is tested.

---

## Sprint 3 Specific Notes

**Teams:** Campaign bug fix (Team 1) + Analytics bug fix (Team 5)
**Watch for:** Teams fixing the bug without writing an RCA first. Stop them.
**Key teaching moment:** The off-by-one bug in Campaign Service. Walk through the RCA process publicly with Team 1 — show how Claude helps identify root cause from a symptom description.

---

## Sprint 4 Specific Notes

**Team:** Frontend (Team 6)
**Watch for:** Team 6 building UI before confirming all 3 backend flows work end-to-end. Have them test each API call manually first.
**Key teaching moment:** Flow 3 — budget exhaustion auto-pause. When the burn chart hits 95% and the campaign disappears from the active list in real time — that is the session's demo moment.

---

## The Demo (End of Sprint 4)

Run the 3 flows live in front of everyone. See `demo-script.md` for exact steps.

After the demo:

> "What you just saw — 6 teams, 4 sprints, 8 hours total — is a functioning trade promotions platform. Oracle charges $500,000 a year for this. You built the core in 8 hours. Not because you are faster than Oracle's engineers. Because Claude compressed the distance between a requirement and working, validated code. That is the multiplier."

---

## Common Facilitator Mistakes

| Mistake | Impact |
|---------|--------|
| Sharing sprint requirements before sprint starts | Teams skip reading and go straight to code |
| Letting teams skip specs under time pressure | Kills the SDD demonstration |
| Not enforcing contract skill updates | Teams communicate via Slack, defeats the point |
| Skipping the quick round at sprint close | Teams don't reflect on Claude's contribution |
| Not stopping the off-by-one RCA as a public moment | Biggest teaching opportunity of Sprint 3 |
