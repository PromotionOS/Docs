# Team 6 — Frontend Dashboard — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

Sprint 1 built MX campaign list, campaign form, and CX offer check. Sprint 2 added analytics dashboard and real-time polling. Sprint 3 wires the budget exhaustion alert and adds the campaign detail view for MX team.

---

## Sprint 3 Goal

By end of sprint — budget exhaustion alert fires in real time, campaign detail view shows full campaign health, and all 3 demo flows are functional end-to-end.

---

## Requirements

### Requirement 1 — Budget Exhaustion Alert (Flow 3)

This is the session's most important frontend moment. Wire it correctly.

1. Poll `GET /analytics/campaigns/:id/burn` every 10 seconds for each active campaign
2. When `budgetExhaustedEmitted: true` is returned:
   - Change campaign status badge to PAUSED (red)
   - Show alert banner at top of MX Dashboard:
     `"[Campaign Name] has been paused — budget exhausted"`
   - Remove campaign from active offer serving (it will already be gone from Eligibility)
3. Alert banner:
   - Persistent until dismissed
   - Stacks if multiple campaigns exhaust simultaneously
   - Includes campaign name, burn percentage, timestamp

### Requirement 2 — Campaign Detail View

Implement `/mx/campaigns/:id` detail page:

1. Campaign header: name, status badge, date range
2. Offer section: type, value, UPC list with exclusion indicators
3. Funding section: vendorId, vendor share %, Kroger share %
4. Budget section: total, burned, burn percentage bar
5. Analytics preview: lift %, ROI, redemption count (fetched from Analytics Service)
6. Edit button → open campaign form pre-filled (name and dates only — offer and funding locked after publish)
7. Budget Top-Up section (if PAUSED due to BUDGET_EXHAUSTED):
   - Input for new totalAmount
   - "Top Up Budget" button → `PUT /campaigns/:id/budget`
   - On success → campaign reactivates → status returns to ACTIVE

### Requirement 3 — Offer Eligibility Detail (CX View)

Enhance offer check results:

1. Show full eligibility breakdown per offer:
   - Segment check: passed/failed
   - Threshold check: passed/failed (show how much more needed if failed)
   - Exclusion check: list any excluded UPCs from their cart
   - Geo check: passed/failed
2. This makes the eligibility rules visible to session participants — educational

---

## Validation Checklist — All 3 Demo Flows

**Flow 1 — Campaign Creation:**
- [ ] Create campaign form submits correctly
- [ ] Publish without funding shows `NO_FUNDING_SOURCE` error
- [ ] Published campaign appears in list as ACTIVE
- [ ] UPC overlap shows `UPC_OVERLAP` error with conflicting campaign name

**Flow 2 — Customer Offer Fetch:**
- [ ] cust-001 (PLATINUM) sees wine promo from camp-004
- [ ] cust-002 (GOLD) sees SEGMENT_MISMATCH for camp-004
- [ ] cust-003 cartTotal:35.00 sees THRESHOLD_NOT_MET for camp-003
- [ ] cust-003 cartTotal:67.50 sees $10 off from camp-003
- [ ] cust-010 (southwest) sees GEO_MISMATCH for camp-001

**Flow 3 — Budget Exhaustion:**
- [ ] camp-001 shows 95.2% burn in red
- [ ] camp-001 status shows PAUSED
- [ ] Alert banner visible: "Weekend Mega Sale has been paused — budget exhausted"
- [ ] After exhaustion — cust-001 no longer sees camp-001 in offer check

---

## Sprint 3 ADR Topics

Record an ADR for:
- Budget top-up UX flow — inline vs modal vs separate page
- Eligibility breakdown display — how much detail to show to CX users
