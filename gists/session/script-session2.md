# Session 2 Script — Depth

> Facilitator guide.
> Duration: 2 hours
> Teams: All 8 simultaneously
> Depends on: Session 1 complete, all services deployed

---

## Pre-Session (15 min before)

- Verify all 8 services are still live from Session 1
- Confirm Session 1 scenarios still pass (regression check)
- Have Sprint 2 secret gists ready
- Confirm camp-013 exists in test data: $75K budget, vendor-pepsi-001, status PENDING_APPROVAL
- Note: Vendor TL and Campaign TL have the key contract moment this session — watch for it

---

## Opening (0:00 — 0:08)

**"Welcome back. Last session you built foundations. Today the system starts talking to itself."**

**"This session introduces the most important concept of the workshop: the contract skill. When your team changes what your service publishes or exposes — you do not send a Slack message. You do not call a meeting. You update the contract skill. The consuming team uses that skill to understand the blast radius and refactor safely."**

**"This is how engineering orgs scale. Not by adding more coordination. By encoding knowledge in skills that travel with the code."**

**"Today you will feel this for real. Not once — twice. Vendor TL will publish a new domain event. Campaign TL needs to consume it. The contract skill is how they communicate. And separately, Notification TL is replacing their polling approach with a WebSocket hub. Frontend TL will switch over. Watch for both moments — they are the key teaching moments of this session."**

---

## Session 1 Retrospective (0:08 — 0:15)

**"Quick round. 20 seconds each team. One thing that surprised you in Session 1."**

Listen for: teams that discovered the spec saved them time, teams that hit integration issues, teams that found the domain language confusing at first. Watch especially for Notification TL and how the first cross-service event feel landed.

Respond to each with one sentence. Do not over-elaborate.

Then:

**"The most common thing I heard last session: 'I thought writing the spec would slow me down. It sped me up.' That is the insight. Hold onto it."**

---

## Deployment Order Reminder (0:15 — 0:18)

**"Critical note for this session. Deployment order matters."**

Write or share:

```
Vendor TL deploys FundingApproved event (new contract skill first)
→ Campaign TL consumes FundingApproved → camp-013 transitions PENDING_APPROVAL → ACTIVE
→ Catalog TL deploys (if not already done)
→ Team 2 validates exclusions
→ Team 3 deploys
→ Team 5 consumes OfferRedeemed
→ Team 1 receives BudgetExhausted
→ Notification TL deploys WebSocket hub
→ Frontend TL switches from polling to WebSocket
```

**"If you are Team 2 and Team 4 has not deployed yet — keep implementing. Use the catalog-eligibility-contract skill to build against the contract. When they deploy, your integration will work."**

**"If you are Team 5 and Team 3 has not deployed — same. Your event consumer is wired. When Team 3 pushes their first redemption event, your burn tracker fires. You do not need to wait."**

**"Vendor TL — before you publish FundingApproved, you must define the event schema in a new contract skill. That skill is Campaign TL's only source of truth for what the event looks like. Do not implement first and document later. The skill is the contract. Write it first."**

---

## Sprint 2 Requirements (0:18 — 0:22)

**"Sharing Sprint 2 requirements now."**

Share secret gists to each team.

**"Important: Team 2 — your sprint is the most loaded in the entire programme. You have threshold evaluation, exclusion check, geo validation, and stack validation. In the next session you will get stack validation. Today focus on threshold, exclusion, and geo. Do not try to do everything. Do it right."**

**"Team 3 — scenario 28, the T+24 claim. We have seeded a redemption record with scheduled_at set to 23 hours 59 minutes ago. Your claim processing job will fire it within minutes of deploying. You do not need to wait 24 hours."**

**"Vendor TL — Sprint 2 scope is the approval workflow. FundingProposalSubmitted already fires when camp-013 is submitted. Your job this session is to implement the approver endpoint and publish FundingApproved or FundingRejected. Campaign TL is waiting for that event to unblock camp-013."**

**"Notification TL — Sprint 2 scope is the WebSocket hub. Replace the polling approach with a real-time push channel. Frontend TL will switch their connection over when you are live."**

---

## Build Phase (0:22 — 1:40)

Teams work. You watch for both contract moments.

**The approval workflow contract moment — when it happens, pause everyone:**

When Vendor TL publishes the FundingApproved event and Campaign TL's service auto-transitions camp-013 to ACTIVE:

**"Everyone pause for 90 seconds. Watch Campaign TL's logs."**

Show Campaign Service log: `FundingApproved received for camp-013 → status PENDING_APPROVAL → ACTIVE`

**"camp-013. $75,000 budget. Status was PENDING_APPROVAL. Now it is ACTIVE. Nobody touched it. No manual step. No button click. Vendor TL's service published a domain event. Campaign TL's service consumed it and executed the state transition automatically."**

**"Two TLs. Two services. Zero coordination beyond a contract skill."**

Pause for effect.

**"That is the key moment of Session 2. That is what contract-driven event architecture looks like in production."**

Resume.

**The WebSocket contract moment — when it happens:**

When Notification TL's WebSocket hub is live and Frontend TL switches over:

**"Everyone check the Frontend. No more polling. Budget alerts now arrive in under one second from the moment the event fires in Analytics."**

**"Team 8 built a push channel. Team 6 consumed it. The contract skill described the message shape. That is the pattern."**

**At 0:50 — the event chain moment:**

If Team 3 has deployed their OfferRedeemed event and Team 5 is consuming it:

**"Everyone check Railway logs. Team 5 — show your analytics service logs."**

If burn tracking is updating: **"That is a live OfferRedeemed event flowing from Redemption to Analytics. Real data. Real event. Two services talking to each other through the Event Store."**

**At 1:15 — check-in:**
**"We are 53 minutes from end. If you are still on your first requirement — that is fine. Do one thing well. Teams building the event chain — BudgetExhausted needs to fire before we close. Prioritise that."**

---

## The Event Chain Demo (1:40 — 1:48)

**If the chain is working:**

**"Let me show you something."**

Trigger a redemption via the API or show it in the Redemption Service.

**"One redemption. Watch what happens."**

Show in sequence:
1. Redemption Service logs: `OfferRedeemed published`
2. Analytics Service logs: `burn updated, 95.2%`
3. Analytics Service logs: `BudgetExhausted published`
4. Campaign Service logs: `CampaignPaused`
5. Notification Service logs: `CampaignPaused email delivered` (or `WebSocket alert pushed`)

**"Six seconds. Five services. Zero manual coordination. That is what event-driven architecture feels like when it works."**

**If the chain is not working:**
Skip the live demo. Show the test data instead — camp-001 already paused at 95.2% burn. Explain the chain conceptually. If the approval workflow fired, lead with that instead — it is the more powerful story for this session.

---

## Sprint Close (1:48 — 2:00)

**"30 seconds each. Scenarios passing. One thing you learned."**

After the round:

**"This session you built something harder. You integrated. You hit real dependency problems and solved them with skills instead of meetings. You saw an event chain fire across multiple services. And — if the stars aligned — you saw a $75,000 campaign go from PENDING_APPROVAL to ACTIVE without a single human action after the approver clicked approve."**

**"Next session: integration, hardening, forecasting, and store geo routing. After that — the demo. See you then."**

---

## Session 2 Checklist

- [ ] Vendor TL wrote FundingApproved contract skill before implementing
- [ ] Camp-013 transitions from PENDING_APPROVAL to ACTIVE when FundingApproved fires (scenario 33)
- [ ] Contract skill update happened at least once publicly (catalog-eligibility or funding approval)
- [ ] Scenarios 1, 4, 5, 6 passing (Team 2 threshold + exclusion)
- [ ] Scenarios 13, 14 passing (Team 2 geo)
- [ ] Scenario 10 passing (Team 1 UPC overlap)
- [ ] Scenarios 24, 25 passing (tobacco geo)
- [ ] Scenario 7 still passing (Team 3 regression)
- [ ] OfferRedeemed event flowing to Analytics (even if chain not complete)
- [ ] Scenario 29 passing (claim deduction calculation)
- [ ] Scenario 36 passing (WebSocket alert fires in under 1 second after BudgetExhausted)
- [ ] Scenario 37 passing (midwest store managers only notified for camp-001 CampaignPaused)
- [ ] All PRs merged and deployed
