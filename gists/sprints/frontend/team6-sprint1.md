# Team 6 — Frontend Dashboard — Sprint 1

> Type: Build via SDD — wired to live APIs from day one
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

The Frontend Dashboard serves two audiences — the Merchandising (MX) team who create and manage campaigns, and the Customer Experience (CX) view that simulates offer eligibility.

**Campaign Service is pre-built and live.** You are wiring to real data from minute one. No mock data. No stubs. Real APIs.

Sprint 1 focuses on the MX Campaign List — wired to the live Campaign Service — and the campaign form scaffold.

---

## Sprint 1 Goal

By end of sprint — MX Dashboard shows real campaigns from the live Campaign Service. Campaign list renders with correct status badges, burn bars, and live data. Campaign form is scaffolded and submits to the real API.

---

## Requirements

### Requirement 1 — Campaign List (wired to live Campaign Service)

Implement `CampaignList` component on `/mx`:

1. Fetch from `GET /campaigns?tenantId=tenant-kroger-001` — **live Campaign Service URL**
2. Display table with columns: Name, Status badge, Offer Type, Budget Burn % bar, Date Range
3. Status badge colours: ACTIVE green, PAUSED red, SCHEDULED blue, DRAFT grey
4. Budget burn bar colours:
   - 0-79% → green
   - 80-94% → amber
   - 95%+ → red
5. Loading state while fetching
6. Error state if API returns error — show message, not blank screen
7. Empty state if no campaigns returned

**Validation against live data:**
- camp-001 (Weekend Mega Sale) → PAUSED badge, red burn bar at 95.2%
- camp-002 (BOGO Cereal) → ACTIVE badge, green burn bar at 40.0%
- camp-006 (Vitamin Bundle) → PAUSED badge, burn at 100%
- camp-007 (Back to School) → SCHEDULED badge, burn at 0%

### Requirement 2 — Campaign Form (wired to live Campaign Service)

Implement `CampaignForm` component:

1. Fields: name, offer type (dropdown: PCT_OFF, AMT_OFF, BOGO, THRESHOLD), offer value, UPC scope, date range, stack permission toggle
2. Funding section: vendor ID, vendor share %, Kroger share % (auto-calculated: 100 - vendorShare)
3. Client-side validation: vendorShare + krogerShare must equal 100
4. Submit → `POST /campaigns` → on success campaign appears in list
5. Publish button → `PUT /campaigns/:id/publish`
6. Error display:
   - `NO_FUNDING_SOURCE` → "Please add a funding source before publishing"
   - `UPC_OVERLAP` → "One or more UPCs conflict with an active campaign"

### Requirement 3 — Campaign Summary (wired to live Campaign Service)

Implement `GET /campaigns/:id/summary` call:

1. Click any campaign row → show summary panel
2. Display: name, status, offer type, vendorShare, krogerShare, totalAmount, burnedAmount
3. Used to verify Team 1's summary endpoint is live and correct

---

## Live API URLs

Set in `.env`:
```
VITE_CAMPAIGN_API_URL=https://<campaign-service-railway-url>
VITE_TENANT_ID=tenant-kroger-001
```

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| — | Campaign list loads | camp-001 shows PAUSED + 95.2% burn red bar |
| — | Campaign list loads | camp-007 shows SCHEDULED + blue badge |
| — | Create campaign + publish without funding | 400 NO_FUNDING_SOURCE shown |
| — | Create campaign + publish with UPC overlap | 409 UPC_OVERLAP shown |
| — | Click campaign row | Summary panel shows vendorShare correctly |

---

## SDD Process

```
Read requirements
    ↓
Write Spec (sdd-spec-writer skill) — 20 min
    ↓
Record ADR (adr-recorder skill) — 10 min
    ↓
Implement — 60 min
    ↓
Validate against live Campaign Service — 20 min
    ↓
PR → deploy to Vercel — 10 min
```

---

## Sprint 1 ADR Topics

Record an ADR for:
- Error handling strategy — how you surface API errors to the MX user without exposing raw error codes
