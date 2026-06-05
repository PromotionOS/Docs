# Team 8 — Notification Service — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, notification-service-guide, adr-recorder, pr-reviewer, campaign-notification-contract

---

## Context

The Notification Service is a new Go/Gin service being built from scratch in Cohort 1 alongside the existing six services. No skeleton exists — the team bootstraps the project this sprint.

The domain problem: other services emit events (CampaignPaused, BudgetExhausted) but no one is listening on behalf of human actors. Merchandisers and Store Managers need to know when a campaign stops. This sprint wires email delivery to the two most urgent events and establishes the Notification Log so every sent notification is auditable.

Notification Preferences are loaded from Supabase — the team does not manage a separate preference store yet.

---

## Sprint 1 Goal

Stand up email notifications for CampaignPaused and BudgetExhausted events, persisting every sent notification to a Notification Log read from Supabase-stored Notification Preferences.

---

## Requirements

### Requirement 1 — Project Bootstrap

1. Initialise a Go/Gin project: `notification-service`
2. Connect to Supabase (PostgreSQL) — connection string from environment variable `DATABASE_URL`
3. Define the `notifications` table: `id`, `tenantId`, `eventType`, `recipientEmail`, `channel`, `sentAt`, `status` (`SENT` | `FAILED`), `payload` (jsonb)
4. Define the `notification_preferences` table: `id`, `tenantId`, `role`, `eventType`, `channel`, `enabled`
5. Expose a health endpoint: `GET /health` → `{ status: "ok" }`

### Requirement 2 — Subscribe to CampaignPaused

1. Subscribe to the `campaign.paused` topic (Supabase Realtime or message bus — record ADR)
2. On receiving `CampaignPaused`: extract `tenantId`, `campaignId`, `campaignName`, `pausedAt`
3. Load Notification Preferences for `role: MX_TEAM` and `eventType: CAMPAIGN_PAUSED` from Supabase
4. For each enabled preference with `channel: EMAIL` — send an email via SMTP or configured provider
5. Write one row to the `notifications` log per recipient regardless of send outcome
6. If send fails — log `status: FAILED` and do not retry in this sprint

### Requirement 3 — Subscribe to BudgetExhausted

1. Subscribe to the `budget.exhausted` topic
2. On receiving `BudgetExhausted`: extract `tenantId`, `campaignId`, `campaignName`, `exhaustedAt`, `totalAmount`, `burnedAmount`
3. Load Notification Preferences for `role: MX_TEAM` and `eventType: BUDGET_EXHAUSTED`
4. Send email to each enabled MX_TEAM recipient
5. Log every attempt to the `notifications` table

### Requirement 4 — Notification Log API

1. `GET /notifications` — return paginated list: `page`, `limit` query params, default limit 20
2. Filter by `tenantId` (required), optional `eventType`, optional `status`
3. Response shape: `{ data: [NotificationLog], total: int, page: int }`
4. Each log entry: `id`, `eventType`, `recipientEmail`, `channel`, `sentAt`, `status`
5. No delete or update endpoints — the Notification Log is immutable

---

## Domain Rules

From ubiquitous-language.md:
- Notification Preference is a per-role configuration: roles are `MX_TEAM`, `STORE_MANAGER`, `VENDOR_CONTACT`
- Delivery Channel values: `EMAIL`, `WEBSOCKET`, `WEBHOOK` — this sprint only EMAIL
- Domain Events carry `tenantId`, `occurredAt`, `schemaVersion` — always present
- Events are past tense: CampaignPaused, BudgetExhausted — use these exact names in log records
- Budget Burn threshold is >= 95.0% — BudgetExhausted is emitted by the Campaign Service, Notification Service only consumes it

---

## Contracts Involved

- `contract-campaign-notification.md` — CampaignPaused and BudgetExhausted event shapes consumed by this service. Read field mappings before writing your spec.
- No contracts published by this service yet — the Notification Log API is internal this sprint.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 32 | CampaignPaused received for camp-001, MX_TEAM has EMAIL preference enabled | Email sent, notification log row with status SENT |
| 33 | BudgetExhausted received for camp-001, MX_TEAM has EMAIL preference enabled | Email sent, notification log row with status SENT |
| 34 | CampaignPaused received, SMTP fails | Notification log row with status FAILED, no panic/crash |
| 35 | GET /notifications?tenantId=t-001 | Returns paginated log for that tenant only |
| 36 | GET /notifications?tenantId=t-001&eventType=CAMPAIGN_PAUSED | Returns only CampaignPaused entries |

---

## Sprint 1 ADR Topics

Record an ADR for:
- Event transport choice — Supabase Realtime vs dedicated message bus (Kafka / RabbitMQ / Redis Streams): tradeoffs for a 2-hour sprint vs production readiness
- Email provider choice — SMTP vs transactional email API (SendGrid / Resend): why and what the interface abstraction looks like to keep the provider swappable
