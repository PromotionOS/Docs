# Team 8 — Notification Service — Sprint 4

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, notification-service-guide, adr-recorder, pr-reviewer, campaign-notification-contract, vendor-notification-contract

---

## Context

Sprints 1–3 delivered: email for CampaignPaused and BudgetExhausted, WebSocket hub with Store Manager routing, FundingApproved/FundingRejected notifications, ClaimSubmitted webhook delivery, and the Notification History API.

This final sprint completes coverage across all 10 event types, enforces per-role Notification Preference gates so no notification is sent to a role that has not opted in, and delivers the cohort demo scenario: a budget exhaustion triggers a WebSocket alert in the browser in under 1 second with no polling anywhere in the system.

---

## Sprint 4 Goal

Handle all 10 domain event types, enforce per-role Notification Preferences on every delivery path, and demonstrate end-to-end: budget exhausts → WebSocket alert fires in under 1 second.

---

## Requirements

### Requirement 1 — Complete Event Coverage

Wire subscriptions and delivery logic for the remaining event types not covered in Sprints 1–3:

1. `CampaignPublished` → notify MX_TEAM (EMAIL + WEBSOCKET)
2. `CampaignExpired` → notify MX_TEAM and STORE_MANAGER (EMAIL)
3. `OfferRedeemed` → no notification by default; honour preferences if VENDOR_CONTACT has opted in (WEBSOCKET only — high volume, no email)
4. `SegmentUpdated` → notify MX_TEAM (EMAIL) when a Segment that affects an ACTIVE Campaign changes
5. `BudgetExhausted` → already handled in Sprint 1; add STORE_MANAGER routing consistent with CampaignPaused pattern from Sprint 2
6. Each event type must be logged to the `notifications` table with correct `eventType` value

### Requirement 2 — Notification Preference Enforcement

1. Before every delivery attempt — load the Notification Preference for `{ tenantId, role, eventType, channel }`
2. If `enabled: false` — skip delivery silently; do not log a `FAILED` entry (it was not attempted)
3. If no preference row exists for a role + eventType combination — default to `enabled: false` (opt-in model)
4. MX_TEAM, STORE_MANAGER, and VENDOR_CONTACT must be evaluated independently — a preference for MX_TEAM does not imply the same for STORE_MANAGER
5. Add `GET /notification-preferences` — returns all preferences for a `tenantId`; supports `role` and `eventType` filter params
6. Add `PUT /notification-preferences/:id` — updates `enabled` flag only; channel and role are immutable after creation

### Requirement 3 — Sub-1-Second WebSocket Demo

1. Measure and confirm end-to-end latency from BudgetExhausted event publication to WebSocket message receipt at a connected browser client is under 1 second under local demo conditions
2. Instrument the delivery path: log `enqueuedAt` (when event received), `processedAt` (when preference loaded), `sentAt` (when WebSocket push completed)
3. Expose `GET /notifications/:id/timings` → returns `{ enqueuedAt, processedAt, sentAt, latencyMs }` for any notification log entry
4. This endpoint is used during the demo to show the latency measurement on screen

### Requirement 4 — Integration Smoke Test

Document (in the ADR) the manual integration test sequence that confirms all services connect correctly:

1. Trigger camp-001 budget exhaustion → BudgetExhausted fires → MX_TEAM WebSocket push received → Store Managers of division-midwest notified by email
2. Submit Funding Proposal for camp-013 → FundingApproved fires → Campaign Service unblocks camp-013 → MX_TEAM and VENDOR_CONTACT notified
3. ClaimSubmitted for a Redemption → Vendor webhook receives signed payload
4. Confirm no notification fires for a role with `enabled: false` preference

---

## Domain Rules

From ubiquitous-language.md:
- Budget Burn threshold: `budgetBurnPercent >= 95.0` triggers BudgetExhausted — this is the event the demo centres on
- Notification Preference is per-role — three distinct roles: `MX_TEAM`, `STORE_MANAGER`, `VENDOR_CONTACT`
- Delivery Channel: `EMAIL`, `WEBSOCKET`, `WEBHOOK` — OfferRedeemed uses WEBSOCKET only due to volume
- Domain Events are named in past tense and carry `tenantId`, `occurredAt`, `schemaVersion`
- All Tenant data is isolated — a WebSocket push must never cross Tenant boundaries
- Division `division-all` matches any Store — when a Campaign targets all divisions, all Store Managers are in scope

---

## Contracts Involved

- `contract-campaign-notification.md` — CampaignPublished, CampaignExpired events; confirm all 10 event types are documented
- `contract-vendor-notification.md` — FundingApproved, FundingRejected final payload review
- Notification Preferences API is a new outbound contract — share with any team whose frontend consumes it

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 32 | BudgetExhausted for camp-001, MX_TEAM WebSocket connected | WebSocket push received; latency logged under 1000ms |
| 33 | BudgetExhausted for camp-001 (division-midwest), STORE_MANAGER EMAIL enabled | Store Managers of midwest Stores receive email |
| 36 | FundingApproved for camp-013, VENDOR_CONTACT EMAIL enabled | Vendor contact email delivered |
| 38 | MX_TEAM has CAMPAIGN_PUBLISHED EMAIL preference enabled:false | No email sent; no FAILED log entry |
| 39 | OfferRedeemed, VENDOR_CONTACT has WEBSOCKET preference enabled | WebSocket push sent; no email sent |
| 40 | GET /notifications/:id/timings for a BudgetExhausted notification | Returns enqueuedAt, processedAt, sentAt, latencyMs |

---

## Sprint 4 ADR Topics

Record an ADR for:
- Opt-in vs opt-out default for Notification Preferences — why opt-in (enabled: false by default) was chosen and what the operational risk is if preferences are not seeded
- Latency budget allocation — where the sub-1-second budget is spent across event consumption, preference lookup, and WebSocket fan-out; where the bottleneck is most likely to appear at scale
- OfferRedeemed volume throttling — why email was excluded and what the WEBSOCKET rate-limiting strategy should be in a production system
