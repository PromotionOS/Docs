# Team 6 — Frontend Dashboard — Sprint 2

> Type: Build via SDD — wiring to live Eligibility + Analytics services
> Duration: 2 hours
> Skills: sdd-spec-writer, frontend-guide, adr-recorder, pr-reviewer

---

## Context

Sprint 1 wired the MX Campaign List to live data. Sprint 2 adds the CX Offer Check (wired to the now-live Eligibility Service from Team 2 Sprint 1) and the Analytics Dashboard (wired to the live Analytics Service).

Both services are live by start of Sprint 2. Real data. Real APIs.

---

## Sprint 2 Goal

By end of sprint — CX View shows real eligible offers from live Eligibility Service. Analytics Dashboard shows real burn and lift data. Real-time polling detects campaign status changes.

---

## Requirements

### Requirement 1 — CX View (wired to live Eligibility Service)

Implement `CXView` page at `/cx`:

1. Customer selector dropdown — load these from constants (hardcoded for session):
   - Alice Johnson (PLATINUM, division-midwest)
   - Bob Martinez (GOLD, division-southeast)
   - Carol White (SILVER, division-midwest)
   - David Kim (BASIC, division-southeast)
   - Karen Adams (PLATINUM, multi-segment, division-midwest)
   - Jack Wilson (BASIC, division-southwest — geo miss)

2. Cart total input (number field, default 0)

3. "Check Offers" button → calls `GET /offers?customerId=X&tenantId=Y&cartTotal=Z` on **live Eligibility Service**

4. Display eligible offers as cards:
   - Campaign name
   - Offer type + discount value
   - Expiry date

5. Display ineligible offers with human-readable reason:
   - `SEGMENT_MISMATCH` → "This offer is for Platinum members only"
   - `THRESHOLD_NOT_MET` → "Add $X more to your cart to unlock this offer" (calculate the gap)
   - `GEO_MISMATCH` → "This offer is not available in your region"
   - `CAMPAIGN_INACTIVE` → filter out entirely — do not show

6. Redeem button next to each eligible offer → calls `POST /redeem` on live Redemption Service
   - idempotencyKey: `browser-${Date.now()}-${Math.random().toString(36).slice(2)}`
   - On success → "Redeemed! $X discount applied"
   - On 409 duplicate → "Already redeemed"

### Requirement 2 — Analytics Dashboard (wired to live Analytics Service)

Implement `AnalyticsDashboard` page at `/analytics`:

1. Campaign Performance Table — per campaign:
   - Name
   - Lift % (green if positive, red if negative, grey if zero)
   - ROI (`1.64x` or `N/A — Kroger funded` if null)
   - Budget Burn % (progress bar, same colour rules)
   - Redemption Count

2. Burn Chart — horizontal bar chart (recharts):
   - One bar per active campaign
   - Sorted by burn % descending
   - Colours match burn bar rules

3. Data from `GET /analytics/campaigns/:id/report` for each campaign

### Requirement 3 — Real-Time Campaign Status Polling

Enhance `CampaignList`:

1. Poll `GET /analytics/campaigns/:id/burn` every 10 seconds for each ACTIVE campaign
2. When `budgetExhaustedEmitted: true` returned:
   - Update status badge to PAUSED (red)
   - Show alert banner: "[Campaign Name] has been paused — budget exhausted"
   - Alert is dismissible
3. Poll must not show loading spinner on subsequent fetches — only initial load

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 3 | Alice Johnson (PLATINUM) check offers | Wine promo from camp-004 eligible |
| 2 | Bob Martinez (GOLD) check offers | Wine promo shows SEGMENT_MISMATCH |
| 5 | Carol White + cartTotal 35.00 | camp-003 shows THRESHOLD_NOT_MET |
| 6 | Carol White + cartTotal 67.50 | camp-003 eligible, $10 discount |
| 14 | Jack Wilson check offers | camp-001 shows GEO_MISMATCH |
| — | Analytics dashboard loads | camp-001 shows burn 95.2%, lift 65% |
| — | Analytics dashboard loads | camp-003 shows ROI: N/A — Kroger funded |
| — | camp-001 polled | Status shows PAUSED, alert banner visible |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Polling interval — why 10 seconds vs WebSocket vs SSE for this session's constraints
- THRESHOLD_NOT_MET gap calculation — how you compute how much more the customer needs to spend
