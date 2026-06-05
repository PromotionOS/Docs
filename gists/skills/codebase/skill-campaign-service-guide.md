---
name: campaign-service-guide
description: Codebase guide for Campaign Service — folder structure, pre-built vs stub, domain model, event wiring, implementation patterns
metadata:
  type: reference
---

# Campaign Service — Codebase Guide

## Quick Facts
- Language: Java 17 / Spring Boot 3.x
- Framework: Spring Boot (Maven)
- Port: 8081
- DB Schema: `campaign` (PostgreSQL via Flyway)
- Status: Pre-built with business logic + bug planted in `Campaign.checkBudgetExhaustion()`

## Folder Structure

```
campaign-service/
├── src/main/java/com/promotionos/campaign/
│   ├── CampaignServiceApplication.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Campaign.java              ← Aggregate Root
│   │   │   ├── CampaignStatus.java        ← DRAFT, ACTIVE, PAUSED, SCHEDULED, PENDING_APPROVAL
│   │   │   ├── Offer.java                 ← entity inside Campaign aggregate
│   │   │   ├── OfferType.java             ← PCT_OFF, AMT_OFF, BOGO, THRESHOLD
│   │   │   ├── Funding.java               ← entity — vendorId, vendorShare, krogerShare
│   │   │   ├── Budget.java                ← tracks burnedAmount vs totalAmount
│   │   │   ├── Money.java                 ← value object — BigDecimal + "USD"
│   │   │   ├── Percentage.java            ← value object — wraps BigDecimal
│   │   │   ├── DateRange.java             ← value object — startDate, endDate, overlaps()
│   │   │   ├── TenantId.java              ← value object — wraps String
│   │   │   └── UPC.java                   ← value object — wraps String code
│   │   ├── service/
│   │   │   ├── FundingValidator.java      ← interface — validate(Funding)
│   │   │   ├── ConflictResolver.java      ← interface — checkUPCOverlap(Campaign, List<Campaign>)
│   │   │   └── BudgetTracker.java         ← contains the planted bug (see below)
│   │   ├── repository/
│   │   │   └── CampaignRepository.java    ← interface
│   │   └── event/
│   │       ├── CampaignPublished.java     ← record, published on campaign.publish()
│   │       ├── CampaignPaused.java        ← record, published on campaign.pause()
│   │       └── BudgetExhausted.java       ← record, published when burn >= 95%
│   ├── application/
│   │   └── CampaignApplicationService.java  ← orchestrates all use cases
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   └── CampaignRepositoryImpl.java
│   │   └── event/
│   │       ├── EventPublisher.java           ← sends events to Redis
│   │       └── BudgetExhaustedConsumer.java  ← listens for analytics.budget.exhausted
│   └── api/
│       └── CampaignController.java           ← REST endpoints
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V1__campaign_schema.sql
│       └── V2__seed_test_data.sql
└── src/test/java/com/promotionos/campaign/
    └── CampaignContractTest.java
```

## What's Pre-built

Everything in `domain/model/` is fully implemented — all value objects, Campaign aggregate methods including `draft()`, `publish()`, `pause()`, `checkBudgetExhaustion()`, `pullDomainEvents()`.

`CampaignApplicationService` is fully implemented: `createDraft()`, `addFunding()`, `publish()`, `handleBudgetExhausted()`, `getSummary()`.

`EventPublisher` — fully wired to Redis; resolves channel from event type automatically.

`BudgetExhaustedConsumer` — fully wired; calls `campaignService.handleBudgetExhausted()`.

`GET /campaigns/{id}/summary` — implemented and exposed immediately; unblocks Redemption Service Sprint 2.

**Planted bug:** In `Campaign.checkBudgetExhaustion()`, the condition is `burnPercent > 94` instead of `>= 95`. This triggers a pause at 94.9% instead of 95.0%. Sprint 1 task is to find and fix this via RCA. The correct threshold is defined in the ubiquitous language: `budgetBurnPercent >= 95.0`.

## What You Implement (by sprint)

**Sprint 1**
- `POST /campaigns` — wire `CreateCampaignRequest` into `campaignService.createDraft()`
- `PUT /campaigns/{id}/publish` — wire into `campaignService.publish()`
- `POST /campaigns/{id}/funding` — wire into `campaignService.addFunding()`
- Fix the bug in `Campaign.checkBudgetExhaustion()` (after RCA)

**Sprint 2**
- Implement `FundingValidator` (validates `vendorShare + krogerShare = 100`, vendor is not null)
- Implement `ConflictResolver` (checks UPC overlap with active campaigns using `DateRange.overlaps()`)

**Sprint 3**
- `GET /campaigns/{id}/validate-funding` — calls `FundingValidator`
- `PUT /campaigns/{id}/budget` — `BudgetUpdateRequest` → `campaign.setBudget()`

**Sprint 4**
- `GET /campaigns` — list with `?status=` filter
- Add `PENDING_APPROVAL` to `CampaignStatus` (required by Vendor Service Sprint 2 integration)

## Domain Model

```
Campaign (Aggregate Root)
  id: UUID
  tenantId: TenantId
  name: String
  status: CampaignStatus
  offer: Offer
  funding: Funding
  budget: Budget
  dateRange: DateRange
  stackPermission: boolean
  stackLimit: int
  segmentRestriction: String
  geoScope: List<String>
  domainEvents: List<DomainEvent>   ← cleared by pullDomainEvents()

Offer (Entity)
  id: UUID
  type: OfferType
  value: Money
  thresholdAmount: Money
  discountAmount: Money
  upcScope: List<UPC>

Funding (Entity)
  id: UUID
  vendorId: String
  vendorShare: Percentage
  krogerShare: Percentage

Budget (Entity)
  id: UUID
  totalAmount: Money
  burnedAmount: Money
  exhaustionThreshold: Percentage
  budgetExhaustedEmitted: boolean

Money (Value Object)        — BigDecimal amount + "USD" currency
Percentage (Value Object)   — BigDecimal value
DateRange (Value Object)    — LocalDate startDate, endDate; overlaps(DateRange)
TenantId (Value Object)     — String value
UPC (Value Object)          — String code
```

## Publishing Events

Events are published via `EventPublisher.publishAll()` after calling `campaign.pullDomainEvents()`. The application service owns this flow — never call `EventPublisher` from a controller.

```java
// In CampaignApplicationService:
Campaign saved = campaignRepository.save(campaign);
eventPublisher.publishAll(saved.pullDomainEvents());
```

Channel names (from `application.yml`):
- `CampaignPublished` → `promotionos.campaign.published`
- `CampaignPaused`   → `promotionos.campaign.paused`
- `BudgetExhausted`  → published by Analytics Service, consumed here on `promotionos.analytics.budget.exhausted`

To add a new event type:
1. Create a record in `domain/event/`
2. Add a `domainEvents.add(new YourEvent(...))` call in the aggregate method
3. Add the channel mapping in `EventPublisher.resolveChannel()`
4. Add the channel key to `application.yml` under `promotionos.redis.channels`

## Consuming Events

One consumer exists: `infrastructure/event/BudgetExhaustedConsumer.java`

- Listens on: `promotionos.analytics.budget.exhausted`
- Deserialises to: `BudgetExhaustedEvent`
- Calls: `campaignService.handleBudgetExhausted(event)` which pauses the campaign and publishes `CampaignPaused`
- Pattern: implements Spring `MessageListener`; registered via `RedisMessageListenerContainer` in config

## Key Interfaces You Must Implement

```java
// domain/service/FundingValidator.java
void validate(Funding funding);
// Must throw DomainException("NO_FUNDING_SOURCE") if funding is null
// Must throw DomainException("INVALID_SPLIT") if vendorShare + krogerShare != 100

// domain/service/ConflictResolver.java
void checkUPCOverlap(Campaign campaign, List<Campaign> activeCampaigns);
// Must throw DomainException("UPC_OVERLAP") if any UPC in campaign.offer.upcScope
// appears in any active campaign's offer.upcScope where date ranges overlap

// domain/repository/CampaignRepository.java
Campaign findById(UUID id, TenantId tenantId);
List<Campaign> findByStatus(CampaignStatus status, TenantId tenantId);
List<Campaign> findByUPC(UPC upc, TenantId tenantId);
Campaign save(Campaign campaign);
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/campaign-db
export REDIS_URL=redis://localhost:6379
export TENANT_ID=tenant-kroger-001
mvn spring-boot:run
# Health: http://localhost:8081/actuator/health
```

Run tests:
```bash
mvn test -Dspring.profiles.active=test
# or: ./scripts/test.sh
```

## Patterns In This Codebase

**Domain events via pullDomainEvents():** The aggregate collects events during state transitions (stored in `domainEvents` list). The application service calls `pullDomainEvents()` after saving — this clears the list and returns events for publishing. Never publish before saving; never save without pulling events.

**TenantId on every query:** All repository calls include `TenantId`. The Campaign Service never returns data across tenant boundaries. If you forget the `TenantId` parameter, a `findById` that returns null will throw `EntityNotFoundException` — this is intentional.

**Value objects are immutable:** `Money`, `Percentage`, `DateRange`, `TenantId`, `UPC` use Lombok `@Value` (all fields final, no setters). Mutate them by creating new instances — e.g. `Budget.addBurn()` returns a new `Money`.

## Do NOT Do This

**Do not put business logic in the controller.** `CampaignController` must only translate HTTP requests to commands and delegate to `CampaignApplicationService`. Validation (`FundingValidator`, `ConflictResolver`) lives in the domain — not the controller.

**Do not publish events directly from domain methods.** `Campaign.publish()` adds a `CampaignPublished` to `domainEvents` — it does not call Redis. Only `EventPublisher` touches Redis, and only after the application service has saved the aggregate. Publish-before-save creates phantom events on save failures.

**Do not use the Campaign's `status` field to check if a campaign is active inside another service.** Eligibility and Redemption must call the Eligibility Service — never call Campaign Service for eligibility decisions. Eligibility Service is the single source of truth for whether a campaign's offer is applicable to a customer.
