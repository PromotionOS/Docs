# Team 8 — Notification Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, notification-service-guide, adr-recorder, pr-reviewer, campaign-notification-contract, catalog-customer-contract

---

## Context

Sprint 1 delivered email notifications for CampaignPaused and BudgetExhausted and a queryable Notification Log. The system works but clients poll the Notification Log API every 10 seconds to show alerts in-app — this creates unnecessary load and latency.

This sprint introduces a WebSocket hub so the frontend switches to a persistent connection and receives real-time push alerts the moment an event fires. It also solves the Store Manager routing problem: when camp-001 (a midwest campaign) pauses, the Notification Service must notify the correct Store Managers — which requires resolving which stores belong to the midwest Division by calling Catalog & Customer Service.

---

## Sprint 2 Goal

Replace 10-second frontend polling with a WebSocket hub that pushes real-time in-app alerts, and route CampaignPaused notifications to Store Managers for the correct Division by calling Catalog Service.

---

## Requirements

### Requirement 1 — WebSocket Hub

1. Expose `GET /ws` — upgrades the HTTP connection to WebSocket using Gorilla WebSocket or stdlib `golang.org/x/net/websocket` (record ADR)
2. Clients authenticate with a `token` query parameter (pass-through — validate it exists, no full auth this sprint)
3. On upgrade — register the client connection in an in-memory hub keyed by `tenantId` and `role`
4. On disconnect — deregister cleanly; hub must not hold dead connections
5. Hub must support concurrent connections — use Go channels or sync primitives (record ADR choice)

### Requirement 2 — Push Alerts Over WebSocket

1. When CampaignPaused is received: after sending EMAIL to MX_TEAM, also push a WebSocket message to all connected clients with `role: MX_TEAM` and the matching `tenantId`
2. When BudgetExhausted is received: same — push to MX_TEAM WebSocket connections
3. WebSocket message shape: `{ eventType, campaignId, campaignName, occurredAt, message: string }`
4. If no WebSocket clients are connected — do not error; log at INFO level
5. Log every WebSocket push to the `notifications` table with `channel: WEBSOCKET`

### Requirement 3 — Store Manager Routing for CampaignPaused

1. When CampaignPaused is received — extract `divisionId` from the event payload
2. Call `GET /stores?divisionId={divisionId}` on Catalog & Customer Service to resolve the list of Stores in that Division
3. For each Store — look up the Store Manager's contact email from the response
4. Load Notification Preferences for `role: STORE_MANAGER` and `eventType: CAMPAIGN_PAUSED`
5. If `STORE_MANAGER` preference has `channel: EMAIL` enabled — send email to each resolved Store Manager
6. If Catalog & Customer Service is unavailable — log error, continue delivering to MX_TEAM, do not fail the entire notification
7. Specific scenario: when camp-001 (division-midwest) pauses — Store Managers of all midwest Stores receive the notification

### Requirement 4 — Remove Frontend Polling

1. Document (in the ADR) the polling endpoint that the frontend was calling and confirm it is no longer the primary path
2. The `GET /notifications` log endpoint remains — it is now for historical queries, not live alerts
3. Frontend connection lifecycle: connect on login, disconnect on logout, reconnect on drop with exponential backoff (document expected client behaviour in ADR — Notification Service does not own the frontend)

---

## Domain Rules

From ubiquitous-language.md:
- Store is owned by Catalog & Customer Service — Notification Service does not own Store data, it calls the Catalog to resolve Stores by Division
- Store Manager is the role that receives store-scoped notifications
- Division values: `division-all`, `division-midwest`, `division-southeast`, `division-west`, `division-northeast`, `division-southwest`
- Delivery Channel: `WEBSOCKET` is real-time in-app — distinct from EMAIL and WEBHOOK
- Notification Preference is per-role — Store Managers may have different channel preferences than MX_TEAM
- All data is isolated per Tenant — WebSocket hub must never route a message to a client from a different Tenant

---

## Contracts Involved

- `contract-campaign-notification.md` — CampaignPaused event must carry `divisionId`; confirm field is present or raise with Campaign Service team
- `contract-catalog-customer.md` — `GET /stores?divisionId=X` response shape: verify `storeId`, `name`, `divisionId`, `storeManagerEmail` fields before writing your spec
- No new outbound contracts published this sprint

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 33 | CampaignPaused for camp-001 (division-midwest), WebSocket client connected as MX_TEAM | WebSocket push received within 1 second |
| 34 | CampaignPaused for camp-001 (division-midwest), Store Manager preference EMAIL enabled | Emails sent to all midwest Store Managers resolved from Catalog Service |
| 35 | Catalog Service returns 503 during Store Manager lookup | MX_TEAM still notified; error logged; no crash |
| 36 | Two tenants both connected via WebSocket | Tenant A's CampaignPaused does not appear in Tenant B's connection |
| 37 | Frontend was polling GET /notifications every 10s | After sprint: frontend receives push; polling removed from client |

---

## Sprint 2 ADR Topics

Record an ADR for:
- WebSocket library choice — Gorilla WebSocket vs stdlib: concurrency model, ping/pong keepalive, graceful shutdown behaviour
- In-memory hub vs Redis pub/sub for multi-instance deployments: why in-memory is acceptable now and what the migration path is when the service scales horizontally
- Store Manager routing strategy — fan-out per Store vs batched Division lookup: latency and failure mode tradeoffs
