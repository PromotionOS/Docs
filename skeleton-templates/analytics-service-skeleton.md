# Analytics Service — Skeleton Build Template

> Language: Go 1.21
> Framework: Gin
> DB: PostgreSQL via Gorm + Goose migrations
> Events: Redis Pub/Sub (consumes OfferRedeemed, CampaignPublished, BudgetExhausted; publishes BudgetExhausted)
> Deploy: Railway
> Status: Pre-built with business logic + bug planted in LiftCalculator.calculateROI()

---

## Repo Structure

```
analytics-service/
├── cmd/
│   └── main.go
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   └── campaign_metrics.go
│   │   ├── service/
│   │   │   ├── lift_calculator.go
│   │   │   └── budget_burn_tracker.go
│   │   └── repository/
│   │       └── analytics_repository.go
│   ├── application/
│   │   └── analytics_service.go
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   └── analytics_repo_impl.go
│   │   └── event/
│   │       ├── publisher.go
│   │       ├── offer_redeemed_consumer.go
│   │       ├── campaign_published_consumer.go
│   │       └── budget_updated_consumer.go
│   └── api/
│       └── analytics_handler.go
├── db/
│   └── migrations/
│       ├── 001_analytics_schema.sql
│       └── 002_seed_test_data.sql
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

---

## Domain Model

### model/campaign_metrics.go

```go
package model

type CampaignMetrics struct {
    ID                      string
    CampaignID              string
    TenantID                string
    BaselineSalesPerDay     float64
    ActualSalesPerDay       float64
    Lift                    *float64  // pointer — can be nil
    LiftPercentage          *float64  // pointer — nil when baseline is 0
    TotalFundingCost        float64
    IncrementalMargin       *float64
    ROI                     *float64  // pointer — nil when totalFundingCost is 0
    BudgetBurnPercent       float64
    BurnedAmount            float64
    TotalAmount             float64
    RedemptionCount         int
    BudgetExhaustedEmitted  bool
}

func (m *CampaignMetrics) AddRedemption(discountApplied float64) {
    m.BurnedAmount += discountApplied
    m.RedemptionCount++
    if m.TotalAmount > 0 {
        m.BudgetBurnPercent = (m.BurnedAmount / m.TotalAmount) * 100
    }
}
```

---

## Domain Service Interfaces

### service/lift_calculator.go

```go
package service

type LiftCalculator interface {
    Calculate(metrics *model.CampaignMetrics) error
}
```

### service/budget_burn_tracker.go

```go
package service

type BudgetBurnTracker interface {
    Update(metrics *model.CampaignMetrics, discountApplied float64) (exhausted bool, err error)
    IsExhausted(metrics *model.CampaignMetrics) bool
}
```

---

## Implementation — LiftCalculator (Bug Planted)

```go
// infrastructure/lift_calculator_impl.go
package service

type LiftCalculatorImpl struct{}

func (l *LiftCalculatorImpl) Calculate(metrics *model.CampaignMetrics) error {
    // Calculate lift
    lift := metrics.ActualSalesPerDay - metrics.BaselineSalesPerDay
    metrics.Lift = &lift

    // Calculate lift percentage
    if metrics.BaselineSalesPerDay > 0 {
        liftPct := (lift / metrics.BaselineSalesPerDay) * 100
        metrics.LiftPercentage = &liftPct
    }
    // If baseline is 0 — LiftPercentage stays nil (correct)

    // Calculate incremental margin (30% average margin)
    if metrics.Lift != nil {
        margin := *metrics.Lift * 0.30
        metrics.IncrementalMargin = &margin
    }

    // BUG PLANTED HERE: missing null guard on totalFundingCost
    // This causes divide-by-zero panic on 100% Kroger-funded campaigns
    // where vendorShare = 0 and totalFundingCost = 0
    // Teams find this in Sprint 1 via RCA
    if metrics.IncrementalMargin != nil {
        roi := *metrics.IncrementalMargin / metrics.TotalFundingCost // PANIC when TotalFundingCost = 0
        metrics.ROI = &roi
    }

    return nil
}
```

### BudgetBurnTracker Implementation (Correct)

```go
// infrastructure/budget_burn_tracker_impl.go
type BudgetBurnTrackerImpl struct{}

func (b *BudgetBurnTrackerImpl) Update(metrics *model.CampaignMetrics,
                                        discountApplied float64) (bool, error) {
    metrics.AddRedemption(discountApplied)

    exhausted := b.IsExhausted(metrics)
    if exhausted && !metrics.BudgetExhaustedEmitted {
        metrics.BudgetExhaustedEmitted = true
        return true, nil
    }
    return false, nil
}

func (b *BudgetBurnTrackerImpl) IsExhausted(metrics *model.CampaignMetrics) bool {
    return metrics.BudgetBurnPercent >= 95.0
}
```

---

## Application Service

```go
// application/analytics_service.go
package application

type AnalyticsService struct {
    repo           repository.AnalyticsRepository
    liftCalculator service.LiftCalculator
    burnTracker    service.BudgetBurnTracker
    publisher      *event.Publisher
}

func (s *AnalyticsService) HandleOfferRedeemed(e OfferRedeemedEvent) error {
    metrics, err := s.repo.FindByCampaign(e.CampaignID, e.TenantID)
    if err != nil {
        return err
    }

    exhausted, err := s.burnTracker.Update(metrics, e.Payload.DiscountApplied)
    if err != nil {
        return err
    }

    if err := s.liftCalculator.Calculate(metrics); err != nil {
        return err
    }

    if err := s.repo.Save(metrics); err != nil {
        return err
    }

    if exhausted {
        return s.publisher.PublishBudgetExhausted(BudgetExhaustedEvent{
            EventID:    uuid.New().String(),
            TenantID:   e.TenantID,
            CampaignID: e.CampaignID,
            OccurredAt: time.Now().UTC().Format(time.RFC3339),
            SchemaVersion: 1,
            Payload: BudgetExhaustedPayload{
                TotalAmount:       metrics.TotalAmount,
                BurnedAmount:      metrics.BurnedAmount,
                BudgetBurnPercent: metrics.BudgetBurnPercent,
                RedemptionCount:   metrics.RedemptionCount,
                ExhaustedAt:       time.Now().UTC().Format(time.RFC3339),
            },
        })
    }

    return nil
}

func (s *AnalyticsService) HandleCampaignPublished(e CampaignPublishedEvent) error {
    // Initialise campaign metrics record
    existing, _ := s.repo.FindByCampaign(e.CampaignID, e.TenantID)
    if existing != nil {
        return nil // idempotent — already initialised
    }
    metrics := &model.CampaignMetrics{
        ID:         uuid.New().String(),
        CampaignID: e.CampaignID,
        TenantID:   e.TenantID,
        // BaselineSalesPerDay loaded from seeded test data
    }
    return s.repo.Save(metrics)
}

func (s *AnalyticsService) GetReport(campaignID, tenantID string) (*model.CampaignMetrics, error) {
    return s.repo.FindByCampaign(campaignID, tenantID)
}
```

---

## API Handler

```go
// api/analytics_handler.go
package api

type AnalyticsHandler struct {
    service *application.AnalyticsService
}

func (h *AnalyticsHandler) GetReport(c *gin.Context) {
    campaignID := c.Param("id")
    tenantID := c.Query("tenantId")
    if tenantID == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "tenantId required"})
        return
    }
    metrics, err := h.service.GetReport(campaignID, tenantID)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, metrics)
}

func (h *AnalyticsHandler) GetBurn(c *gin.Context) {
    // TODO Team 5 Sprint 2 — return burn state for real-time frontend polling
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}
```

---

## Event Consumers

```go
// infrastructure/event/offer_redeemed_consumer.go
type OfferRedeemedConsumer struct {
    service *application.AnalyticsService
}

func (c *OfferRedeemedConsumer) Subscribe(client *redis.Client) {
    pubsub := client.Subscribe(context.Background(), "promotionos.redemption.redeemed")
    go func() {
        for msg := range pubsub.Channel() {
            var event OfferRedeemedEvent
            if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
                continue
            }
            c.service.HandleOfferRedeemed(event)
        }
    }()
}

// campaign_published_consumer.go — same pattern, calls HandleCampaignPublished
// budget_updated_consumer.go — TODO Team 5 Sprint 3, handles BudgetUpdated reactivation
```

---

## main.go

```go
func main() {
    db, _ := gorm.Open(postgres.Open(os.Getenv("DB_URL")), &gorm.Config{})
    opt, _ := redis.ParseURL(os.Getenv("REDIS_URL"))
    redisClient := redis.NewClient(opt)

    runMigrations(os.Getenv("DB_URL"))

    publisher := event.NewPublisher(redisClient)
    repo := repository.NewAnalyticsRepositoryImpl(db)
    liftCalc := &service.LiftCalculatorImpl{}
    burnTracker := &service.BudgetBurnTrackerImpl{}
    svc := application.NewAnalyticsService(repo, liftCalc, burnTracker, publisher)

    // Start consumers
    event.NewOfferRedeemedConsumer(svc).Subscribe(redisClient)
    event.NewCampaignPublishedConsumer(svc).Subscribe(redisClient)

    r := gin.Default()
    r.GET("/health", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

    h := api.NewAnalyticsHandler(svc)
    r.GET("/analytics/campaigns/:id/report", h.GetReport)
    r.GET("/analytics/campaigns/:id/burn", h.GetBurn)
    r.GET("/analytics/campaigns/:id/lift", h.GetLift)

    r.Run(":" + os.Getenv("PORT"))
}
```
