# Notification Service — Skeleton Build Template

> Language: Go 1.21
> Framework: Gin + Gorilla WebSocket
> DB: PostgreSQL via Gorm + Goose migrations
> Events: Redis Pub/Sub (consumes all 10 platform events; publishes nothing)
> Deploy: Railway
> Status: Pure skeleton — all business logic stubs throw NotImplemented

---

## Repo Structure

```
notification-service/
├── cmd/
│   └── main.go
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── notification.go
│   │   │   └── notification_preference.go
│   │   ├── service/
│   │   │   ├── notification_router.go
│   │   │   └── delivery_service.go
│   │   └── repository/
│   │       ├── notification_repository.go
│   │       └── notification_preference_repository.go
│   ├── application/
│   │   └── notification_service.go
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   ├── notification_repo_impl.go
│   │   │   └── notification_preference_repo_impl.go
│   │   ├── websocket/
│   │   │   └── hub.go
│   │   └── event/
│   │       ├── campaign_published_consumer.go
│   │       ├── campaign_paused_consumer.go
│   │       ├── campaign_activated_consumer.go
│   │       ├── campaign_drafted_consumer.go
│   │       ├── budget_exhausted_consumer.go
│   │       ├── budget_updated_consumer.go
│   │       ├── offer_redeemed_consumer.go
│   │       ├── claim_submitted_consumer.go
│   │       ├── funding_approved_consumer.go
│   │       └── funding_rejected_consumer.go
│   └── api/
│       └── notification_handler.go
├── db/
│   └── migrations/
│       ├── 001_notification_schema.sql
│       └── 002_seed_test_data.sql
├── scripts/
│   ├── test.sh
│   └── deploy.sh
├── Dockerfile
├── go.mod
└── .github/
    └── workflows/
        └── test.yml
```

---

## go.mod

```go
module github.com/promotionos/notification-service

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/gorilla/websocket v1.5.1
    github.com/google/uuid v1.4.0
    github.com/redis/go-redis/v9 v9.3.0
    gorm.io/gorm v1.25.5
    gorm.io/driver/postgres v1.5.4
    github.com/pressly/goose/v3 v3.16.0
)
```

---

## Domain Model

### model/notification.go

```go
package model

import "time"

// DeliveryChannel represents the mechanism through which a notification is sent.
type DeliveryChannel string

const (
    ChannelEmail     DeliveryChannel = "EMAIL"
    ChannelWebSocket DeliveryChannel = "WEBSOCKET"
    ChannelWebhook   DeliveryChannel = "WEBHOOK"
)

// NotificationStatus tracks the delivery lifecycle of a single Notification.
type NotificationStatus string

const (
    StatusPending   NotificationStatus = "PENDING"
    StatusDelivered NotificationStatus = "DELIVERED"
    StatusFailed    NotificationStatus = "FAILED"
)

// Notification is the aggregate root for this bounded context.
// One logical event can produce multiple Notification records —
// one per (recipient, channel) pair.
type Notification struct {
    ID              string
    TenantID        string
    EventType       string             // e.g. "CampaignPaused", "BudgetExhausted"
    DeliveryChannel DeliveryChannel
    RecipientID     string             // userId or external contact id
    RecipientRole   RecipientRole
    Payload         map[string]any     // JSON envelope forwarded to channel adapter
    Status          NotificationStatus
    CreatedAt       time.Time
    DeliveredAt     *time.Time         // pointer — nil until delivered
}
```

### model/notification_preference.go

```go
package model

// RecipientRole determines which users receive which event types.
type RecipientRole string

const (
    RoleMXTeam         RecipientRole = "MX_TEAM"
    RoleStoreManager   RecipientRole = "STORE_MANAGER"
    RoleVendorContact  RecipientRole = "VENDOR_CONTACT"
)

// NotificationPreference is a tenant-scoped routing rule:
// "for this role + event type, use these delivery channels."
type NotificationPreference struct {
    TenantID  string
    Role      RecipientRole
    EventType string
    Channels  []DeliveryChannel
}
```

---

## Repository Interfaces

### repository/notification_repository.go

```go
package repository

import "github.com/promotionos/notification-service/internal/domain/model"

type NotificationRepository interface {
    Save(n *model.Notification) error
    FindByTenant(tenantID string) ([]*model.Notification, error)
    FindByRecipient(tenantID, recipientID string) ([]*model.Notification, error)
}
```

### repository/notification_preference_repository.go

```go
package repository

import "github.com/promotionos/notification-service/internal/domain/model"

type NotificationPreferenceRepository interface {
    FindByTenantAndRole(tenantID string, role model.RecipientRole) ([]*model.NotificationPreference, error)
    FindByTenantAndEventType(tenantID, eventType string) ([]*model.NotificationPreference, error)
}
```

---

## Domain Service Interfaces

### service/notification_router.go

```go
package service

import "github.com/promotionos/notification-service/internal/domain/model"

// RoutingTarget describes a single (recipient, channel) pair resolved for a given event.
type RoutingTarget struct {
    RecipientID   string
    RecipientRole model.RecipientRole
    Channel       model.DeliveryChannel
}

// NotificationRouter resolves who should receive a notification and via which channel.
// Given an event type and tenantId, returns all (recipient, channel) pairs to notify.
type NotificationRouter interface {
    Route(tenantID, eventType string) ([]RoutingTarget, error)
}
```

### service/delivery_service.go

```go
package service

import "github.com/promotionos/notification-service/internal/domain/model"

// DeliveryService dispatches a Notification to the correct channel adapter.
// Implementations: EmailAdapter, WebSocketAdapter, WebhookAdapter.
type DeliveryService interface {
    Send(n *model.Notification) error
}
```

---

## Infrastructure — WebSocketHub

```go
// infrastructure/websocket/hub.go
// WebSocketHub is NOT a domain component — it is infrastructure.
// It manages active WebSocket connections keyed by tenantId.
package websocket

import (
    "sync"
    gws "github.com/gorilla/websocket"
)

// Hub holds all active WebSocket connections for the service.
// Connections are scoped per tenant so broadcasts stay isolated.
type Hub struct {
    mu      sync.RWMutex
    clients map[string][]*gws.Conn // tenantID → connections
}

func NewHub() *Hub {
    return &Hub{clients: make(map[string][]*gws.Conn)}
}

func (h *Hub) Register(tenantID string, conn *gws.Conn) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.clients[tenantID] = append(h.clients[tenantID], conn)
}

func (h *Hub) Unregister(tenantID string, conn *gws.Conn) {
    h.mu.Lock()
    defer h.mu.Unlock()
    conns := h.clients[tenantID]
    for i, c := range conns {
        if c == conn {
            h.clients[tenantID] = append(conns[:i], conns[i+1:]...)
            return
        }
    }
}

// Broadcast sends a raw JSON message to all connected clients for a tenant.
func (h *Hub) Broadcast(tenantID string, message []byte) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    for _, conn := range h.clients[tenantID] {
        _ = conn.WriteMessage(gws.TextMessage, message)
    }
}
```

---

## Application Service

```go
// application/notification_service.go
package application

import "errors"

var ErrNotImplemented = errors.New("not implemented")

// NotificationService is the application-layer facade.
// All methods are stubs — teams implement during sprints.
type NotificationService struct {
    // repo       repository.NotificationRepository           // inject Sprint 1
    // prefRepo   repository.NotificationPreferenceRepository // inject Sprint 1
    // router     service.NotificationRouter                  // inject Sprint 2
    // delivery   service.DeliveryService                     // inject Sprint 2
    // hub        *websocket.Hub                              // inject Sprint 2
}

// HandleCampaignPaused notifies MX team + affected store managers.
// Routing: MX_TEAM via EMAIL + WEBSOCKET; STORE_MANAGER via EMAIL.
// TODO Team 6 Sprint 2
func (s *NotificationService) HandleCampaignPaused(event map[string]any) error {
    return ErrNotImplemented
}

// HandleBudgetExhausted notifies MX team that a campaign budget reached exhaustion threshold.
// Routing: MX_TEAM via EMAIL + WEBSOCKET.
// TODO Team 6 Sprint 2
func (s *NotificationService) HandleBudgetExhausted(event map[string]any) error {
    return ErrNotImplemented
}

// HandleClaimSubmitted notifies vendor contact of a new claim against their campaign funding.
// Routing: VENDOR_CONTACT via EMAIL.
// TODO Team 6 Sprint 3
func (s *NotificationService) HandleClaimSubmitted(event map[string]any) error {
    return ErrNotImplemented
}

// HandleFundingApproved notifies MX team that a vendor funding proposal was approved.
// Campaign will unblock and move to ACTIVE.
// Routing: MX_TEAM via EMAIL + WEBSOCKET.
// TODO Team 6 Sprint 3
func (s *NotificationService) HandleFundingApproved(event map[string]any) error {
    return ErrNotImplemented
}

// BroadcastWebSocket pushes a raw JSON message to all connected clients for a tenant.
// Used by WebSocket handler after a domain event is processed.
// TODO Team 6 Sprint 2
func (s *NotificationService) BroadcastWebSocket(tenantID string, message []byte) error {
    return ErrNotImplemented
}

// GetHistory returns notification history for a tenant, optionally filtered by recipient.
// TODO Team 6 Sprint 1
func (s *NotificationService) GetHistory(tenantID, recipientID string) ([]any, error) {
    return nil, ErrNotImplemented
}
```

---

## API Handler

```go
// api/notification_handler.go
package api

import (
    "net/http"
    gws "github.com/gorilla/websocket"
    "github.com/gin-gonic/gin"
    "github.com/promotionos/notification-service/internal/application"
    ws "github.com/promotionos/notification-service/internal/infrastructure/websocket"
)

var upgrader = gws.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true }, // TODO: restrict in prod
}

type NotificationHandler struct {
    service *application.NotificationService
    hub     *ws.Hub
}

func NewNotificationHandler(svc *application.NotificationService, hub *ws.Hub) *NotificationHandler {
    return &NotificationHandler{service: svc, hub: hub}
}

// GET /notifications?tenantId=X&recipientId=Y
// Returns notification history for a tenant. recipientId is optional.
// TODO Team 6 Sprint 1 — implement after NotificationRepository is wired
func (h *NotificationHandler) GetNotifications(c *gin.Context) {
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

// WS /ws?tenantId=X
// Upgrades the connection to WebSocket and registers with the Hub.
// Clients must provide tenantId as a query param.
// TODO Team 6 Sprint 2 — implement after Hub and BroadcastWebSocket are wired
func (h *NotificationHandler) WebSocketUpgrade(c *gin.Context) {
    tenantID := c.Query("tenantId")
    if tenantID == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "tenantId required"})
        return
    }
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return
    }
    h.hub.Register(tenantID, conn)
    defer func() {
        h.hub.Unregister(tenantID, conn)
        conn.Close()
    }()
    // Read loop — keep connection alive; discard inbound messages (server-push only)
    for {
        if _, _, err := conn.ReadMessage(); err != nil {
            break
        }
    }
}
```

---

## Event Consumers

All consumers follow the same pattern as Analytics Service. Each subscribes to one Redis channel,
deserialises the event envelope, and delegates to the application service. All delegates are stubs
and log the error until implemented.

```go
// infrastructure/event/campaign_paused_consumer.go
package event

import (
    "context"
    "encoding/json"
    "log"
    "github.com/redis/go-redis/v9"
    "github.com/promotionos/notification-service/internal/application"
)

type CampaignPausedConsumer struct {
    service *application.NotificationService
}

func NewCampaignPausedConsumer(svc *application.NotificationService) *CampaignPausedConsumer {
    return &CampaignPausedConsumer{service: svc}
}

func (c *CampaignPausedConsumer) Subscribe(client *redis.Client) {
    pubsub := client.Subscribe(context.Background(), "promotionos.campaign.paused")
    go func() {
        for msg := range pubsub.Channel() {
            var event map[string]any
            if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
                log.Printf("CampaignPausedConsumer: unmarshal error: %v", err)
                continue
            }
            if err := c.service.HandleCampaignPaused(event); err != nil {
                log.Printf("CampaignPausedConsumer: handler error: %v", err)
            }
        }
    }()
}

// Remaining consumers — same pattern, one file each:
//
// campaign_published_consumer.go   → channel: promotionos.campaign.published
//                                    handler: TODO Sprint 3
// campaign_activated_consumer.go   → channel: promotionos.campaign.activated
//                                    handler: TODO Sprint 3
// campaign_drafted_consumer.go     → channel: promotionos.campaign.drafted
//                                    handler: TODO Sprint 4 (low priority)
// budget_exhausted_consumer.go     → channel: promotionos.analytics.budget.exhausted
//                                    handler: HandleBudgetExhausted — Sprint 2
// budget_updated_consumer.go       → channel: promotionos.campaign.budget.updated
//                                    handler: TODO Sprint 4
// offer_redeemed_consumer.go       → channel: promotionos.redemption.redeemed
//                                    handler: TODO Sprint 4
// claim_submitted_consumer.go      → channel: promotionos.claim.submitted
//                                    handler: HandleClaimSubmitted — Sprint 3
// funding_approved_consumer.go     → channel: promotionos.vendor.funding.approved
//                                    handler: HandleFundingApproved — Sprint 3
// funding_rejected_consumer.go     → channel: promotionos.vendor.funding.rejected
//                                    handler: TODO Sprint 3
```

---

## DB Migrations

### db/migrations/001_notification_schema.sql

```sql
-- Goose Up
-- +goose Up

CREATE SCHEMA IF NOT EXISTS notification;

-- notifications: one row per (event, recipient, channel) delivery attempt
CREATE TABLE notification.notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           TEXT NOT NULL,
    event_type          TEXT NOT NULL,
    delivery_channel    TEXT NOT NULL CHECK (delivery_channel IN ('EMAIL','WEBSOCKET','WEBHOOK')),
    recipient_id        TEXT NOT NULL,
    recipient_role      TEXT NOT NULL CHECK (recipient_role IN ('MX_TEAM','STORE_MANAGER','VENDOR_CONTACT')),
    payload             JSONB NOT NULL DEFAULT '{}',
    status              TEXT NOT NULL CHECK (status IN ('PENDING','DELIVERED','FAILED')) DEFAULT 'PENDING',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    delivered_at        TIMESTAMPTZ
);

CREATE INDEX idx_notifications_tenant     ON notification.notifications (tenant_id);
CREATE INDEX idx_notifications_recipient  ON notification.notifications (tenant_id, recipient_id);
CREATE INDEX idx_notifications_status     ON notification.notifications (status);

-- notification_preferences: routing rules per (tenant, role, event_type)
CREATE TABLE notification.notification_preferences (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   TEXT NOT NULL,
    role        TEXT NOT NULL CHECK (role IN ('MX_TEAM','STORE_MANAGER','VENDOR_CONTACT')),
    event_type  TEXT NOT NULL,
    channels    TEXT[] NOT NULL,           -- e.g. {'EMAIL','WEBSOCKET'}
    UNIQUE (tenant_id, role, event_type)
);

-- websocket_connections: audit log of active/historical connections
-- Note: the in-process Hub (infrastructure/websocket/hub.go) is the live registry.
-- This table is for observability — e.g. "how many clients were connected during incident?"
CREATE TABLE notification.websocket_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       TEXT NOT NULL,
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    disconnected_at TIMESTAMPTZ,
    remote_addr     TEXT
);

CREATE INDEX idx_ws_connections_tenant ON notification.websocket_connections (tenant_id);

-- +goose Down
DROP TABLE IF EXISTS notification.websocket_connections;
DROP TABLE IF EXISTS notification.notification_preferences;
DROP TABLE IF EXISTS notification.notifications;
DROP SCHEMA IF EXISTS notification;
```

### db/migrations/002_seed_test_data.sql

```sql
-- +goose Up
-- Seed notification preferences for tenant-kroger-001

INSERT INTO notification.notification_preferences (tenant_id, role, event_type, channels) VALUES
    ('tenant-kroger-001', 'MX_TEAM',        'CampaignPaused',     ARRAY['EMAIL','WEBSOCKET']),
    ('tenant-kroger-001', 'MX_TEAM',        'BudgetExhausted',    ARRAY['EMAIL','WEBSOCKET']),
    ('tenant-kroger-001', 'MX_TEAM',        'FundingApproved',    ARRAY['EMAIL','WEBSOCKET']),
    ('tenant-kroger-001', 'MX_TEAM',        'FundingRejected',    ARRAY['EMAIL','WEBSOCKET']),
    ('tenant-kroger-001', 'STORE_MANAGER',  'CampaignPaused',     ARRAY['EMAIL']),
    ('tenant-kroger-001', 'VENDOR_CONTACT', 'ClaimSubmitted',     ARRAY['EMAIL']),
    ('tenant-kroger-001', 'VENDOR_CONTACT', 'FundingApproved',    ARRAY['EMAIL']),
    ('tenant-kroger-001', 'VENDOR_CONTACT', 'FundingRejected',    ARRAY['EMAIL']);

-- +goose Down
DELETE FROM notification.notification_preferences WHERE tenant_id = 'tenant-kroger-001';
```

---

## main.go

```go
package main

import (
    "log"
    "os"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/promotionos/notification-service/internal/api"
    "github.com/promotionos/notification-service/internal/application"
    infra_event "github.com/promotionos/notification-service/internal/infrastructure/event"
    "github.com/promotionos/notification-service/internal/infrastructure/websocket"
)

func main() {
    db, err := gorm.Open(postgres.Open(os.Getenv("DB_URL")), &gorm.Config{})
    if err != nil {
        log.Fatalf("failed to connect to database: %v", err)
    }
    _ = db // TODO: pass to repository impls Sprint 1

    opt, err := redis.ParseURL(os.Getenv("REDIS_URL"))
    if err != nil {
        log.Fatalf("failed to parse REDIS_URL: %v", err)
    }
    redisClient := redis.NewClient(opt)

    hub := websocket.NewHub()
    svc := &application.NotificationService{} // TODO: inject deps Sprint 1

    // Start event consumers
    infra_event.NewCampaignPausedConsumer(svc).Subscribe(redisClient)
    infra_event.NewBudgetExhaustedConsumer(svc).Subscribe(redisClient)
    infra_event.NewClaimSubmittedConsumer(svc).Subscribe(redisClient)
    infra_event.NewFundingApprovedConsumer(svc).Subscribe(redisClient)
    infra_event.NewFundingRejectedConsumer(svc).Subscribe(redisClient)
    infra_event.NewCampaignPublishedConsumer(svc).Subscribe(redisClient)
    infra_event.NewCampaignActivatedConsumer(svc).Subscribe(redisClient)
    infra_event.NewCampaignDraftedConsumer(svc).Subscribe(redisClient)
    infra_event.NewBudgetUpdatedConsumer(svc).Subscribe(redisClient)
    infra_event.NewOfferRedeemedConsumer(svc).Subscribe(redisClient)

    r := gin.Default()
    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok", "service": "notification-service"})
    })

    h := api.NewNotificationHandler(svc, hub)
    r.GET("/notifications", h.GetNotifications)
    r.GET("/ws", h.WebSocketUpgrade)

    port := os.Getenv("PORT")
    if port == "" {
        port = "8085"
    }
    r.Run(":" + port)
}
```

---

## Dockerfile

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o notification-service ./cmd/main.go

FROM alpine:3.18
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=builder /app/notification-service .
COPY --from=builder /app/db/migrations ./db/migrations
EXPOSE 8085
CMD ["./notification-service"]
```

---

## scripts/test.sh

```bash
#!/bin/bash
set -e
go test ./... -v -count=1
```

## scripts/deploy.sh

```bash
#!/bin/bash
set -e
go build -o notification-service ./cmd/main.go
./notification-service
```

---

## .github/workflows/test.yml

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: notification-db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Download dependencies
        run: go mod download

      - name: Run tests
        env:
          DB_URL: postgresql://postgres:postgres@localhost:5432/notification-db
          REDIS_URL: redis://localhost:6379
        run: ./scripts/test.sh
```

---

## README.md

```markdown
# Notification Service

Bounded context: Notification
Language: Go 1.21 / Gin
Status: Pure skeleton — all business logic stubs return ErrNotImplemented

## Documentation
- SDL: [link]
- Ubiquitous Language: [link]
- Architecture: [link]
- Contract (Notification → Catalog): contract-notification-catalog.md

## Sprint Requirements
Handed out by facilitator at sprint start.

## Local Development
\`\`\`bash
export DB_URL=postgresql://localhost:5432/notification-db
export REDIS_URL=redis://localhost:6379
export PORT=8085
go run ./cmd/main.go
\`\`\`
```
