# Event Store — Skeleton Build Template

> Language: Go 1.21
> Framework: Gin
> DB: PostgreSQL (append-only — no UPDATE, no DELETE)
> Events: Redis Pub/Sub (consumes ALL domain events)
> Deploy: Railway
> Status: Pre-built — pure infrastructure, no business logic for teams to implement

---

## Purpose

The Event Store is pure infrastructure. It consumes every domain event from every service and appends them to an append-only PostgreSQL table. No team builds the Event Store — it is pre-built by the facilitator.

It exists for two reasons:
1. **Audit trail** — every domain event ever emitted is recorded forever
2. **AI/ML readiness** — versioned, structured event history ready for future ML pipelines

Teams only interact with it by reading from it (for debugging). They never write to it directly.

---

## Repo Structure

```
event-store/
├── cmd/
│   └── main.go
├── internal/
│   ├── model/
│   │   └── domain_event.go
│   ├── repository/
│   │   └── event_store_repo.go
│   └── consumer/
│       └── all_events_consumer.go
├── db/
│   └── migrations/
│       └── 001_event_store_schema.sql
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

---

## model/domain_event.go

```go
package model

import "time"

type DomainEvent struct {
    ID            string    `gorm:"primaryKey"`
    TenantID      string    `gorm:"index"`
    Context       string    `gorm:"index"` // campaign, eligibility, redemption, analytics, catalog
    AggregateID   string    `gorm:"index"`
    AggregateType string
    EventType     string    `gorm:"index"`
    SchemaVersion int
    Payload       string    `gorm:"type:jsonb"`
    OccurredAt    time.Time `gorm:"index"`
    RecordedAt    time.Time `gorm:"autoCreateTime"`
}

// TableName enforces lowercase
func (DomainEvent) TableName() string { return "domain_events" }
```

---

## consumer/all_events_consumer.go

```go
package consumer

import (
    "context"
    "encoding/json"
    "log"
    "time"

    "github.com/go-redis/redis/v8"
    "github.com/google/uuid"
    "github.com/promotionos/event-store/internal/model"
    "github.com/promotionos/event-store/internal/repository"
)

// All channels the event store subscribes to
var allChannels = []string{
    "promotionos.campaign.published",
    "promotionos.campaign.paused",
    "promotionos.catalog.item.excluded",
    "promotionos.customer.segment.updated",
    "promotionos.redemption.redeemed",
    "promotionos.redemption.claim.submitted",
    "promotionos.analytics.budget.exhausted",
    "promotionos.campaign.budget.updated",
}

type AllEventsConsumer struct {
    repo   repository.EventStoreRepository
    client *redis.Client
}

func NewAllEventsConsumer(repo repository.EventStoreRepository,
                           client *redis.Client) *AllEventsConsumer {
    return &AllEventsConsumer{repo: repo, client: client}
}

func (c *AllEventsConsumer) Start() {
    pubsub := c.client.Subscribe(context.Background(), allChannels...)
    log.Println("Event Store: subscribed to all channels")

    go func() {
        for msg := range pubsub.Channel() {
            c.handle(msg)
        }
    }()
}

func (c *AllEventsConsumer) handle(msg *redis.Message) {
    var raw map[string]interface{}
    if err := json.Unmarshal([]byte(msg.Payload), &raw); err != nil {
        log.Printf("Event Store: failed to parse event on %s: %v", msg.Channel, err)
        return
    }

    event := &model.DomainEvent{
        ID:            uuid.New().String(),
        TenantID:      str(raw["tenantId"]),
        Context:       extractContext(msg.Channel),
        AggregateID:   str(raw["campaignId"]),
        AggregateType: extractAggregateType(msg.Channel),
        EventType:     extractEventType(msg.Channel),
        SchemaVersion: int(num(raw["schemaVersion"])),
        Payload:       msg.Payload,
        OccurredAt:    parseTime(str(raw["occurredAt"])),
    }

    if err := c.repo.Append(event); err != nil {
        log.Printf("Event Store: failed to append event: %v", err)
    }
}

func extractContext(channel string) string {
    switch {
    case contains(channel, "campaign"):
        return "campaign"
    case contains(channel, "catalog"), contains(channel, "customer"):
        return "catalog"
    case contains(channel, "redemption"):
        return "redemption"
    case contains(channel, "analytics"):
        return "analytics"
    default:
        return "unknown"
    }
}

func extractEventType(channel string) string {
    types := map[string]string{
        "promotionos.campaign.published":             "CampaignPublished",
        "promotionos.campaign.paused":                "CampaignPaused",
        "promotionos.catalog.item.excluded":          "CatalogItemExcluded",
        "promotionos.customer.segment.updated":       "SegmentUpdated",
        "promotionos.redemption.redeemed":            "OfferRedeemed",
        "promotionos.redemption.claim.submitted":     "ClaimSubmitted",
        "promotionos.analytics.budget.exhausted":     "BudgetExhausted",
        "promotionos.campaign.budget.updated":        "BudgetUpdated",
    }
    if t, ok := types[channel]; ok {
        return t
    }
    return "Unknown"
}

func extractAggregateType(channel string) string {
    switch extractContext(channel) {
    case "campaign":
        return "Campaign"
    case "catalog":
        return "CatalogItem"
    case "redemption":
        return "Redemption"
    case "analytics":
        return "CampaignMetrics"
    default:
        return "Unknown"
    }
}

func str(v interface{}) string {
    if v == nil { return "" }
    if s, ok := v.(string); ok { return s }
    return ""
}

func num(v interface{}) float64 {
    if v == nil { return 0 }
    if n, ok := v.(float64); ok { return n }
    return 0
}

func contains(s, sub string) bool {
    return len(s) >= len(sub) && (s == sub ||
        len(s) > 0 && containsStr(s, sub))
}

func containsStr(s, sub string) bool {
    for i := 0; i <= len(s)-len(sub); i++ {
        if s[i:i+len(sub)] == sub { return true }
    }
    return false
}

func parseTime(s string) time.Time {
    t, err := time.Parse(time.RFC3339, s)
    if err != nil { return time.Now().UTC() }
    return t
}
```

---

## repository/event_store_repo.go

```go
package repository

import (
    "github.com/promotionos/event-store/internal/model"
    "gorm.io/gorm"
)

type EventStoreRepository interface {
    Append(event *model.DomainEvent) error
    FindByAggregate(aggregateID string, tenantID string) ([]*model.DomainEvent, error)
    FindByContext(context string, tenantID string, from string, to string) ([]*model.DomainEvent, error)
}

type EventStoreRepositoryImpl struct {
    db *gorm.DB
}

func NewEventStoreRepositoryImpl(db *gorm.DB) *EventStoreRepositoryImpl {
    return &EventStoreRepositoryImpl{db: db}
}

func (r *EventStoreRepositoryImpl) Append(event *model.DomainEvent) error {
    // Append-only — never UPDATE, never DELETE
    return r.db.Create(event).Error
}

func (r *EventStoreRepositoryImpl) FindByAggregate(aggregateID, tenantID string) ([]*model.DomainEvent, error) {
    var events []*model.DomainEvent
    err := r.db.Where("aggregate_id = ? AND tenant_id = ?", aggregateID, tenantID).
        Order("occurred_at ASC").
        Find(&events).Error
    return events, err
}
```

---

## main.go

```go
package main

import (
    "log"
    "os"

    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "github.com/promotionos/event-store/internal/consumer"
    "github.com/promotionos/event-store/internal/repository"
)

func main() {
    db, err := gorm.Open(postgres.Open(os.Getenv("DB_URL")), &gorm.Config{})
    if err != nil {
        log.Fatalf("Failed to connect to DB: %v", err)
    }

    opt, err := redis.ParseURL(os.Getenv("REDIS_URL"))
    if err != nil {
        log.Fatalf("Failed to parse Redis URL: %v", err)
    }
    redisClient := redis.NewClient(opt)

    repo := repository.NewEventStoreRepositoryImpl(db)

    // Start consuming all events
    c := consumer.NewAllEventsConsumer(repo, redisClient)
    c.Start()

    log.Println("Event Store started — consuming all domain events")

    // Minimal API for debugging
    r := gin.Default()
    r.GET("/health", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })
    r.GET("/events", func(c *gin.Context) {
        // Query events by context, aggregateId, tenantId, date range
        // Useful for session debugging
        c.JSON(200, gin.H{"message": "query params: tenantId, context, aggregateId, from, to"})
    })

    port := os.Getenv("PORT")
    if port == "" { port = "8086" }
    r.Run(":" + port)
}
```

---

## README.md

```markdown
# Event Store

Pure infrastructure — consumes ALL domain events and appends to append-only PostgreSQL.
No business logic. Pre-built by facilitator. Teams do not modify this service.

## Purpose
- Audit trail for all domain events
- AI/ML readiness — versioned event history

## Channels Subscribed
- promotionos.campaign.published
- promotionos.campaign.paused
- promotionos.catalog.item.excluded
- promotionos.customer.segment.updated
- promotionos.redemption.redeemed
- promotionos.redemption.claim.submitted
- promotionos.analytics.budget.exhausted
- promotionos.campaign.budget.updated

## Rules
- APPEND ONLY. No UPDATE. No DELETE. Ever.
- Teams can READ from this for debugging during the session.
```
