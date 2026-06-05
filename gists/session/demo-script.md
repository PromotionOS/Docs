# PromotionOS — Demo Script

> For: Facilitator
> When: End of Sprint 4
> Duration: ~15 minutes
> Audience: All participants

---

## Setup Before Demo

- [ ] All 8 services deployed and healthy on Railway
- [ ] Event Store live and healthy
- [ ] Frontend deployed on Vercel
- [ ] Test data seeded (all 12 campaigns including camp-013, 12 customers, 48 UPCs)
- [ ] Browser open on the Frontend Dashboard
- [ ] Terminal open with Railway logs visible (optional but impressive)
- [ ] All validation scenarios passing
- [ ] Camp-001 burnedAmount set to $47,400 (94.8%)
- [ ] Camp-013 reset to DRAFT ($75K, vendor-pepsi-001) — approval workflow must fire live
- [ ] Notification Service email delivery visible in logs
- [ ] WebSocket connection confirmed active in browser dev tools

---

## The Narrative

Before touching the screen:

> "Before I show you what you built — let me tell you what Oracle Trade Management does. It is the industry standard for retail trade promotions. Kroger, Walmart, every major grocery chain uses it. Implementation costs between $2 million and $10 million. Licensing costs $500,000 a year. It takes 18 months to implement. And it looks like it was built in 2003 — because it was.
>
> What you are about to see is what you built today. 8 teams. 4 sprints. 8 hours. 9 components — 8 services and a shared Event Store — all integrated and running in production. Let's go."

---

## Flow 1 — Campaign Creation (3 minutes)

**What it shows:** MX team creating and publishing a campaign. Funding validation. UPC conflict detection.

### Step 1 — Create a valid campaign

Navigate to MX Dashboard → click "New Campaign"

Fill in:
- Name: `Summer Drinks Promo`
- Offer Type: `PCT_OFF`
- Value: `15`
- UPCs: select `upc-water-24pk`, `upc-sparkling-12pk`
- Date Range: today → end of month
- Stack Permission: off

Say:
> "This is what a merchandiser does every day. In Oracle TM this is 20 clicks across 5 screens and takes 15 minutes. Here — one form."

### Step 2 — Add funding

Fill funding:
- Vendor ID: `vendor-pepsi-001`
- Vendor Share: `70`
- Kroger Share: `30`
- Budget: `$25,000`

Click Publish.

> "Campaign is live. Active. Eligibility rules are loaded. Analytics tracking has started."

Show campaign in the active list with status ACTIVE and burn at 0%.

### Step 3 — Show conflict detection

Click "New Campaign" again.

Fill in:
- Name: `Conflicting Drinks Promo`
- Offer Type: `PCT_OFF`
- Value: `10`
- UPCs: select `upc-water-24pk` (same UPC as above)

Click Publish.

Show the error:
> "UPC overlap detected. `upc-water-24pk` is already in an active campaign. Oracle TM would let this through and you'd discover the conflict at redemption time — two weeks later, in a report, after you've already paid out duplicate discounts."

---

## Flow 2 — Customer Offer Fetch (3 minutes)

**What it shows:** Eligibility engine in action. Segment matching. Exclusions. Geo validation.

Navigate to CX View.

### Step 1 — PLATINUM customer sees wine offer

Select customer: `Alice Johnson (PLATINUM)`
Cart total: `45.00`

Click "Check Offers"

Show results:
- `Weekend Mega Sale` — eligible — 20% off cola and chips
- `Platinum Member Exclusive` — eligible — 30% off wine
- `Spend $50 Get $10 Off` — ineligible — THRESHOLD_NOT_MET

Say:
> "Alice is PLATINUM. She sees the wine promo. She doesn't meet the $50 threshold yet."

### Step 2 — GOLD customer rejected from PLATINUM campaign

Select customer: `Bob Martinez (GOLD)`
Cart total: `45.00`

Click "Check Offers"

Show results:
- `Weekend Mega Sale` — eligible
- `Platinum Member Exclusive` — ineligible — SEGMENT_MISMATCH

Say:
> "Bob is GOLD. Same cart, different customer. He does not see the wine promo. The eligibility engine enforced the segment restriction without any if-statement in the frontend — that logic lives in the domain where it belongs."

### Step 3 — Threshold met

Select customer: `Carol White (SILVER)`
Cart total: `67.50`

Click "Check Offers"

Show results:
- `Spend $50 Get $10 Off` — eligible — $10 discount applied

Say:
> "Carol hits the threshold. $10 off. The cart total is evaluated after excluded items are removed. That is a non-trivial business rule. Team 2 implemented it from a spec in 45 minutes."

---

## Flow 3 — Budget Exhaustion (4 minutes)

**What it shows:** Real-time budget burn. Auto-pause. Cross-service event flow. Geo-targeted notification. WebSocket alert on frontend.

**This is the moment that lands.**

Navigate to MX Dashboard.

### Step 1 — Show camp-001 burn at 94.8%

Point to `Weekend Mega Sale` in the campaign list.

> "This campaign has burned 94.8% of its $50,000 budget. In Oracle TM — you would find this out tomorrow morning in a batch report. Here — look at the burn bar."

Show the amber progress bar at 94.8%.

### Step 2 — Trigger the final redemption

POST one redemption of $100 via API or UI button.

Wait. Watch the logs.

Show in sequence:
1. Redemption Service logs: `OfferRedeemed published`
2. Analytics Service logs: `burn updated, 95.2%`
3. Analytics Service logs: `BudgetExhausted published`
4. Campaign Service logs: `CampaignPaused`
5. Notification Service logs: `Resolving store managers for division MIDWEST → 12 recipients → email delivered`
6. Frontend: alert banner appears in under 1 second via WebSocket

Show campaign status: PAUSED.

Show the alert banner: "Weekend Mega Sale has been paused — budget exhausted"

### Step 3 — Explain what happened

> "Here is what happened automatically, with no human intervention:
>
> 1. A redemption came in from a POS terminal in Chicago
> 2. Redemption Service confirmed it and published OfferRedeemed
> 3. Analytics Service consumed that event and updated the burn tracker
> 4. Burn hit 95% — Analytics published BudgetExhausted
> 5. Campaign Service consumed BudgetExhausted and paused the campaign
> 6. Campaign published CampaignPaused
> 7. Notification Service called Catalog Service to resolve which store managers in the midwest division to notify — 12 managers, not a broadcast to everyone
> 8. Eligibility Service consumed CampaignPaused and removed the rules from its engine
> 9. Frontend received CampaignPaused via WebSocket and showed you this alert in under one second
>
> Seven services. Six domain events. Zero human intervention. All of it driven by the contracts your teams defined and respected."

### Step 4 — Prove offers stopped serving

Navigate to CX View.

Select customer: `Alice Johnson (PLATINUM)`

Click "Check Offers"

Show results — Weekend Mega Sale is NOT in the eligible offers list.

> "Alice can no longer redeem this offer. The campaign is paused. The eligibility engine has no rules for it anymore. This is what a live system looks like."

---

## Flow 4 — Approval Workflow (4 minutes)

**What it shows:** Campaign lifecycle gate for large-budget campaigns. Vendor approval loop. Automatic state transition via domain events.

**This is the newest flow — the one that required 8 teams to build together.**

Navigate to MX Dashboard.

### Step 1 — Show camp-013 in DRAFT

Point to `Pepsi Summer Push` (camp-013) in the campaign list.

> "This is a $75,000 campaign. Pepsi is the vendor. The MX team wants to publish it."

### Step 2 — Attempt to publish — get blocked

Click Publish on camp-013.

Show the error: `FUNDING_APPROVAL_REQUIRED — campaigns with budget >= $50,000 require vendor funding approval before publishing`

> "The system blocked it. A $75,000 campaign does not go live until the vendor approves the funding. This is a real business rule — no one hard-coded this check in the frontend. It is a domain rule enforced by Campaign Service."

Show the Event Store log: `FundingProposalSubmitted published for camp-013`

### Step 3 — Switch to Vendor approver view

Navigate to Vendor Portal → Vendor Service (Team 7's service).

Show the pending proposal for camp-013 visible in the approver queue.

> "Vendor TL's service is listening. FundingProposalSubmitted arrived from the Event Store. The proposal is in the queue."

Click Approve.

> "Approver approves. Watch Campaign Service."

### Step 4 — Camp-013 goes ACTIVE automatically

Show Campaign Service logs: `FundingApproved received for camp-013 → status PENDING_APPROVAL → ACTIVE`

Switch back to MX Dashboard.

Show camp-013 status: ACTIVE.

> "Nobody from the MX team did anything after clicking Publish. The system paused, waited for vendor approval, received it via a domain event, and activated the campaign automatically.
>
> Two TLs. Two services. Zero coordination beyond a contract skill that Team 7 wrote before implementing a single line of code."

---

## The Close

> "What you just saw is a fully functional trade promotions platform. It handles campaign lifecycle including a full funding approval workflow with automatic state transitions, eligibility rules with segment matching and exclusions, idempotent redemptions, real-time budget tracking, automatic campaign management, geo-targeted notifications routed by store division, vendor forecasting, and webhook delivery to external vendor systems. All validated against real-world retail data.
>
> Oracle charges $500,000 a year for this. You built the core in 8 hours.
>
> But here is what I actually want you to take away — it is not that you built it fast. It is HOW you built it.
>
> Every feature started with a spec. Every decision has an ADR. Every bug has an RCA. Every service respects its contracts. When Vendor TL needed Campaign TL to consume a new event — they did not send a Slack message. They wrote a contract skill. Campaign TL read it, built against it, and when Vendor TL deployed — it worked.
>
> And Claude was there for every step — not writing your code for you, but making the distance between a requirement and working, validated, documented code shorter than it has ever been.
>
> That is the multiplier. That is what you now know how to do."

---

## Fallback Plan

If any service is down during the demo:

- Have screenshots ready of each flow working
- Fall back to showing the validation scenario results in the terminal
- The narrative still lands — the code is real, the architecture is real, the decisions are documented

If the budget exhaustion flow doesn't trigger live:

- camp-001 is already PAUSED in test data at 95.2% burn
- Show the pre-existing state and walk through the event sequence manually
- The explanation of what happened is as powerful as watching it happen live

If the approval workflow (Flow 4) doesn't fire live:

- Show the Session 2 logs where camp-013 transitioned automatically
- Walk through the FundingProposalSubmitted → FundingApproved → ACTIVE sequence from the log output
- Emphasise that the contract skill was written before the code and both teams built against the same schema
