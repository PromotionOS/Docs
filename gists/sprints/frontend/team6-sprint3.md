# Team 6 — Frontend Dashboard — Sprint 3

> Type: Build via SDD — deepening live integration
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

Sprint 1 wired campaign list. Sprint 2 wired offer check, analytics, and polling. Sprint 3 adds campaign detail view, budget top-up UI, and full eligibility breakdown — all wired to live services.

---

## Sprint 3 Goal

By end of sprint — MX team can view campaign detail, top up budget on paused campaigns, and the CX view shows full eligibility reasoning per offer. All 3 demo flows are functionally complete (Sprint 4 polishes and demos them).

---

## Requirements

### Requirement 1 — Campaign Detail View

Implement `/mx/campaigns/:id` detail page:

1. Campaign header: name, status badge, date range
2. Offer section: type, value, UPC list (with excluded indicator from Catalog Service)
3. Funding section: vendorId, vendor share %, Kroger share %
4. Budget section: total, burned, burn % progress bar
5. Analytics preview: lift %, ROI, redemption count (from Analytics Service)
6. "Top Up Budget" section — only visible when status is PAUSED with reason BUDGET_EXHAUSTED:
   - Input for new totalAmount (must be > current burnedAmount)
   - Button → `PUT /campaigns/:id/budget` on live Campaign Service
   - On success → campaign status returns to ACTIVE → badge updates → alert clears

### Requirement 2 — Full Eligibility Breakdown (CX View)

Enhance offer check results:

1. Expandable breakdown per offer showing each check result:
   - Segment check: ✅ PLATINUM matched / ❌ GOLD does not meet PLATINUM requirement
   - Threshold check: ✅ $67.50 meets $50.00 minimum / ❌ $35.00 is $15.00 short
   - Geo check: ✅ division-midwest in scope / ❌ division-southwest not in scope
   - Exclusion check: ✅ no excluded items / ❌ upc-wine-cab excluded (alcohol category)
2. Makes eligibility logic visible — educational for the session audience

### Requirement 3 — Flow 3 Prep (Budget Exhaustion)

Make Flow 3 demo-ready:

1. Show camp-001 in MX Dashboard at 94.8% burn (pre-seeded)
2. "Trigger Redemption" button — sends one $100 redemption via `POST /redeem`
   - customerId: cust-001
   - campaignId: camp-001
   - discountApplied: 100.00
   - cartTotal: 150.00
   - idempotencyKey: `demo-trigger-${Date.now()}`
3. After redemption — burn tips to 95%
4. Polling detects PAUSED status within 10 seconds
5. Alert banner fires
6. Offers stop serving for camp-001 in CX view

This is the live demo of the event chain firing.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| — | Campaign detail for camp-001 | vendorShare: 60%, burn: 95.2%, status: PAUSED |
| — | Budget top-up on camp-006 | Status → ACTIVE, badge updates |
| — | Alice Johnson eligibility breakdown | Shows PLATINUM ✅, geo ✅, no threshold |
| — | Bob Martinez breakdown for camp-004 | Shows GOLD ❌ for PLATINUM requirement |
| — | Trigger redemption button | camp-001 tips to 95%, alert fires within 10s |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Budget top-up UX — why you chose inline form vs modal vs separate page
- "Trigger Redemption" demo button — why this is acceptable for a demo but not production
