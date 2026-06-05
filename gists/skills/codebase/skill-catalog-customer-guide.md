---
name: catalog-customer-guide
description: Codebase guide for Catalog & Customer Service — folder structure, pre-built vs stub, domain model, event wiring, hierarchy resolution, implementation patterns
metadata:
  type: reference
---

# Catalog & Customer Service — Codebase Guide

## Quick Facts
- Language: Go 1.21
- Framework: Gin
- Port: 8084
- DB Schema: `catalog` (PostgreSQL via Goose migrations)
- Status: Skeleton only — all application service methods return `ErrNotImplemented`

## Folder Structure

```
catalog-customer-service/
├── cmd/
│   └── main.go                              ← dependency wiring + route registration
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── upc.go                       ← UPC struct — code, tenantID, name, price, categoryID, excluded
│   │   │   ├── category.go                  ← Category struct — tree node with parentID, children, excluded, excludedInheritedFrom
│   │   │   ├── customer.go                  ← Customer struct — LoyaltyTier() calculated method
│   │   │   ├── loyalty_tier.go              ← LoyaltyTier enum — BASIC, SILVER, GOLD, PLATINUM
│   │   │   ├── purchase.go                  ← Purchase struct — customerID, upcCode, quantity, spend, purchasedAt
│   │   │   └── store.go                     ← Store + StoreManager structs
│   │   ├── service/
│   │   │   ├── hierarchy_resolver.go        ← interface — IsExcluded(), ResolveInheritance()
│   │   │   └── segment_calculator.go        ← interface — Calculate(customerID, tenantID)
│   │   └── repository/
│   │       ├── catalog_repository.go        ← interface — UPC, Category, Store, StoreManager queries
│   │       └── customer_repository.go       ← interface — Customer, Purchase queries
│   ├── application/
│   │   ├── catalog_service.go               ← STUBS — GetUPC, GetUPCsBulk, GetCategory, UpdateExclusion, GetStoresByDivision, GetStoreManager
│   │   └── customer_service.go             ← STUBS — GetCustomer, RecalculateSegments, RecordPurchase
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   ├── catalog_repo_impl.go
│   │   │   └── customer_repo_impl.go
│   │   └── event/
│   │       └── publisher.go                 ← PublishCatalogItemExcluded, PublishSegmentUpdated
│   └── api/
│       ├── catalog_handler.go               ← HTTP handlers for catalog routes
│       └── customer_handler.go              ← HTTP handlers for customer routes
├── db/
│   └── migrations/
│       ├── 001_catalog_schema.sql           ← includes stores + store_managers tables
│       └── 002_seed_test_data.sql
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

## What's Pre-built

Route registration in `main.go` is fully wired — all routes exist and call the handler stubs.

`Customer.LoyaltyTier()` method is implemented in `model/customer.go` — it calculates tier from `annualSpend` at runtime. Do not store the tier.

Event publisher constants and method signatures are in `infrastructure/event/publisher.go`.

DB migrations including `stores` and `store_managers` tables are ready in `db/migrations/001_catalog_schema.sql`.

All interface definitions in `domain/service/` and `domain/repository/` are complete.

## What You Implement (by sprint)

**Sprint 1**
- `CatalogService.GetUPC(code, tenantID)` — lookup UPC, call `hierarchyResolver.IsExcluded()`, return `UPCResponse` with `excluded` flag
- `CatalogService.GetCategory(id, tenantID)` — return category with hierarchy context
- `CustomerService.GetCustomer(id, tenantID)` — load from DB, call `customer.LoyaltyTier()`, return response with tier
- Implement `HierarchyResolverImpl.IsExcluded()` — traverse category tree upward until root or excluded ancestor

**Sprint 2**
- `CatalogService.UpdateExclusion(categoryID, excluded, reason, tenantID)`:
  1. Call `hierarchyResolver.ResolveInheritance(categoryID, tenantID)` to get all affected UPC codes
  2. Update `excluded` flag on all affected UPCs in DB
  3. Call `publisher.PublishCatalogItemExcluded(event)` with affected UPC list
- `CustomerService.RecalculateSegments(customerID, tenantID)`:
  1. Call `segmentCalculator.Calculate(customerID, tenantID)` — 90-day rolling purchase history
  2. Compare result with current `customer.Segments`
  3. If changed: call `repo.UpdateSegments(...)` then `publisher.PublishSegmentUpdated(event)`
- `CatalogService.GetStoresByDivision(divisionID, tenantID)`
- `CatalogService.GetStoreManager(storeID, tenantID)`
- Implement `HierarchyResolverImpl.ResolveInheritance()` — traverse category tree downward, collect all UPC codes

**Sprint 3**
- `CatalogService.GetUPCsBulk(codes, tenantID)` — batch lookup, used by Eligibility Service
- `CustomerService.RecordPurchase(purchase)` — write purchase, trigger segment recalculation async

## Domain Model

```go
// model/upc.go
UPC struct
  Code:       string
  TenantID:   string
  Name:       string
  Price:      float64
  CategoryID: string
  Excluded:   bool        ← true if this UPC or any ancestor category is excluded

// model/category.go
Category struct
  ID:                    string
  TenantID:              string
  Name:                  string
  ParentID:              string
  Excluded:              bool
  ExcludedInheritedFrom: string   ← which ancestor caused the exclusion
  ExcludeReason:         string
  Children:              []string

// model/customer.go
Customer struct
  ID:          string
  TenantID:    string
  Name:        string
  AnnualSpend: float64
  Segments:    []string   ← recalculated from 90-day purchase history
  Division:    string
// Customer.LoyaltyTier() → LoyaltyTier  (BASIC/SILVER/GOLD/PLATINUM — never stored)

// model/loyalty_tier.go
LoyaltyTier  — BASIC (<$500) | SILVER ($500-$1999) | GOLD ($2000-$4999) | PLATINUM ($5000+)

// model/purchase.go
Purchase struct
  ID:          string
  CustomerID:  string
  TenantID:    string
  UPCCode:     string
  Quantity:    int
  Spend:       float64
  PurchasedAt: time.Time
  StoreID:     string

// model/store.go
Store struct
  ID:             string
  TenantID:       string
  Name:           string
  DivisionID:     string
  Region:         string
  Address:        string
  StoreManagerID: string

StoreManager struct
  ID:       string
  StoreID:  string
  TenantID: string
  Name:     string
  Email:    string
```

## Publishing Events

Two events are published by this service, via `infrastructure/event/publisher.go`:

```go
// Publish when a category exclusion is updated (affects one or many UPCs):
publisher.PublishCatalogItemExcluded(CatalogItemExcludedEvent{...})
// Channel: promotionos.catalog.item.excluded

// Publish when a customer's segment list changes:
publisher.PublishSegmentUpdated(SegmentUpdatedEvent{...})
// Channel: promotionos.customer.segment.updated

// Both use the same publish pattern:
payload, _ := json.Marshal(event)
client.Publish(ctx, channelName, payload).Err()
```

The channel constants are defined in `publisher.go`:
- `ChannelCatalogItemExcluded = "promotionos.catalog.item.excluded"`
- `ChannelSegmentUpdated = "promotionos.customer.segment.updated"`

Only publish `SegmentUpdated` when the segment list actually changes — compare old vs new before publishing.

## Consuming Events

This service does not consume any events. Eligibility Service and Notification Service call this service via HTTP when they need catalog or customer data.

Routes served (registered in `main.go`):
```
GET  /catalog/upc/:code
POST /catalog/upcs/bulk
GET  /catalog/category/:id
POST /catalog/exclusions
GET  /stores                  ← ?divisionId=X&tenantId=Y
GET  /stores/:id/manager      ← ?tenantId=Y
GET  /customers/:id           ← ?tenantId=Y
POST /customers/:id/recalculate-segments
POST /customers/:id/purchases
GET  /health
```

## Key Interfaces You Must Implement

```go
// domain/service/hierarchy_resolver.go
type HierarchyResolver interface {
    IsExcluded(upcCode string, tenantID string) (excluded bool, excludedFrom string, err error)
    // Walk category tree upward from the UPC's category
    // Return excluded=true + the categoryId that first has Excluded=true

    ResolveInheritance(categoryID string, tenantID string) ([]string, error)
    // Walk category tree downward, collect all UPC codes in all descendant categories
    // Used by UpdateExclusion to find every affected UPC
}

// domain/service/segment_calculator.go
type SegmentCalculator interface {
    Calculate(customerID string, tenantID string) ([]string, error)
    // Load 90-day rolling purchases via CustomerRepository.FindPurchases90Days()
    // Return segment names based on spend patterns, categories purchased
}

// domain/repository/catalog_repository.go + customer_repository.go
// Implement in infrastructure/repository/catalog_repo_impl.go and customer_repo_impl.go
// Use gorm.DB injected via constructor
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/catalog-db
export REDIS_URL=redis://localhost:6379
export PORT=8084
go run ./cmd/main.go
# Health: http://localhost:8084/health
```

Run tests:
```bash
go test ./... -v -count=1
# or: ./scripts/test.sh
```

Migrations run automatically at startup via `runMigrations(dbURL)` in `main.go`.

## Patterns In This Codebase

**Loyalty tier is a computed property, never stored.** `Customer.LoyaltyTier()` is a method on the struct, not a DB column. Every call to `GetCustomer` must call `customer.LoyaltyTier()` and include the result in the response. Never add a `loyalty_tier` column to the customers table.

**Exclusion inheritance flows downward through the category tree.** When `UpdateExclusion` is called for `cat-alcohol`, `ResolveInheritance` must walk every child and grandchild category and collect their UPCs. All those UPCs get `excluded=true`. When a parent is re-included, the same traversal clears the flag downward.

**Segment recalculation is event-driven, not cached.** After `RecordPurchase`, trigger `RecalculateSegments` asynchronously. Only publish `SegmentUpdated` when the resulting segment list differs from the current stored list. This prevents event storms during bulk purchase imports.

## Do NOT Do This

**Do not own customer segment logic in Eligibility Service.** Eligibility Service calls this service for customer data — it does not store segments locally (except for caching). Segment truth lives here. When Eligibility needs to match a customer to a segment restriction, it fetches `CustomerProfile` by calling `GET /customers/:id`.

**Do not traverse the category hierarchy by loading all categories into memory.** The category tree can be large. `IsExcluded` must traverse lazily — load one node at a time, walk upward until you find an excluded ancestor or reach the root. `ResolveInheritance` must traverse lazily downward. Loading the entire catalog into memory on each request will OOM the service.

**Do not expose LoyaltyTier as a request input.** The `POST /customers/:id/recalculate-segments` endpoint accepts no tier input — tier is always derived from `annualSpend`. If an API caller sends a `loyaltyTier` field in the request body, ignore it. Tier cannot be set externally.
