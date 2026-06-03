# Team 6 — Frontend Dashboard
## Type: Build via SDD

---

## Context

The Frontend Dashboard has two views — one for the Merchandising (MX) team who create and manage campaigns, and one that simulates the Customer (CX) offer experience. The page scaffolds and API clients exist but contain no logic.

Your job is to wire the UI to the real backend APIs and make the 3 demo flows visible end-to-end.

**Your job:**
1. Read the skeleton using the frontend-guide skill
2. Write a spec using the sdd-spec-writer skill covering all three flows
3. Implement the UI logic
4. Record an ADR using the adr-recorder skill
5. Validate all 3 demo flows visually
6. Open a PR

---

## The 3 Demo Flows You Must Make Visible

### Flow 1 — Campaign Creation (MX View)
1. MX team fills in the campaign form (name, offer type, UPCs, date range)
2. MX team adds a funding source (vendorId, vendorShare, krogerShare)
3. MX team clicks Publish
4. Campaign appears in the active campaign list with status ACTIVE
5. If publish fails (no funding, UPC conflict) — show the error clearly

### Flow 2 — Customer Offer Fetch (CX View)
1. Select a customer from a dropdown (use test data customers)
2. Enter a cart total
3. Click "Check Offers"
4. Display the eligible offers returned — showing discount type, value, and reason if ineligible
5. Show when an offer is ineligible and why (SEGMENT_MISMATCH, THRESHOLD_NOT_MET, etc.)

### Flow 3 — Budget Exhaustion (MX View)
1. Campaign list shows real-time budget burn percentage per campaign
2. When burn hits 95% — campaign status changes to PAUSED automatically (via event)
3. A visible alert banner appears: "Campaign [name] has been paused — budget exhausted"
4. Paused campaign is visually distinguished in the list (greyed out, PAUSED badge)

---

## Pages to Implement

### MX Dashboard (`/mx`)
- **Campaign List** — table of all campaigns with: name, status, offer type, budget burn %, date range
  - Status badge: ACTIVE (green), PAUSED (red), SCHEDULED (blue), DRAFT (grey)
  - Budget burn progress bar — turns red at 90%
- **Campaign Form** — create new campaign
  - Fields: name, offer type (dropdown), offer value, UPC scope (multi-select), date range, stack permission toggle
  - Funding section: vendor ID, vendor share %, Kroger share % (must sum to 100)
  - Submit → calls Campaign Service → shows success or error
- **Alert Banner** — appears when CampaignPaused event received (poll every 10s)

### CX View (`/cx`)
- **Customer Selector** — dropdown of test customers (name + loyalty tier)
- **Cart Total Input** — number input
- **Check Offers Button** — calls Eligibility Service
- **Offer List** — cards showing:
  - Campaign name
  - Discount (e.g. "20% off" or "$10 off")
  - Eligible: yes/no
  - If no: reason (human readable, not error code)

### Analytics Dashboard (`/analytics`)
- **Campaign Performance Table** — per campaign: lift %, ROI, burn %, redemption count
- **Burn Chart** — bar chart of budget burn % across active campaigns
- **Lift Chart** — comparison of actual vs baseline sales per campaign

---

## API Calls You Must Wire

| Action | API Call |
|--------|---------|
| Load campaigns | GET /campaigns?tenantId=tenant-kroger-001 |
| Create campaign | POST /campaigns |
| Publish campaign | PUT /campaigns/:id/publish |
| Check customer offers | POST /eligibility/check |
| Load analytics | GET /analytics/campaigns/:id/report |
| Poll campaign status | GET /campaigns/:id (every 10s for paused status) |

---

## UI Rules

1. All API errors must be shown to the user — never swallow errors silently
2. Loading states must be shown on all async operations
3. The tenant ID `tenant-kroger-001` is hardcoded for the session
4. No authentication required — session is single-tenant
5. Budget burn progress bar colours: 0-79% green, 80-94% amber, 95%+ red

---

## Validation Checklist

- [ ] Flow 1: Can create a campaign and publish it — appears in list as ACTIVE
- [ ] Flow 1: Publish with no funding shows error `NO_FUNDING_SOURCE`
- [ ] Flow 2: cust-001 (PLATINUM) sees wine offer from camp-004
- [ ] Flow 2: cust-002 (GOLD) does NOT see wine offer from camp-004 — shows SEGMENT_MISMATCH
- [ ] Flow 2: cust-003 with cartTotal 35.00 does NOT qualify for camp-003 — shows THRESHOLD_NOT_MET
- [ ] Flow 3: camp-001 shows 95.2% burn — status PAUSED — alert banner visible

---

## Skills to Use

- `sdd-spec-writer` — write your spec covering all 3 flows before touching code
- `frontend-guide` — understand the scaffold and API client structure
- `adr-recorder` — record your polling vs websocket decision for real-time updates
- `pr-reviewer` — review your own PR against the spec before submitting
