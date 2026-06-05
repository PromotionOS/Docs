# Catalog & Customer Service — Skeleton Build Template

> Language: Go 1.21
> Framework: Gin
> DB: PostgreSQL via Gorm + Goose migrations
> Events: Redis Pub/Sub (publishes CatalogItemExcluded, SegmentUpdated)
> Deploy: Railway
> Status: Skeleton only — all business logic stubbed

---

## Repo Structure

```
catalog-customer-service/
├── cmd/
│   └── main.go
├── internal/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── upc.go
│   │   │   ├── category.go
│   │   │   ├── customer.go
│   │   │   ├── loyalty_tier.go
│   │   │   └── purchase.go
│   │   ├── service/
│   │   │   ├── hierarchy_resolver.go
│   │   │   └── segment_calculator.go
│   │   └── repository/
│   │       ├── catalog_repository.go
│   │       └── customer_repository.go
│   ├── application/
│   │   ├── catalog_service.go
│   │   └── customer_service.go
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   ├── catalog_repo_impl.go
│   │   │   └── customer_repo_impl.go
│   │   └── event/
│   │       └── publisher.go
│   └── api/
│       ├── catalog_handler.go
│       └── customer_handler.go
├── db/
│   └── migrations/
│       ├── 001_catalog_schema.sql
│       └── 002_seed_test_data.sql
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

---

## go.mod

```go
module github.com/promotionos/catalog-customer-service

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-redis/redis/v8 v8.11.5
    github.com/google/uuid v1.4.0
    github.com/joho/godotenv v1.5.1
    gorm.io/driver/postgres v1.5.4
    gorm.io/gorm v1.25.5
    github.com/pressly/goose/v3 v3.16.0
)
```

---

## Domain Model

### model/upc.go

```go
package model

type UPC struct {
    Code       string
    TenantID   string
    Name       string
    Price      float64
    CategoryID string
    Excluded   bool
}
```

### model/category.go

```go
package model

type Category struct {
    ID                   string
    TenantID             string
    Name                 string
    ParentID             string
    Excluded             bool
    ExcludedInheritedFrom string
    ExcludeReason        string
    Children             []string
}
```

### model/customer.go

```go
package model

type Customer struct {
    ID          string
    TenantID    string
    Name        string
    AnnualSpend float64
    Segments    []string
    Division    string
}

func (c *Customer) LoyaltyTier() LoyaltyTier {
    // Always calculated from annualSpend — never stored
    switch {
    case c.AnnualSpend >= 5000:
        return Platinum
    case c.AnnualSpend >= 2000:
        return Gold
    case c.AnnualSpend >= 500:
        return Silver
    default:
        return Basic
    }
}
```

### model/loyalty_tier.go

```go
package model

type LoyaltyTier string

const (
    Basic    LoyaltyTier = "BASIC"
    Silver   LoyaltyTier = "SILVER"
    Gold     LoyaltyTier = "GOLD"
    Platinum LoyaltyTier = "PLATINUM"
)
```

### model/purchase.go

```go
package model

import "time"

type Purchase struct {
    ID         string
    CustomerID string
    TenantID   string
    UPCCode    string
    Quantity   int
    Spend      float64
    PurchasedAt time.Time
    StoreID    string
}
```

### model/store.go

```go
package model

type Store struct {
    ID             string
    TenantID       string
    Name           string
    DivisionID     string
    Region         string
    Address        string
    StoreManagerID string
}

type StoreManager struct {
    ID      string
    StoreID string
    TenantID string
    Name    string
    Email   string
}
```

---

## Domain Service Interfaces

### service/hierarchy_resolver.go

```go
package service

import "github.com/promotionos/catalog-customer-service/internal/domain/model"

type HierarchyResolver interface {
    // IsExcluded traverses the category hierarchy upward from the UPC's category
    // Returns excluded=true and the categoryId that caused the exclusion
    IsExcluded(upcCode string, tenantID string) (excluded bool, excludedFrom string, err error)

    // ResolveInheritance returns all UPC codes affected by a category exclusion change
    // Traverses downward through all child categories
    ResolveInheritance(categoryID string, tenantID string) ([]string, error)
}
```

### service/segment_calculator.go

```go
package service

type SegmentCalculator interface {
    // Calculate computes segment membership from 90-day rolling purchase history
    Calculate(customerID string, tenantID string) ([]string, error)
}
```

### repository/catalog_repository.go

```go
package repository

import "github.com/promotionos/catalog-customer-service/internal/domain/model"

type CatalogRepository interface {
    FindUPC(code string, tenantID string) (*model.UPC, error)
    FindUPCsBulk(codes []string, tenantID string) ([]*model.UPC, error)
    FindCategory(id string, tenantID string) (*model.Category, error)
    FindCategoryChildren(parentID string, tenantID string) ([]*model.Category, error)
    FindCategoryByUPC(upcCode string, tenantID string) (*model.Category, error)
    SaveExclusion(categoryID string, excluded bool, reason string, tenantID string) error
    FindStoresByDivision(divisionID string, tenantID string) ([]*Store, error)
    FindStoreManager(storeID string, tenantID string) (*StoreManager, error)
}

type CustomerRepository interface {
    FindByID(id string, tenantID string) (*model.Customer, error)
    FindPurchases90Days(customerID string, tenantID string) ([]*model.Purchase, error)
    SavePurchase(purchase *model.Purchase) error
    UpdateSegments(customerID string, segments []string, tenantID string) error
}
```

---

## Application Services (Stubs)

### application/catalog_service.go

```go
package application

import "errors"

var ErrNotImplemented = errors.New("not implemented")

type CatalogService struct {
    repo             repository.CatalogRepository
    hierarchyResolver service.HierarchyResolver
    publisher        *event.Publisher
}

func (s *CatalogService) GetUPC(code string, tenantID string) (*UPCResponse, error) {
    // TODO Team 4 Sprint 1 — implement UPC lookup with exclusion inheritance
    return nil, ErrNotImplemented
}

func (s *CatalogService) GetUPCsBulk(codes []string, tenantID string) ([]*UPCResponse, error) {
    // TODO Team 4 Sprint 3
    return nil, ErrNotImplemented
}

func (s *CatalogService) GetCategory(id string, tenantID string) (*CategoryResponse, error) {
    // TODO Team 4 Sprint 1
    return nil, ErrNotImplemented
}

func (s *CatalogService) UpdateExclusion(categoryID string, excluded bool,
                                          reason string, tenantID string) error {
    // TODO Team 4 Sprint 2
    // 1. Resolve all affected UPCs via hierarchyResolver.ResolveInheritance()
    // 2. Update exclusion flags in DB
    // 3. Publish CatalogItemExcluded event
    return ErrNotImplemented
}

func (s *CatalogService) GetStoresByDivision(divisionID string, tenantID string) ([]*StoreResponse, error) {
    // TODO Catalog TL Sprint 2 — used by Notification Service for store manager routing
    return nil, ErrNotImplemented
}

func (s *CatalogService) GetStoreManager(storeID string, tenantID string) (*StoreManagerResponse, error) {
    // TODO Catalog TL Sprint 2
    return nil, ErrNotImplemented
}
```

### application/customer_service.go

```go
type CustomerService struct {
    repo              repository.CustomerRepository
    segmentCalculator service.SegmentCalculator
    publisher         *event.Publisher
}

func (s *CustomerService) GetCustomer(id string, tenantID string) (*CustomerResponse, error) {
    // TODO Team 4 Sprint 1 — implement with runtime loyalty tier calculation
    return nil, ErrNotImplemented
}

func (s *CustomerService) RecalculateSegments(customerID string, tenantID string) error {
    // TODO Team 4 Sprint 2
    // 1. segmentCalculator.Calculate()
    // 2. Compare with current segments
    // 3. If changed — save + publish SegmentUpdated event
    return ErrNotImplemented
}

func (s *CustomerService) RecordPurchase(purchase *model.Purchase) error {
    // TODO Team 4 Sprint 3
    return ErrNotImplemented
}
```

---

## API Handlers (Stubs)

### api/catalog_handler.go

```go
package api

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

type CatalogHandler struct {
    catalogService *application.CatalogService
}

func (h *CatalogHandler) GetUPC(c *gin.Context) {
    // TODO Team 4 Sprint 1
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CatalogHandler) GetUPCsBulk(c *gin.Context) {
    // TODO Team 4 Sprint 3
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CatalogHandler) GetCategory(c *gin.Context) {
    // TODO Team 4 Sprint 1
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CatalogHandler) UpdateExclusion(c *gin.Context) {
    // TODO Team 4 Sprint 2
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CatalogHandler) GetStoresByDivision(c *gin.Context) {
    // TODO Catalog TL Sprint 2
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CatalogHandler) GetStoreManager(c *gin.Context) {
    // TODO Catalog TL Sprint 2
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}
```

### api/customer_handler.go

```go
type CustomerHandler struct {
    customerService *application.CustomerService
}

func (h *CustomerHandler) GetCustomer(c *gin.Context) {
    // TODO Team 4 Sprint 1
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CustomerHandler) RecalculateSegments(c *gin.Context) {
    // TODO Team 4 Sprint 2
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}

func (h *CustomerHandler) RecordPurchase(c *gin.Context) {
    // TODO Team 4 Sprint 3
    c.JSON(http.StatusNotImplemented, gin.H{"error": "not implemented"})
}
```

---

## Event Publisher

```go
// infrastructure/event/publisher.go
package event

import (
    "context"
    "encoding/json"
    "github.com/go-redis/redis/v8"
)

type Publisher struct {
    client *redis.Client
}

const (
    ChannelCatalogItemExcluded = "promotionos.catalog.item.excluded"
    ChannelSegmentUpdated      = "promotionos.customer.segment.updated"
)

func (p *Publisher) PublishCatalogItemExcluded(event CatalogItemExcludedEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return err
    }
    return p.client.Publish(context.Background(), ChannelCatalogItemExcluded, payload).Err()
}

func (p *Publisher) PublishSegmentUpdated(event SegmentUpdatedEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return err
    }
    return p.client.Publish(context.Background(), ChannelSegmentUpdated, payload).Err()
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
)

func main() {
    dbURL := os.Getenv("DB_URL")
    redisURL := os.Getenv("REDIS_URL")
    port := os.Getenv("PORT")
    if port == "" {
        port = "8084"
    }

    // DB connection
    db, err := gorm.Open(postgres.Open(dbURL), &gorm.Config{})
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }

    // Redis connection
    opt, err := redis.ParseURL(redisURL)
    if err != nil {
        log.Fatalf("Failed to parse Redis URL: %v", err)
    }
    redisClient := redis.NewClient(opt)

    // Run migrations
    runMigrations(dbURL)

    // Wire dependencies
    publisher := event.NewPublisher(redisClient)
    catalogRepo := repository.NewCatalogRepositoryImpl(db)
    customerRepo := repository.NewCustomerRepositoryImpl(db)
    hierarchyResolver := service.NewHierarchyResolverImpl(catalogRepo)
    segmentCalc := service.NewSegmentCalculatorImpl(customerRepo)
    catalogService := application.NewCatalogService(catalogRepo, hierarchyResolver, publisher)
    customerService := application.NewCustomerService(customerRepo, segmentCalc, publisher)

    // Routes
    r := gin.Default()
    r.GET("/health", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

    catalogH := api.NewCatalogHandler(catalogService)
    r.GET("/catalog/upc/:code", catalogH.GetUPC)
    r.POST("/catalog/upcs/bulk", catalogH.GetUPCsBulk)
    r.GET("/catalog/category/:id", catalogH.GetCategory)
    r.POST("/catalog/exclusions", catalogH.UpdateExclusion)
    r.GET("/stores", catalogH.GetStoresByDivision)
    r.GET("/stores/:id/manager", catalogH.GetStoreManager)

    customerH := api.NewCustomerHandler(customerService)
    r.GET("/customers/:id", customerH.GetCustomer)
    r.POST("/customers/:id/recalculate-segments", customerH.RecalculateSegments)
    r.POST("/customers/:id/purchases", customerH.RecordPurchase)

    log.Printf("Catalog & Customer Service starting on port %s", port)
    r.Run(":" + port)
}
```

---

## DB Migration — Catalog Schema (001_catalog_schema.sql)

```sql
-- existing tables omitted for brevity

CREATE TABLE IF NOT EXISTS catalog.stores (
    id VARCHAR(255) NOT NULL,
    tenant_id VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    division_id VARCHAR(255) NOT NULL,
    region VARCHAR(255),
    address VARCHAR(500),
    store_manager_id VARCHAR(255),
    PRIMARY KEY (id, tenant_id)
);

CREATE TABLE IF NOT EXISTS catalog.store_managers (
    id VARCHAR(255) NOT NULL,
    store_id VARCHAR(255) NOT NULL,
    tenant_id VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (id, tenant_id)
);
```
