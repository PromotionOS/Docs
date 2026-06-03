# PromotionOS — Demo Script

> For: Facilitator
> When: End of Sprint 4
> Duration: ~10 minutes
> Audience: All participants

---

## Setup Before Demo

- [ ] All 6 services deployed and healthy on Railway
- [ ] Frontend deployed on Vercel
- [ ] Test data seeded (all 12 campaigns, 12 customers, 48 UPCs)
- [ ] Browser open on the Frontend Dashboard
- [ ] Terminal open with Railway logs visible (optional but impressive)
- [ ] All 30 validation scenarios passing

---

## The Narrative

Before touching the screen:

> "Before I show you what you built — let me tell you what Oracle Trade Management does. It is the industry standard for retail trade promotions. Kroger, Walmart, every major grocery chain uses it. Implementation costs between $2 million and $10 million. Licensing costs $500,000 a year. It takes 18 months to implement. And it looks like it was built in 2003 — because it was.
>
> What you are about to see is what you built today. 6 teams. 4 sprints. 8 hours. Let's go."

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

**What it shows:** Real-time budget burn. Auto-pause. Cross-service event flow. Alert on frontend.

**This is the moment that lands.**

Navigate to MX Dashboard.

### Step 1 — Show camp-001 burn at 95.2%

Point to `Weekend Mega Sale` in the campaign list.

> "This campaign has burned 95.2% of its $50,000 budget. In Oracle TM — you would find this out tomorrow morning in a batch report. Here — look at the burn bar."

Show the red progress bar at 95.2%.

Show campaign status: PAUSED.

Show the alert banner: "Weekend Mega Sale has been paused — budget exhausted"

### Step 2 — Explain what happened

> "Here is what happened automatically, with no human intervention:
>
> 1. A redemption came in from a POS terminal in Chicago
> 2. Redemption Service confirmed it and published OfferRedeemed
> 3. Analytics Service consumed that event and updated the burn tracker
> 4. Burn hit 95% — Analytics published BudgetExhausted
> 5. Campaign Service consumed BudgetExhausted and paused the campaign
> 6. Campaign published CampaignPaused
> 7. Eligibility Service consumed CampaignPaused and removed the rules from its engine
> 8. Frontend consumed CampaignPaused and showed you this alert
>
> Six services. Five domain events. Zero human intervention. All of it driven by the contracts your teams defined and respected."

### Step 3 — Prove offers stopped serving

Navigate to CX View.

Select customer: `Alice Johnson (PLATINUM)`

Click "Check Offers"

Show results — Weekend Mega Sale is NOT in the eligible offers list.

> "Alice can no longer redeem this offer. The campaign is paused. The eligibility engine has no rules for it anymore. This is what a live system looks like."

---

## The Close

> "What you just saw is a fully functional trade promotions platform. It handles campaign lifecycle, eligibility rules with segment matching and exclusions, idempotent redemptions, real-time budget tracking, and automatic campaign management — all validated against real-world retail data.
>
> Oracle charges $500,000 a year for this. You built the core in 8 hours.
>
> But here is what I actually want you to take away — it is not that you built it fast. It is HOW you built it.
>
> Every feature started with a spec. Every decision has an ADR. Every bug has an RCA. Every service respects its contracts. And Claude was there for every step — not writing your code for you, but making the distance between a requirement and working, validated, documented code shorter than it has ever been.
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
