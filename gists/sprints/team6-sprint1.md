# Team 6 — Frontend Dashboard — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

The Frontend Dashboard is the face of PromotionOS. Sprint 1 focuses on the MX Campaign List — the first thing a merchandiser sees when they open the dashboard. Get this right and every sprint adds to it naturally.

---

## Sprint 1 Focus

**Campaign list. One component. Full SDD process.**

Note: Backend services are being built in parallel this sprint. Build against the API contract — use mock data where Campaign Service is not yet live. The contract is the agreement — you build to it, not to the running service.

---

## Requirements

### Requirement 1 — Campaign List

Implement `CampaignList` component on `/mx`:

1. Fetch from `GET /campaigns?tenantId=tenant-kroger-001`
2. Display table with columns:
   - Name
   - Status badge (ACTIVE green, PAUSED red, SCHEDULED blue, DRAFT grey)
   - Offer Type
   - Budget Burn % as a progress bar
   - Date Range
3. Budget burn bar colours:
   - 0-79% → green
   - 80-94% → amber
   - 95%+ → red
4. Loading state while fetching
5. Empty state if no campaigns
6. Error state if API returns error — show message, not blank screen

### Requirement 2 — Campaign Form (Scaffold Only)

Scaffold `CampaignForm` component — wire the UI but `POST /campaigns` call returns mock success:

1. Fields: name, offer type (dropdown), offer value, date range
2. Funding: vendorId, vendor share %, Kroger share % (auto-calculated)
3. Submit button — mock success for now, real API wired in Sprint 2
4. Validation: vendorShare + krogerShare must equal 100 (client-side)

---

## Mock Data for Sprint 1

While Campaign Service is being built — use this mock for development:

```json
[
  { "id": "camp-001", "name": "Weekend Mega Sale", "status": "ACTIVE", "offerType": "PCT_OFF", "budgetBurnPercent": 95.2, "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" } },
  { "id": "camp-002", "name": "Buy 2 Get 1 Free — Cereal", "status": "ACTIVE", "offerType": "BOGO", "budgetBurnPercent": 40.0, "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-15" } },
  { "id": "camp-006", "name": "Pharmacy — Vitamin Bundle", "status": "PAUSED", "offerType": "PCT_OFF", "budgetBurnPercent": 100.0, "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" } },
  { "id": "camp-007", "name": "Back to School", "status": "SCHEDULED", "offerType": "PCT_OFF", "budgetBurnPercent": 0.0, "dateRange": { "startDate": "2026-08-01", "endDate": "2026-08-31" } }
]
```

Switch to live API call once Campaign Service is deployed.

---

## Validation Checklist This Sprint

- [ ] Campaign list renders with correct status badges
- [ ] camp-001 burn bar shows red at 95.2%
- [ ] camp-002 burn bar shows green at 40.0%
- [ ] camp-006 shows PAUSED in red
- [ ] camp-007 shows SCHEDULED in blue
- [ ] Loading state appears before data loads
- [ ] Error state appears when API returns 500
- [ ] Campaign form renders with all fields and client-side validation

---

## Sprint 1 ADR Topics

Record an ADR for:
- Mock data strategy — hardcoded JSON vs mock API server — why you chose your approach
