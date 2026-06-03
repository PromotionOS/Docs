# Team 6 — Frontend Dashboard — Sprint 4

> Type: Demo Preparation + Full Integration
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, pr-reviewer

---

## Context

Sprints 1-3 built MX dashboard, CX view, analytics dashboard, and real-time polling. Sprint 4 is the final integration sprint — wire all 3 demo flows end-to-end with live backend services and make the system demo-ready.

---

## Sprint 4 Goal

All 3 demo flows work end-to-end against live backend services. The Frontend is the cohesive face of everything every team built. Demo-ready on Vercel.

---

## Requirements

### Requirement 1 — Demo Flow 1: Campaign Creation (End-to-End)

Wire fully against live Campaign Service:

1. Create campaign form → `POST /campaigns` (live)
2. Add funding → part of create request
3. Publish → `PUT /campaigns/:id/publish` (live)
4. Campaign appears in list → `GET /campaigns` (live, polling)
5. Error: UPC overlap → show conflicting campaign name (fetch from campaign list)
6. Error: no funding → clear message

### Requirement 2 — Demo Flow 2: Customer Offer Fetch (End-to-End)

Wire fully against live Eligibility Service:

1. Customer selector → 5 pre-loaded test customers
2. Cart total input
3. Check Offers → `POST /eligibility/check` for each active campaign (live)
4. Show eligible offers with discount value
5. Show ineligible offers with human-readable reason
6. Show eligibility breakdown (segment, threshold, exclusion, geo)

Key demo scenarios to verify:
- cust-001 (PLATINUM) sees camp-004 wine offer
- cust-002 (GOLD) does NOT see camp-004 — shows "Platinum members only"
- cust-003 cartTotal:35.00 sees "Spend $15 more for $10 off" for camp-003
- cust-003 cartTotal:67.50 sees $10 off from camp-003

### Requirement 3 — Demo Flow 3: Budget Exhaustion (End-to-End)

Wire fully against live Analytics + Campaign Services:

1. Campaign list shows camp-001 with 95.2% burn bar (red)
2. Status shows PAUSED
3. Alert banner: "Weekend Mega Sale has been paused — budget exhausted"
4. After exhaustion — cust-001 offer check no longer shows camp-001

### Requirement 4 — Polish

Final polish before demo:

1. Loading states on all async calls — no blank screens
2. All API errors surface to user — no silent failures
3. Page titles and navigation are clean
4. Mobile-responsive is NOT required — desktop only for demo
5. No console errors in browser DevTools

---

## Full Demo Checklist

### Flow 1 — Campaign Creation
- [ ] Create `Summer Drinks Promo` → publishes successfully → appears as ACTIVE
- [ ] Create conflicting campaign on upc-water-24pk → blocked with UPC_OVERLAP
- [ ] Create campaign with no funding → blocked with NO_FUNDING_SOURCE

### Flow 2 — Customer Offer Fetch
- [ ] cust-001 (PLATINUM): sees wine offer + general offers
- [ ] cust-002 (GOLD): no wine offer, SEGMENT_MISMATCH shown
- [ ] cust-003 cartTotal:35.00: THRESHOLD_NOT_MET for camp-003
- [ ] cust-003 cartTotal:67.50: $10 off from camp-003
- [ ] cust-010 (southwest): GEO_MISMATCH for camp-001

### Flow 3 — Budget Exhaustion
- [ ] camp-001: burn bar red at 95.2%
- [ ] camp-001: status PAUSED
- [ ] Alert banner visible and readable
- [ ] Post-pause offer check: camp-001 gone from cust-001's offers

---

## Sprint 4 ADR Topics

Record an ADR for:
- How you handle the case where multiple services are needed for one page render (campaign list + analytics burn) — parallel requests vs sequential
