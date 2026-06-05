# Session 4 Script — Demo

> Facilitator guide.
> Duration: 2 hours
> Teams: All 8 simultaneously
> Depends on: Sessions 1, 2, 3 complete

---

## Pre-Session (15 min before)

- Verify ALL 8 services live and healthy
- Run all validation scenarios — note which ones pass and which do not
- Have the demo script open (demo-script.md)
- Have Vercel frontend URL ready
- Set camp-001 burnedAmount to $47,400 (94.8% burn) if not already there
- One test redemption of $100 will tip it to 95% live in the demo
- Reset camp-013 to DRAFT status for the Flow 4 demo (delete and re-seed, or reset status directly)
- Confirm vendor-pepsi-001 approver endpoint is live and the approval webhook is wired

---

## Opening (0:00 — 0:08)

**"This is the final session. Today we wire everything together and we demo."**

**"But before the demo — Team 6 has work to do. They are wiring the four demo flows right now. The rest of you have one job in this session: make sure your service stays healthy and fix anything that breaks as Team 6 integrates against you."**

**"Think of it like a real launch day. Frontend is integrating. They will hit your APIs. If something is broken — you will hear about it immediately. Fix it."**

**"Here is the deal we make right now: by 1:45 the demo runs. Whatever state the system is in at 1:45 — that is what we demo. No extensions. No 'just five more minutes'. What ships is what we show."**

---

## Integration Hour (0:08 — 1:00)

**Sprint 4 gists to Team 6:**

Share Team 6 Sprint 4 secret gist.

**"Team 6 — you have four flows to wire this session. Start with Flow 1 (campaign creation). Confirm it works end-to-end before moving to Flow 2. Do not try to build all four at once. Flow 4 is the approval workflow — it is the newest and will need the most coordination with Vendor TL and Campaign TL."**

**"Everyone else — share your service URL with Team 6 right now. They need it to wire their API clients. If your health check is not returning 200, fix that before anything else."**

**During this hour — support Team 6:**

When Team 6 hits a CORS issue:
**"Backend teams — add CORS headers for the Vercel domain. This is expected. One line in your Spring Boot or Gin config."**

When Team 6 hits a 501 Not Implemented:
**"That endpoint has not been implemented yet. Team 6 — build the UI with mock data for that flow and we will swap in the live data when the backend is ready."**

When Team 6's offer check works:
**"Pull that up on the projector."**

Show cust-001 (PLATINUM) seeing the wine promo. Then switch to cust-002 (GOLD) — no wine promo, SEGMENT_MISMATCH shown.

**"That right there. Two customers. Different loyalty tiers. Different offers. The eligibility engine you built in Sessions 1 and 2 just made that decision correctly. Not hardcoded. Not an if-statement in the frontend. A domain rule, enforced by a service, validated against real data."**

When Flow 4 approval workflow is being wired:
**"Vendor TL and Campaign TL — stand by. Team 6 is about to wire the approval flow. They will submit camp-013 through the frontend. Your services need to handle FundingProposalSubmitted → approver action → FundingApproved → ACTIVE transition without any manual step from Team 6."**

---

## Demo Prep (1:00 — 1:30)

**"Team 6 — freeze new features at 1:00. Polish what works. The demo is in 45 minutes."**

Run through demo flows privately with Team 6:

**Flow 1 — Campaign Creation:**
- Create a new campaign
- Add funding
- Publish
- Confirm it appears in the list as ACTIVE

If publish fails with an error — note the error, decide whether to fix or narrate around it.

**Flow 2 — Customer Offer Check:**
- Select Alice Johnson (PLATINUM)
- See wine promo eligible
- Select Bob Martinez (GOLD)
- See SEGMENT_MISMATCH for wine promo
- Select Carol White with cartTotal 35.00
- See THRESHOLD_NOT_MET for spend $50 promo

**Flow 3 — Budget Exhaustion:**
- Show camp-001 at 94.8% burn
- POST one redemption of $100 via API or UI
- Watch burn hit 95%
- BudgetExhausted fires
- Campaign status → PAUSED
- Notification Service delivers email to store managers (midwest geo routing)
- Alert banner appears on frontend via WebSocket

If the live chain does not fire: show the pre-seeded 95.2% camp-001 and walk through what happened. The pre-seeded state is a valid demo — explain the event chain that produced it.

**Flow 4 — Approval Workflow (new this session):**
- Show camp-013 in DRAFT state ($75K budget, vendor-pepsi-001)
- MX team attempts to publish — system blocks with FUNDING_APPROVAL_REQUIRED
- Show FundingProposalSubmitted event in Event Store logs
- Switch to Vendor TL's approver view — FundingProposalSubmitted is visible
- Approver approves — Vendor Service publishes FundingApproved
- Show Campaign Service logs: `FundingApproved received → camp-013 PENDING_APPROVAL → ACTIVE`
- Frontend shows camp-013 as ACTIVE without any manual step from MX team

If Flow 4 does not work end-to-end live: narrate from the Session 2 logs where camp-013 transitioned automatically. The story is in the architecture — tell it clearly.

---

## Final Validation (1:30 — 1:40)

**"Last validation pass. Every team — run your scenarios one more time. Report in."**

Count the passing scenarios. Be honest with the room about what passes and what does not.

**"We have X of 40 scenarios passing. Those X scenarios represent real business logic running on real infrastructure against real data. That is what we demo."**

---

## The Demo (1:40 — 1:55)

Follow the demo-script.md exactly.

Open with:

**"Before I show you what we built — let me tell you what Oracle Trade Management does. Kroger uses it. Every major grocery retailer uses it. Implementation costs between $2 million and $10 million. Licensing costs $500,000 a year. It was built in the early 2000s and it looks like it."**

**"What you are about to see is what these teams built. 8 teams. 4 sessions. 8 hours. A 9-component system — 8 services and a shared Event Store — all integrated and running on Railway."**

Run Flow 1. Run Flow 2. Run Flow 3. Run Flow 4.

Close with:

**"That is a functioning trade promotions platform. It handles campaign lifecycle including a full funding approval workflow, eligibility rules with segment matching and exclusions, idempotent redemptions, real-time budget tracking, automatic campaign management, vendor forecasting, and geo-targeted notifications. All validated against real-world retail data."**

**"Oracle charges $500,000 a year for this. You built the core in 8 hours."**

---

## Closing (1:55 — 2:00)

**"I want to ask each team one question. Not 'what did you build'. You know what you built."**

**"The question is: what did you build, which scenarios pass, and — for Vendor TL and Notification TL — how does your service make the other 7 services smarter?"**

Go around. Listen.

Then:

**"Here is what I heard. Weeks. Maybe months. Not because you are not good engineers. Because the distance between a requirement and working, validated, documented code is normally very long."**

**"Claude did not close that distance by writing your code. It closed it by making the process — spec, domain rules, acceptance criteria, implementation — faster than it has ever been."**

**"That is the multiplier. Not the code it writes. The process it enables."**

**"You now know how to use it. Take it back to your teams. Take it into your next project. Build the next thing faster, with more confidence, with better documentation, and with code that actually does what the requirements say."**

**"Thank you."**

---

## Post-Demo Reset

After Flow 4 is demoed:
- [ ] Delete camp-013 (or reset to DRAFT) so it can be re-run for any audience member who wants to see the approval workflow live again
- [ ] Confirm camp-001 is back at 94.8% if burn exhaustion will be re-demoed

---

## Session 4 Checklist

- [ ] All 8 services healthy at session start
- [ ] Camp-001 burnedAmount set to $47,400 (94.8%) pre-demo
- [ ] Camp-013 reset to DRAFT pre-demo (approval workflow must fire live)
- [ ] Flow 1 works end-to-end
- [ ] Flow 2 works for at least 2 customer scenarios
- [ ] Flow 3 shows budget exhaustion (live or pre-seeded) + Notification email + WebSocket alert
- [ ] Flow 4 shows camp-013 approval workflow end-to-end
- [ ] Scenario count announced honestly (out of 40)
- [ ] Every team speaks in the closing round
- [ ] Vendor TL and Notification TL address how their service makes the other 7 smarter
