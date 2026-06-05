---
name: notification-service-guide
description: Codebase guide for Notification Service — folder structure, pre-built vs stub, domain model, 10 event consumers, WebSocket hub, routing pattern, implementation plan
metadata:
  type: reference
---

# Notification Service — Codebase Guide

## Quick Facts
- Language: Go 1.21
- Framework: Gin + Gorilla WebSocket
- Port: 8086 (defaults to `$PORT`, skeleton default 8085 — set `PORT=8086` to avoid clash with Analytics)
- DB Schema: `notification` (PostgreSQL via Goose migrations)
- Status: Pure skeleton — all application service methods return `ErrNotImplemented`

## Folder Structure

```
notification-service/
├── cmd/
│   └── main.go                                    ← starts all 10 consumers + Gin
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── notification.go                    ← Notification struct (Aggregate Root)
│   │   │   └── notification_preference.go         ← NotificationPreference + RecipientRole + DeliveryChannel
│   │   ├── service/
│   │   │   ├── notification_router.go             ← interface — Route(tenantID, eventType) []RoutingTarget
│   │   │   └── delivery_service.go                ← interface — Send(*Notification) error
│   │   └── repository/
│   │       ├── notification_repository.go         ← interface — Save, FindByTenant, FindByRecipient
│   │       └── notification_preference_repository.go ← interface — FindByTenantAndRole, FindByTenantAndEventType
│   ├── application/
│   │   └── notification_service.go                ← ALL STUBS: HandleCampaignPaused, HandleBudgetExhausted,
│   │                                                   HandleClaimSubmitted, HandleFundingApproved,
│   │                                                   BroadcastWebSocket, GetHistory
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   ├── notification_repo_impl.go
│   │   │   └── notification_preference_repo_impl.go
│   │   ├── websocket/
│   │   │   └── hub.go                             ← PRE-BUILT: Register, Unregister, Broadcast
│   │   └── event/
│   │       ├── campaign_published_consumer.go     ← wired, handler TODO Sprint 3
│   │       ├── campaign_paused_consumer.go        ← wired, handler TODO Sprint 2
│   │       ├── campaign_activated_consumer.go     ← wired, handler TODO Sprint 3
│   │       ├── campaign_drafted_consumer.go       ← wired, handler TODO Sprint 4
│   │       ├── budget_exhausted_consumer.go       ← wired, handler TODO Sprint 2
│   │       ├── budget_updated_consumer.go         ← wired, handler TODO Sprint 4
│   │       ├── offer_redeemed_consumer.go         ← wired, handler TODO Sprint 4
│   │       ├── claim_submitted_consumer.go        ← wired, handler TODO Sprint 3
│   │       ├── funding_approved_consumer.go       ← wired, handler TODO Sprint 3
│   │       └── funding_rejected_consumer.go       ← wired, handler TODO Sprint 3
│   └── api/
│       └── notification_handler.go               ← GetNotifications (stub), WebSocketUpgrade (partial)
├── db/
│   └── migrations/
│       ├── 001_notification_schema.sql            ← notifications, notification_preferences, websocket_connections
│       └── 002_seed_test_data.sql                 ← seeds preferences for tenant-kroger-001
├── scripts/
│   ├── test.sh
│   └── deploy.sh
├── Dockerfile
└── go.mod
```

## What's Pre-built

`infrastructure/websocket/hub.go` — fully implemented. `Register(tenantID, conn)`, `Unregister(tenantID, conn)`, `Broadcast(tenantID, []byte)`. Uses `sync.RWMutex`. Connections are scoped per tenant. You call `hub.Broadcast(tenantID, payload)` inside your WebSocket delivery adapter.

`api/notification_handler.go` — `WebSocketUpgrade` is partially implemented: upgrades the connection, registers with hub, and runs a read-discard loop to keep the connection alive. Wire `hub.Unregister` on disconnect — this is already present in the `defer` block.

DB schema in `001_notification_schema.sql` — tables exist and are ready: `notification.notifications`, `notification.notification_preferences`, `notification.websocket_connections`.

Seed data in `002_seed_test_data.sql` — preferences for `tenant-kroger-001` are seeded:
- `MX_TEAM` receives `CampaignPaused`, `BudgetExhausted`, `FundingApproved`, `FundingRejected` via `EMAIL + WEBSOCKET`
- `STORE_MANAGER` receives `CampaignPaused` via `EMAIL`
- `VENDOR_CONTACT` receives `ClaimSubmitted`, `FundingApproved`, `FundingRejected` via `EMAIL`

All 10 consumers in `infrastructure/event/` are wired in `main.go` and running. Their handlers are stubs — they log the error until you implement.

## What You Implement (by sprint)

**Sprint 1**
- Wire repository implementations into `NotificationService` struct — inject `NotificationRepository` and `NotificationPreferenceRepository`
- `NotificationService.GetHistory(tenantID, recipientID)` — query `NotificationRepository.FindByRecipient()`
- `GET /notifications` handler — wire into `GetHistory`

**Sprint 2**
- Implement `NotificationRouter` — load preferences from `NotificationPreferenceRepository.FindByTenantAndEventType(tenantID, eventType)`, return `[]RoutingTarget` (one per recipient+channel pair)
- Implement `DeliveryService` — route to email adapter, WebSocket adapter, or webhook adapter based on `DeliveryChannel`
- `NotificationService.HandleCampaignPaused(event)` — route to `MX_TEAM` (EMAIL+WEBSOCKET) + `STORE_MANAGER` (EMAIL); store `Notification` records; call `hub.Broadcast()` for WebSocket targets
- `NotificationService.HandleBudgetExhausted(event)` — route to `MX_TEAM` (EMAIL+WEBSOCKET)
- `NotificationService.BroadcastWebSocket(tenantID, message)` — call `hub.Broadcast(tenantID, message)`
- `WS /ws` handler — wire hub registration (partially done; ensure `hub.Unregister` on disconnect)

**Sprint 3**
- `NotificationService.HandleClaimSubmitted(event)` — route to `VENDOR_CONTACT` (EMAIL); resolve vendor contact email via Catalog Service `GET /stores/:id/manager` or vendor profile
- `NotificationService.HandleFundingApproved(event)` — route to `MX_TEAM` (EMAIL+WEBSOCKET)
- `HandleFundingRejected(event)` — same routing as approved

**Sprint 4 (low priority)**
- `HandleCampaignPublished`, `HandleCampaignActivated`, `HandleCampaignDrafted`, `HandleOfferRedeemed`, `HandleBudgetUpdated`

## Domain Model

```go
// model/notification.go
Notification struct (Aggregate Root)
  ID:              string
  TenantID:        string
  EventType:       string              ← e.g. "CampaignPaused", "BudgetExhausted"
  DeliveryChannel: DeliveryChannel    ← EMAIL | WEBSOCKET | WEBHOOK
  RecipientID:     string
  RecipientRole:   RecipientRole      ← MX_TEAM | STORE_MANAGER | VENDOR_CONTACT
  Payload:         map[string]any     ← forwarded to the channel adapter
  Status:          NotificationStatus ← PENDING | DELIVERED | FAILED
  CreatedAt:       time.Time
  DeliveredAt:     *time.Time         ← nil until delivered

// model/notification_preference.go
NotificationPreference struct
  TenantID:  string
  Role:      RecipientRole
  EventType: string
  Channels:  []DeliveryChannel

RecipientRole — MX_TEAM | STORE_MANAGER | VENDOR_CONTACT
DeliveryChannel — EMAIL | WEBSOCKET | WEBHOOK
NotificationStatus — PENDING | DELIVERED | FAILED
```

## Publishing Events

This service publishes nothing. It is a pure consumer — it receives events and delivers notifications.

## Consuming Events

All 10 consumers follow the same pattern as `campaign_paused_consumer.go`:

```go
func (c *CampaignPausedConsumer) Subscribe(client *redis.Client) {
    pubsub := client.Subscribe(context.Background(), "promotionos.campaign.paused")
    go func() {
        for msg := range pubsub.Channel() {
            var event map[string]any
            if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
                log.Printf("unmarshal error: %v", err)
                continue
            }
            if err := c.service.HandleCampaignPaused(event); err != nil {
                log.Printf("handler error: %v", err)
            }
        }
    }()
}
```

| Consumer file | Channel subscribed | Sprints |
|---|---|---|
| `campaign_paused_consumer.go` | `promotionos.campaign.paused` | Sprint 2 |
| `budget_exhausted_consumer.go` | `promotionos.analytics.budget.exhausted` | Sprint 2 |
| `claim_submitted_consumer.go` | `promotionos.claim.submitted` | Sprint 3 |
| `funding_approved_consumer.go` | `promotionos.vendor.funding.approved` | Sprint 3 |
| `funding_rejected_consumer.go` | `promotionos.vendor.funding.rejected` | Sprint 3 |
| `campaign_published_consumer.go` | `promotionos.campaign.published` | Sprint 3 |
| `campaign_activated_consumer.go` | `promotionos.campaign.activated` | Sprint 3 |
| `campaign_drafted_consumer.go` | `promotionos.campaign.drafted` | Sprint 4 |
| `budget_updated_consumer.go` | `promotionos.campaign.budget.updated` | Sprint 4 |
| `offer_redeemed_consumer.go` | `promotionos.redemption.redeemed` | Sprint 4 |

## Key Interfaces You Must Implement

```go
// domain/service/notification_router.go
type NotificationRouter interface {
    Route(tenantID, eventType string) ([]RoutingTarget, error)
    // Query NotificationPreferenceRepository for matching preferences
    // Return one RoutingTarget per (recipient, channel) pair
    // For STORE_MANAGER: must call Catalog Service to resolve storeID → managerID
}

// RoutingTarget (value struct, not interface):
type RoutingTarget struct {
    RecipientID   string
    RecipientRole model.RecipientRole
    Channel       model.DeliveryChannel
}

// domain/service/delivery_service.go
type DeliveryService interface {
    Send(n *model.Notification) error
    // Switch on n.DeliveryChannel:
    // EMAIL    → log "sending email to {recipientID}" (stub for session; real SMTP out of scope)
    // WEBSOCKET → hub.Broadcast(n.TenantID, json.Marshal(n.Payload))
    // WEBHOOK  → HTTP POST to configured endpoint
}

// domain/repository/notification_repository.go
type NotificationRepository interface {
    Save(n *model.Notification) error
    FindByTenant(tenantID string) ([]*model.Notification, error)
    FindByRecipient(tenantID, recipientID string) ([]*model.Notification, error)
}

// domain/repository/notification_preference_repository.go
type NotificationPreferenceRepository interface {
    FindByTenantAndRole(tenantID string, role model.RecipientRole) ([]*model.NotificationPreference, error)
    FindByTenantAndEventType(tenantID, eventType string) ([]*model.NotificationPreference, error)
}
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/notification-db
export REDIS_URL=redis://localhost:6379
export PORT=8086
go run ./cmd/main.go
# Health: http://localhost:8086/health
# WebSocket: ws://localhost:8086/ws?tenantId=tenant-kroger-001
```

Run tests:
```bash
go test ./... -v -count=1
# or: ./scripts/test.sh
```

## Patterns In This Codebase

**One Notification record per (event, recipient, channel) pair.** A single `BudgetExhausted` event can produce three `Notification` records: one for MX_TEAM via EMAIL, one for MX_TEAM via WEBSOCKET, one for STORE_MANAGER via EMAIL. Each is stored and tracked independently. This gives a complete audit trail per delivery attempt.

**WebSocket Hub is infrastructure, not domain.** The `Hub` in `infrastructure/websocket/hub.go` is not a domain concept. It is a connection registry. Domain logic (routing, preferences) lives in `NotificationRouter`. The `DeliveryService` calls `hub.Broadcast()` as the final delivery step for WEBSOCKET-channel notifications. Never import Hub into domain or application layers.

**Routing is preference-driven, not hardcoded.** Do not write `if eventType == "BudgetExhausted" { sendTo(MX_TEAM) }` in code. Load preferences from `notification_preferences` table and route dynamically. This lets the facilitator change routing for test scenarios by modifying seed data without changing code.

## Do NOT Do This

**Do not own store manager data.** When routing `CampaignPaused` to `STORE_MANAGER`, you need the manager's contact. Call `GET /stores/:storeId/manager` on Catalog & Customer Service to resolve this. Do not add a `store_managers` table to the notification schema — Catalog & Customer Service owns that data.

**Do not broadcast to all tenants.** `hub.Broadcast(tenantID, message)` is scoped to a single tenant. Never call `Broadcast` without a `tenantID`. WebSocket connections are registered per tenant — `hub.clients[tenantID]` is the correct scoping. Broadcasting cross-tenant is a data isolation violation.

**Do not block the event consumer goroutine on delivery failures.** If email delivery fails (SMTP down), log the failure, update `Notification.Status = FAILED`, and `continue` to the next message. A blocking retry inside the consumer goroutine will stall all subsequent events on that channel. Retry logic belongs in a separate scheduled job (out of scope for the session).
