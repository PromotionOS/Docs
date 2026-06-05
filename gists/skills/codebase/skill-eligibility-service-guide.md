---
name: eligibility-service-guide
description: Codebase guide for Eligibility Service — folder structure, pre-built vs stub, domain model, event wiring, ACL translation, implementation patterns
metadata:
  type: reference
---

# Eligibility Service — Codebase Guide

## Quick Facts
- Language: Java 17 / Spring Boot 3.x
- Framework: Spring Boot (Maven)
- Port: 8082
- DB Schema: `eligibility` (PostgreSQL via Flyway)
- Status: Skeleton only — all business logic throws `NotImplementedException`

## Folder Structure

```
eligibility-service/
├── src/main/java/com/promotionos/eligibility/
│   ├── EligibilityServiceApplication.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── EligibilityRule.java       ← Aggregate Root (one per campaign)
│   │   │   ├── Exclusion.java             ← value object — categoryId, upcCode, reason
│   │   │   ├── Threshold.java             ← value object — type, spendValue, quantityValue
│   │   │   ├── ThresholdType.java         ← SPEND, QUANTITY
│   │   │   ├── EligibilityResult.java     ← value object — eligible, campaignId, reason
│   │   │   ├── CustomerProfile.java       ← value object — customerId, loyaltyTier, segments, division, annualSpend
│   │   │   ├── Cart.java                  ← value object — total, upcCodes
│   │   │   ├── TenantId.java              ← value object — wraps String
│   │   │   └── UPC.java                   ← value object — wraps String code
│   │   ├── service/
│   │   │   ├── RuleEngine.java            ← interface — evaluate(rule, customer, cart)
│   │   │   └── SegmentMatcher.java        ← interface — matches(customer, segmentRestriction)
│   │   └── repository/
│   │       └── EligibilityRuleRepository.java  ← interface
│   ├── application/
│   │   └── EligibilityApplicationService.java  ← all stubs; this is where you code
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   └── EligibilityRuleRepositoryImpl.java
│   │   ├── client/
│   │   │   ├── CatalogServiceClient.java   ← calls catalog-customer-service for UPC/category lookup
│   │   │   └── CampaignServiceClient.java  ← calls campaign-service for /campaigns/{id}/summary
│   │   └── event/
│   │       ├── CampaignPublishedConsumer.java   ← wired, calls eligibilityService.loadRules()
│   │       ├── CampaignPausedConsumer.java      ← wired, calls eligibilityService.removeRules()
│   │       ├── CatalogItemExcludedConsumer.java ← wired, calls eligibilityService.updateExclusions()
│   │       ├── SegmentUpdatedConsumer.java      ← wired, invalidates customer cache
│   │       └── ACL.java                         ← STUB — translate() throws NotImplementedException
│   └── api/
│       └── EligibilityController.java           ← stubs return 501
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V1__eligibility_schema.sql
│       └── V2__seed_test_data.sql
└── src/test/java/com/promotionos/eligibility/
    └── EligibilityContractTest.java
```

## What's Pre-built

All four event consumers are wired and running. They deserialise incoming Redis messages and call the appropriate `EligibilityApplicationService` method. The consumers catch `NotImplementedException` and log it — the service boots cleanly even while all handlers are stubs.

All repository impl skeletons exist in `infrastructure/repository/`.

The `CatalogServiceClient` and `CampaignServiceClient` HTTP client shells exist in `infrastructure/client/`.

`GET /health` returns 200 OK.

## What You Implement (by sprint)

**Sprint 1**
- `ACL.translate(CampaignPublishedEvent)` — translates campaign event into `EligibilityRule`
  - `payload.stackLimit` → `rule.stackLimit`
  - `payload.segmentRestriction` → `rule.segmentRestriction`
  - `payload.exclusions` → `rule.exclusions` (list of `Exclusion` value objects)
  - `payload.geoScope` → `rule.geoScope`
  - `payload.dateStart/dateEnd` → `rule.dateStart/dateEnd`
- `EligibilityApplicationService.loadRules(event)` — call ACL, save rule to repository
- `EligibilityApplicationService.removeRules(campaignId, tenantId)` — deactivate rule on CampaignPaused
- `POST /eligibility/check` — wire into `eligibilityService.check()` with segment matching and status filter

**Sprint 2**
- Implement `RuleEngine.evaluate()` — apply exclusions, threshold, geo, stack rules
- `EligibilityApplicationService.getOffers()` — find all active rules for tenant, evaluate each
- `GET /offers` — wire into `eligibilityService.getOffers()`
- `EligibilityApplicationService.updateExclusions()` — handle `CatalogItemExcluded` events

**Sprint 3**
- `GET /eligibility/audit` — return audit log of eligibility decisions per campaign

## Domain Model

```
EligibilityRule (Aggregate Root) — one per Campaign per Tenant
  id: UUID
  campaignId: UUID
  tenantId: TenantId
  stackLimit: int           ← from CampaignPublished.payload.stackLimit
  segmentRestriction: String
  geoScope: List<String>
  exclusions: List<Exclusion>
  threshold: Threshold
  active: boolean
  dateStart: LocalDate
  dateEnd: LocalDate

Exclusion (Value Object)
  categoryId: String
  upcCode: String
  reason: String

Threshold (Value Object)
  type: ThresholdType       ← SPEND or QUANTITY
  spendValue: BigDecimal
  quantityValue: Integer

EligibilityResult (Value Object)
  eligible: boolean
  campaignId: UUID
  offerType: String
  discountApplied: BigDecimal
  reason: String            ← SEGMENT_MISMATCH | THRESHOLD_NOT_MET | UPC_EXCLUDED |
                               GEO_MISMATCH | STACK_EXCEEDED | CAMPAIGN_INACTIVE

CustomerProfile (Value Object)  — built from Catalog/Customer Service response
  customerId: UUID
  loyaltyTier: String
  segments: List<String>
  division: String
  annualSpend: double

Cart (Value Object)
  total: double
  upcCodes: List<String>
```

## Publishing Events

This service does not publish any events. It is a pure consumer.

## Consuming Events

All four consumers are in `infrastructure/event/`. They follow the same pattern:

```java
// pattern in every consumer:
public void onMessage(Message message, byte[] pattern) {
    try {
        XxxEvent event = objectMapper.readValue(message.getBody(), XxxEvent.class);
        eligibilityService.handleXxx(event);
    } catch (NotImplementedException e) {
        // expected during skeleton phase — log and continue
    } catch (Exception e) {
        // log and continue — consumers must not crash
    }
}
```

| Consumer file | Channel subscribed | Calls |
|---|---|---|
| `CampaignPublishedConsumer` | `promotionos.campaign.published` | `loadRules(event)` |
| `CampaignPausedConsumer` | `promotionos.campaign.paused` | `removeRules(campaignId, tenantId)` |
| `CatalogItemExcludedConsumer` | `promotionos.catalog.item.excluded` | `updateExclusions(event)` |
| `SegmentUpdatedConsumer` | `promotionos.customer.segment.updated` | cache invalidation (Sprint 2) |

Redis channels are configured in `application.yml` under `promotionos.redis.channels`.

## Key Interfaces You Must Implement

```java
// domain/service/RuleEngine.java
EligibilityResult evaluate(EligibilityRule rule, CustomerProfile customer, Cart cart);
// Check in order: active+dateRange, segment, geo, exclusions, threshold, stack
// Return EligibilityResult.ineligible(campaignId, "SEGMENT_MISMATCH") etc. on failure
// Return EligibilityResult.eligible(campaignId, offerType, discount) on pass

// domain/service/SegmentMatcher.java
boolean matches(CustomerProfile customer, String segmentRestriction);
// segmentRestriction null or empty → matches all customers
// otherwise: customer.segments must contain the segmentRestriction value

// domain/repository/EligibilityRuleRepository.java
List<EligibilityRule> findActive(TenantId tenantId);
Optional<EligibilityRule> findByCampaignId(UUID campaignId, TenantId tenantId);
EligibilityRule save(EligibilityRule rule);
void deactivate(UUID campaignId, TenantId tenantId);

// infrastructure/event/ACL.java
EligibilityRule translate(CampaignPublishedEvent event);
// This is infrastructure, not domain — the ACL lives here intentionally
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/eligibility-db
export REDIS_URL=redis://localhost:6379
export CATALOG_SERVICE_URL=http://localhost:8084
export CAMPAIGN_SERVICE_URL=http://localhost:8081
export TENANT_ID=tenant-kroger-001
mvn spring-boot:run
# Health: http://localhost:8082/health
```

Run tests:
```bash
./scripts/test.sh
```

## Patterns In This Codebase

**ACL in the infrastructure layer:** The `ACL.java` file sits in `infrastructure/event/`, not in `domain/`. That placement is intentional — ACL is an infrastructure concern (it deals with external event formats). The domain model (`EligibilityRule`) knows nothing about `CampaignPublishedEvent`. The ACL translates between them.

**Rule Engine evaluates in a fixed order:** Evaluate active/date check first. If a rule is inactive, return `CAMPAIGN_INACTIVE` immediately — don't run other checks. The order matters for Sprint 2: active → segment → geo → exclusions → threshold → stack.

**EligibilityResult is a value object:** It is created by the `RuleEngine` and returned directly — never stored in the DB. Only audit logs are persisted (Sprint 3). The result is stateless and immutable.

## Do NOT Do This

**Do not store loyalty tier in the database.** Loyalty tier is calculated at runtime from `annualSpend` — it is never stored. When you fetch a `CustomerProfile` from Catalog Service, call `customer.getLoyaltyTier()` to get the current tier. Stored tiers become stale immediately.

**Do not call Campaign Service to check campaign status.** Eligibility Service maintains its own `EligibilityRule` state driven by `CampaignPublished` and `CampaignPaused` events. If you make a synchronous call to Campaign Service inside `evaluate()`, you couple two services in the hot path. The rule's `active` flag is your source of truth.

**Do not translate event fields by name-matching strings.** Use the ACL's explicit field-to-field mapping documented in `contract-campaign-eligibility.md`. Campaign Service uses `payload.upcScope` (a list of strings); Eligibility uses `rule.exclusions` (a list of `Exclusion` value objects). They are not the same thing — map deliberately.
