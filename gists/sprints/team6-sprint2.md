# Team 6 — Frontend Dashboard — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

Sprint 1 delivered MX campaign list, campaign form, and CX offer check. Sprint 2 adds the Analytics Dashboard and real-time campaign status polling — the foundation for the demo's most impactful moment (budget exhaustion auto-pause).

---

## Sprint 2 Goal

By end of sprint — Analytics Dashboard shows live burn and lift data, and the MX Campaign List polls for status changes every 10 seconds showing auto-pause in real time.

---

## Requirements

### Requirement 1 — Analytics Dashboard

Implement `AnalyticsDashboard` page at `/analytics`:

1. Campaign Performance Table — per campaign:
   - Name
   - Lift % (formatted: `+65.0%` green, `-5.0%` red, `0.0%` grey)
   - ROI (formatted: `1.64x` or `null` shown as `N/A — Kroger funded`)
   - Budget Burn % (progress bar with same colour rules as campaign list)
   - Redemption Count

2. Burn Chart — horizontal bar chart:
   - One bar per active campaign
   - Bar width = budgetBurnPercent
   - Colour: same rules as campaign list (green/amber/red)
   - Campaigns sorted by burn % descending

3. Data fetched from `GET /analytics/campaigns/:id/report` for each campaign
4. Refresh button — manually refresh all analytics data
5. Loading state while fetching

### Requirement 2 — Real-Time Campaign Status Polling

Enhance `CampaignList` component:

1. Poll `GET /campaigns?tenantId=tenant-kroger-001` every 10 seconds
2. When a campaign transitions from ACTIVE to PAUSED:
   - Update its status badge to red PAUSED
   - Update its budget burn bar to red
   - Show alert banner: "[Campaign Name] has been paused — budget exhausted"
3. Alert banner:
   - Appears at top of MX Dashboard
   - Shows campaign name and reason
   - Dismissible by clicking X
   - If multiple campaigns pause — show one alert per campaign
4. Polling must not show loading spinner — only the initial load shows spinner

### Requirement 3 — Redemption Flow (CX View)

Add redemption simulation to CX View:

1. After offer check — show "Redeem" button next to each eligible offer
2. On click — call `POST /redeem` with:
   - tenantId: `tenant-kroger-001`
   - customerId: selected customer
   - campaignId: from the offer
   - discountApplied: from the eligibility response
   - cartTotal: from the cart total input
   - idempotencyKey: auto-generated `browser-{timestamp}-{random}`
   - storeId: `store-demo-001`
   - division: customer's division
3. On success — show "Redeemed! Discount: $X applied"
4. On duplicate — show "This offer has already been redeemed"
5. On error — show the error reason

---

## API Calls to Implement

```javascript
// analyticsApi.js
export const getCampaignReport = (campaignId, tenantId) => { }
export const getCampaignBurn = (campaignId, tenantId) => { }

// redemptionApi.js
export const redeem = (redemptionRequest) => { }
```

---

## Validation Checklist

- [ ] Analytics Dashboard shows camp-001 with burn: 95.2%, lift: 65.0%, ROI: 1.64x
- [ ] Analytics Dashboard shows camp-003 with ROI: N/A — Kroger funded
- [ ] Analytics Dashboard shows camp-004 with lift: -5.0% in red
- [ ] Burn chart shows all active campaigns sorted by burn % descending
- [ ] Campaign list polls every 10 seconds (verify with DevTools network tab)
- [ ] Alert banner appears when campaign status changes to PAUSED
- [ ] CX View: redeem button calls POST /redeem correctly
- [ ] CX View: duplicate redemption shows correct message

---

## Sprint 2 ADR Topics

Record an ADR for:
- Polling interval choice — 10 seconds vs WebSocket vs SSE
- How you handle negative lift display (red vs grey vs absolute value)
