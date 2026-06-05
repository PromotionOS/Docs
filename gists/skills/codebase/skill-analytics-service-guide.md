---
name: analytics-service-guide
description: Codebase guide for Analytics Service — folder structure, pre-built vs stub, domain model, event wiring, planted bug location, implementation patterns
metadata:
  type: reference
---

# Analytics Service — Codebase Guide

## Quick Facts
- Language: Go 1.21
- Framework: Gin
- Port: 8085 (defaults to `$PORT`)
- DB Schema: `analytics` (PostgreSQL via Goose migrations)
- Status: Pre-built with business logic + bug planted in `LiftCalculatorImpl.Calculate()`

## Folder Structure

```
analytics-service/
├── cmd/
│   └── main.go                              ← wires deps, starts consumers, starts Gin
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   └── campaign_metrics.go          ← CampaignMetrics struct + AddRedemption()
│   │   ├── service/
│   │   │   ├── lift_calculator.go           ← interface — Calculate(*CampaignMetrics) error
│   │   │   └── budget_burn_tracker.go       ← interface — Update(), IsExhausted()
│   │   └── repository/
│   │       └── analytics_repository.go      ← interface — FindByCampaign, Save
│   ├── application/
│   │   └── analytics_service.go             ← pre-built: HandleOfferRedeemed, HandleCampaignPublished, GetReport
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   └── analytics_repo_impl.go
│   │   └── event/
│   │       ├── publisher.go                 ← publishes BudgetExhausted to Redis
│   │       ├── offer_redeemed_consumer.go   ← subscribes to promotionos.redemption.redeemed
│   │       ├── campaign_published_consumer.go ← subscribes to promotionos.campaign.published
│   │       └── budget_updated_consumer.go   ← TODO Sprint 3 — handles BudgetUpdated reactivation
│   └── api/
│       └── analytics_handler.go             ← GetReport (done), GetBurn (stub Sprint 2), GetLift (stub)
├── db/
│   └── migrations/
│       ├── 001_analytics_schema.sql
│       └── 002_seed_test_data.sql           ← seeds BaselineSalesPerDay for test campaigns
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

## What's Pre-built

`CampaignMetrics.AddRedemption(discountApplied)` — fully implemented. Increments `BurnedAmount`, `RedemptionCount`, recalculates `BudgetBurnPercent`.

`BudgetBurnTrackerImpl` — correctly implements `Update()` and `IsExhausted()`. Threshold is `>= 95.0` (this is correct). The bug is in `LiftCalculator`, not here.

`AnalyticsService.HandleOfferRedeemed(event)` — fully wired: calls `burnTracker.Update()`, calls `liftCalculator.Calculate()`, saves metrics, publishes `BudgetExhausted` if exhausted.

`AnalyticsService.HandleCampaignPublished(event)` — initialises a `CampaignMetrics` record on first `CampaignPublished`. Idempotent — skips if metrics already exist.

`AnalyticsService.GetReport(campaignID, tenantID)` — loads and returns metrics.

`OfferRedeemedConsumer` and `CampaignPublishedConsumer` — both wired and calling their handlers.

`GET /analytics/campaigns/:id/report` — fully implemented.

**Planted bug:** In `LiftCalculatorImpl.Calculate()` (lives in the infrastructure layer as a concrete implementation of the `LiftCalculator` interface):

```go
// BUG — missing nil guard on TotalFundingCost
roi := *metrics.IncrementalMargin / metrics.TotalFundingCost  // PANIC when TotalFundingCost = 0
```

This panics when a campaign is 100% Kroger-funded (`vendorShare = 0`, so `TotalFundingCost = 0`). The fix: if `TotalFundingCost == 0`, set `metrics.ROI = nil` instead of dividing. The ubiquitous language defines this: "If `totalFundingCost` is zero — ROI is `null`, not a division error."

## What You Implement (by sprint)

**Sprint 1**
- Find and fix the `LiftCalculatorImpl` divide-by-zero bug (after RCA)
- Verify `BudgetBurnTrackerImpl.IsExhausted()` uses `>= 95.0` (it does — this is the control to Campaign Service's bug which uses `> 94`)

**Sprint 2**
- `GET /analytics/campaigns/:id/burn` — return real-time burn state (`burnedAmount`, `budgetBurnPercent`, `redemptionCount`)

**Sprint 3**
- `budget_updated_consumer.go` — implement handler for `BudgetUpdated` event (campaign budget reactivation when budget is increased post-exhaustion)

## Domain Model

```go
// model/campaign_metrics.go
CampaignMetrics struct
  ID:                     string
  CampaignID:             string
  TenantID:               string
  BaselineSalesPerDay:    float64    ← seeded from test data; represents pre-campaign baseline
  ActualSalesPerDay:      float64    ← updated as redemptions come in
  Lift:                   *float64   ← nil until calculated; actual - baseline
  LiftPercentage:         *float64   ← nil if baseline is 0 (not an error)
  TotalFundingCost:       float64    ← total $ vendor pays across all redemptions
  IncrementalMargin:      *float64   ← lift × 0.30 (30% average margin)
  ROI:                    *float64   ← nil if TotalFundingCost == 0 (not an error)
  BudgetBurnPercent:      float64    ← (burnedAmount / totalAmount) × 100
  BurnedAmount:           float64    ← cumulative discount applied
  TotalAmount:            float64    ← campaign budget totalAmount
  RedemptionCount:        int
  BudgetExhaustedEmitted: bool       ← gate prevents duplicate BudgetExhausted events

// Methods:
CampaignMetrics.AddRedemption(discountApplied float64)
// Adds to BurnedAmount, increments RedemptionCount, recalculates BudgetBurnPercent
```

## Publishing Events

One event is published by this service:

```go
// publisher.go publishes BudgetExhausted to:
// Channel: promotionos.analytics.budget.exhausted
// (this is what Campaign Service consumes to pause the campaign)

publisher.PublishBudgetExhausted(BudgetExhaustedEvent{
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
```

This is called by `AnalyticsService.HandleOfferRedeemed()` only when `burnTracker.Update()` returns `exhausted = true`. The `BudgetExhaustedEmitted` flag on `CampaignMetrics` ensures it fires exactly once.

## Consuming Events

| Consumer file | Channel | Calls |
|---|---|---|
| `offer_redeemed_consumer.go` | `promotionos.redemption.redeemed` | `HandleOfferRedeemed(event)` |
| `campaign_published_consumer.go` | `promotionos.campaign.published` | `HandleCampaignPublished(event)` |
| `budget_updated_consumer.go` | `promotionos.campaign.budget.updated` | TODO Sprint 3 |

Consumer pattern (consistent across all three):

```go
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
```

## Key Interfaces You Must Implement

The two domain service interfaces are already implemented as concrete types (they are pre-built). Your Sprint 1 task is to fix the bug in the existing implementation.

```go
// domain/service/lift_calculator.go
type LiftCalculator interface {
    Calculate(metrics *model.CampaignMetrics) error
    // Must set ROI = nil when TotalFundingCost == 0
    // Must set LiftPercentage = nil when BaselineSalesPerDay == 0
}

// domain/service/budget_burn_tracker.go
type BudgetBurnTracker interface {
    Update(metrics *model.CampaignMetrics, discountApplied float64) (exhausted bool, err error)
    IsExhausted(metrics *model.CampaignMetrics) bool
    // IsExhausted uses >= 95.0 — this is correct
}

// domain/repository/analytics_repository.go
type AnalyticsRepository interface {
    FindByCampaign(campaignID, tenantID string) (*model.CampaignMetrics, error)
    Save(metrics *model.CampaignMetrics) error
}
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/analytics-db
export REDIS_URL=redis://localhost:6379
export PORT=8085
go run ./cmd/main.go
# Health: http://localhost:8085/health
```

Run tests:
```bash
go test ./... -v -count=1
# or: ./scripts/test.sh
```

## Patterns In This Codebase

**Pointer fields signal "not yet calculable," not "missing data."** `Lift`, `LiftPercentage`, `IncrementalMargin`, `ROI` are all `*float64`. A nil pointer means "this value cannot be calculated given current inputs" — it is semantically valid. Never substitute 0 for nil. Callers must null-check before using.

**`BudgetExhaustedEmitted` is the idempotency gate.** `HandleOfferRedeemed` calls `burnTracker.Update()` on every redemption. `Update()` returns `exhausted=true` only when `BudgetBurnPercent >= 95.0` AND `BudgetExhaustedEmitted` is false, then sets it to true. This ensures `BudgetExhausted` is published exactly once per campaign. After the flag is set, `Update()` always returns `exhausted=false` regardless of burn level.

**Consumers run as goroutines — errors must not panic.** Each consumer's `Subscribe()` starts a `go func()`. Any panic inside that goroutine kills the goroutine silently. Always use `continue` on unmarshal errors, and log handler errors rather than panicking.

## Do NOT Do This

**Do not calculate ROI when `TotalFundingCost` is zero.** This is the planted bug. When `vendorShare = 0%` (100% Kroger-funded campaign), `TotalFundingCost` is 0. The fix is: `if metrics.TotalFundingCost == 0 { metrics.ROI = nil; return nil }`. Do not substitute `TotalFundingCost = 1` or any other sentinel — nil is the correct domain answer.

**Do not use this service as a source of truth for campaign status.** Analytics Service tracks metrics — it does not own campaign lifecycle. When `BudgetExhausted` fires, Campaign Service pauses the campaign. Analytics does not call Campaign Service. Analytics does not know or care what Campaign Service does with the event.

**Do not store `BaselineSalesPerDay` as a runtime calculation.** Baseline sales come from seeded test data (`002_seed_test_data.sql`). They represent historical pre-campaign performance loaded at campaign initialisation. `HandleCampaignPublished` initialises the metrics record and leaves `BaselineSalesPerDay` to be populated from the seed. Do not calculate it from live redemption data — that would overwrite the pre-campaign baseline.
