# Session 3 Script — Integration

> Facilitator guide.
> Duration: 2 hours
> Teams: All 8 simultaneously
> Depends on: Sessions 1 and 2 complete

---

## Pre-Session (15 min before)

- Verify all 8 services live and Session 1 + 2 scenarios still passing
- Confirm Team 1 Sprint 3 BudgetUpdated event is wired correctly
- Confirm camp-013 is ACTIVE (FundingApproved should have fired in Session 2)
- Have Sprint 3 secret gists ready
- Note: Catalog TL's GetStoresByDivision endpoint is a Session 3 deliverable — Notification TL depends on it for midwest geo routing
- Note: Team 1 and Team 5 have a new dependency this session — coordinate their deployment order

---

## Opening (0:00 — 0:08)

**"Two sessions down. You have working services. You have passing scenarios. You have a system that partially integrates. Today we finish the integration and harden everything."**

**"One important note before we start. Session 3 has a dependency that was not obvious in the SDL. Team 1 is publishing a new BudgetUpdated event this session. Team 5 needs to consume it. This is a new contract being created live, in session, by real teams."**

**"This is not a bug in the plan. This is what real engineering looks like. Requirements evolve. Contracts change. The question is: how do you handle it without breaking things?"**

**"The answer: contract skill first. Team 1 — before you implement BudgetUpdated, define the schema in a new contract skill. Team 5 — wait for that skill before you build your consumer. The skill is the handshake."**

**"This session also has a moment for Notification TL and Catalog TL. Notification TL has been routing CampaignPaused emails without knowing which specific store managers to notify — they have been broadcasting. This session, Catalog TL's GetStoresByDivision endpoint goes live. Notification TL calls it to resolve exactly which midwest store managers get the camp-001 alert. Two services. One contract. Watch for it."**

---

## Regression Check (0:08 — 0:15)

**"Before we build anything new — regression check. Every team: run your Sprint 1 and Sprint 2 scenarios right now. 5 minutes. Report if anything broke."**

Wait for results. Address any regressions immediately before proceeding.

**"If a regression happened — that is actually good news. It means your test coverage caught it. Fix it before adding anything new. A broken foundation is worse than a missing feature."**

**"Vendor TL and Notification TL — also verify the approval workflow is still intact. camp-013 should be ACTIVE and scenario 33 should still pass."**

---

## Sprint 3 Requirements (0:15 — 0:20)

Share secret gists.

**"Team 5 — important note. Sprint 3 has you rebuilding the LiftCalculator. You fixed the null guard bug in Sprint 1. When you rebuild the full implementation in Sprint 3, that null guard must survive. Run scenarios 18, 19, 20 as your regression test before merging. If any of those fail — stop and fix before you push."**

**"Team 1 — the summary endpoint you expose this session unblocks Team 3's claim deduction. Make that your first PR. Everything else follows."**

**"Vendor TL — Sprint 3 scope is the forecasting endpoint. GET /vendor/forecasts for camp-013 must return predictedLift calculated from: category (beverage) × offerValue (PCT_OFF) × seasonBenchmark (summer). Write the spec for the calculation before you write a single line of math."**

**"Notification TL — Sprint 3 scope is two things: ClaimSubmitted webhook delivery to vendor-pepsi-001, and FundingRejected notification to the MX team. The ClaimSubmitted webhook is the first outbound HTTP call your service makes to an external vendor endpoint. Write an ADR for your retry and failure strategy before you implement."**

**"Catalog TL — GetStoresByDivision is your Sprint 3 deliverable. It is small but unblocks Notification TL immediately. Prioritise it first."**

---

## Build Phase (0:20 — 1:40)

Teams work.

**Watch for these specific moments:**

**When Team 1 writes the BudgetUpdated contract skill:**
Pull up their skill on screen.

**"Team 5 — this is your contract. This is what Team 1 is promising to publish. Build your consumer against this. Not against Team 1's code. Against this skill."**

**When Team 5's LiftCalculator is rebuilt:**
**"Team 5 — before you merge. Run scenarios 18, 19, 20. The divide-by-zero fix from Sprint 1 must still be in there."**

**When Team 3 gets claim deduction working:**
**"Team 3 — run scenario 29. Discount 6.99, vendor share 60%. Expected deduction: 4.19. Does your system return 4.19?"**

**The key teaching moment — Notification + Catalog integration:**

When Catalog TL's GetStoresByDivision is live and Notification TL calls it to resolve midwest managers:

**"Everyone pause. This is the moment I mentioned at the top."**

Show Notification Service log: `Resolving store managers for division MIDWEST → GET /catalog/stores?division=MIDWEST → 12 managers resolved → CampaignPaused email delivered to 12 recipients`

**"That is Notification Service calling Catalog Service to answer the question: who in the midwest needs to know this campaign paused? Notification TL used the contract-notification-catalog skill to know exactly what Catalog's API returns. They built against the contract. When Catalog deployed — it worked."**

**"This is what a mature microservices system does. Services resolve domain questions by collaborating. The contract skill made that collaboration safe."**

**When Vendor TL's forecasting endpoint is live:**
**"Vendor TL — run scenario 39. GET /vendor/forecasts for camp-013. Check that predictedLift accounts for the beverage category, the PCT_OFF offer, and the summer benchmark. Show the response in the room."**

**When Notification TL's ClaimSubmitted webhook fires:**
**"Team 8 — scenario 38. Trigger a claim submission and show the webhook delivery log for vendor-pepsi-001. Confirmed delivery in under 5 seconds."**

**At 1:10:**
**"30 minutes to end. Integration focus now. If you have a feature half-done, consider whether finishing it or validating what you have is more valuable. A deployed, passing scenario beats an undeployed feature every time."**

---

## Stack Validation (if time permits)

If Team 2 gets to stack validation this session:

**"Team 2 — scenarios 15 and 16. Two stackable campaigns, then a third that exceeds the limit. This is the most complex eligibility rule in the system. Write the spec before you implement. The edge case is subtle: the stack limit is the minimum across all campaigns being applied — not the maximum."**

---

## Sprint Close (1:48 — 2:00)

**"One minute each team. What did you build? What is integrated? What is at risk for the demo?"**

Be honest about what is not done. Do not pretend.

After the round:

**"Here is where we are. We have a real system. 9 components — 8 services and an Event Store. Most of it integrates. Some parts are not complete. That is engineering. You shipped real features in real time under real constraints."**

**"Next session is the demo session. Frontend team wires everything together. You see your work on screen. A campaign gets created. Offers get served. A budget exhausts. And a $75,000 campaign goes through the full approval workflow — live. In front of everyone."**

**"One ask before next session: make sure your service is deployed and healthy. Team 6 is depending on every backend service being live. If your service is down, their demo flow breaks. Check it tonight."**

---

## Session 3 Checklist

- [ ] All Session 1 + 2 scenarios still passing (regression)
- [ ] Scenario 33 still passing (camp-013 ACTIVE — approval workflow intact)
- [ ] Team 1 BudgetUpdated contract skill published before Team 5 builds consumer
- [ ] Scenarios 18, 19, 20 still passing after Team 5 LiftCalculator rebuild
- [ ] Team 1 `/campaigns/:id/summary` endpoint live (unblocks Team 3)
- [ ] Scenario 29 passing (claim deduction)
- [ ] Catalog TL GetStoresByDivision endpoint live
- [ ] Scenario 37 passing (midwest store managers only — Notification calls Catalog)
- [ ] Scenario 38 passing (ClaimSubmitted webhook delivered to vendor-pepsi-001)
- [ ] Scenario 39 passing (predictedLift for camp-013 returns correct forecast)
- [ ] Team 2 audit log working
- [ ] Team 3 GET /redemptions/:id working
- [ ] All services healthy heading into Session 4
