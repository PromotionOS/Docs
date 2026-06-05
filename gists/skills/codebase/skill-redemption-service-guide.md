---
name: redemption-service-guide
description: Codebase guide for Redemption Service — folder structure, pre-built vs stub, domain model, event wiring, idempotency, claim generation patterns
metadata:
  type: reference
---

# Redemption Service — Codebase Guide

## Quick Facts
- Language: Java 17 / Spring Boot 3.x
- Framework: Spring Boot (Maven)
- Port: 8083
- DB Schema: `redemption` (PostgreSQL via Flyway)
- Status: Skeleton only — all business logic throws `NotImplementedException`

## Folder Structure

```
redemption-service/
├── src/main/java/com/promotionos/redemption/
│   ├── RedemptionServiceApplication.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Redemption.java        ← Aggregate Root — immutable after confirm()
│   │   │   ├── RedemptionStatus.java  ← CONFIRMED, CLAIMED
│   │   │   ├── Claim.java             ← Entity inside Redemption aggregate
│   │   │   ├── ClaimStatus.java       ← PENDING, SUBMITTED
│   │   │   ├── IdempotencyKey.java    ← value object — format: pos-{storeId}-txn-{txnId}
│   │   │   ├── CustomerId.java        ← value object — wraps UUID
│   │   │   ├── Money.java             ← value object — BigDecimal + "USD"
│   │   │   └── TenantId.java          ← value object — wraps String
│   │   ├── service/
│   │   │   ├── IdempotencyGuard.java  ← interface — duplicate detection per tenant
│   │   │   └── ClaimGenerator.java    ← interface — generates Claim from Redemption
│   │   └── repository/
│   │       └── RedemptionRepository.java  ← interface
│   ├── application/
│   │   └── RedemptionApplicationService.java  ← all stubs; this is where you code
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   └── RedemptionRepositoryImpl.java
│   │   ├── client/
│   │   │   ├── EligibilityServiceClient.java  ← POST /eligibility/check
│   │   │   └── CampaignServiceClient.java     ← GET /campaigns/{id}/summary
│   │   └── event/
│   │       └── EventPublisher.java            ← publishes OfferRedeemed, ClaimSubmitted
│   └── api/
│       └── RedemptionController.java          ← stubs return 501
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V1__redemption_schema.sql
│       └── V2__seed_test_data.sql
└── src/test/java/com/promotionos/redemption/
    └── RedemptionContractTest.java
```

## What's Pre-built

`Redemption.java` aggregate is fully implemented — `confirm()` factory method, `attachClaim()`, `pullDomainEvents()`.

`Claim.java` entity fields are defined.

`EligibilityServiceClient` — shell exists with `check()` method signature.

`CampaignServiceClient` — shell exists with `getSummary()` method signature.

`EventPublisher` — shell exists; publishes to `promotionos.redemption.redeemed` and `promotionos.redemption.claim.submitted`.

All value objects (`IdempotencyKey`, `CustomerId`, `Money`) are implemented.

## What You Implement (by sprint)

**Sprint 1**
- `RedemptionApplicationService.redeem(RedeemCommand)` — follow this exact order:
  1. `idempotencyGuard.isDuplicate(key, tenantId)` — if true, return 409 with original redemption ID
  2. `eligibilityClient.check(campaignId, tenantId, customerId, cartTotal, cartUPCs)` — if ineligible, return 400
  3. `Redemption.confirm(...)` — creates aggregate and adds `OfferRedeemed` to domain events
  4. `idempotencyGuard.register(key, tenantId, redemption.getId())`
  5. `redemptionRepository.save(redemption)`
  6. `eventPublisher.publishAll(redemption.pullDomainEvents())`
- `POST /redeem` — wire into `redemptionService.redeem()`
- Implement `IdempotencyGuard` — backed by a DB table keyed on `(idempotency_key, tenant_id)`

**Sprint 2**
- `RedemptionApplicationService.findById()` — load from repository
- `GET /redemptions/{id}` — wire into `findById`
- `RedemptionApplicationService.processPendingClaims()` — scheduled `@Scheduled(fixedDelay = 60000)`:
  - Find redemptions where `scheduledAt <= now()` and no claim exists
  - Call `claimGenerator.generate(redemption, vendorShare, vendorId)`
  - Call `redemption.attachClaim(claim)`
  - Save and publish `ClaimSubmitted`
- Implement `ClaimGenerator` — `deduction = discountApplied × vendorShare%`; schedule T+24h

**Sprint 3**
- `RedemptionApplicationService.findByCustomer()` — paginated
- `GET /redemptions` — wire with `?customerId=` + `?page=`
- `GET /redemptions/stats` — aggregate by campaign

## Domain Model

```
Redemption (Aggregate Root) — immutable after confirm()
  id: UUID
  tenantId: TenantId
  idempotencyKey: IdempotencyKey
  customerId: CustomerId
  campaignId: UUID
  discountApplied: Money
  cartTotal: Money
  storeId: String
  division: String
  redeemedAt: Instant          ← always set server-side in confirm(), never from request
  status: RedemptionStatus
  claim: Claim                 ← null until attachClaim()
  domainEvents: List<DomainEvent>

Claim (Entity inside Redemption)
  id: UUID
  redemptionId: UUID
  tenantId: TenantId
  vendorId: String
  amount: Money
  deduction: Money             ← discountApplied × vendorShare%
  status: ClaimStatus
  scheduledAt: Instant         ← redeemedAt + 24h
  submittedAt: Instant

IdempotencyKey (Value Object)  — format: pos-{storeId}-txn-{transactionId}
CustomerId (Value Object)      — wraps UUID
Money (Value Object)           — BigDecimal amount + "USD"
```

## Publishing Events

Two events are published by this service:

```java
// EventPublisher sends to these channels (configured in application.yml):
// OfferRedeemed   → promotionos.redemption.redeemed
// ClaimSubmitted  → promotionos.redemption.claim.submitted

// Always via application service, after save:
Redemption saved = redemptionRepository.save(redemption);
eventPublisher.publishAll(saved.pullDomainEvents());
```

`OfferRedeemed` is added to domain events inside `Redemption.confirm()`.
`ClaimSubmitted` is added inside `Redemption.attachClaim(claim)`.

Never call `eventPublisher` before calling `redemptionRepository.save()`.

## Consuming Events

This service does not consume any Redis events. It calls Eligibility Service and Campaign Service via HTTP.

HTTP calls made by this service:
- `POST http://eligibility-service/eligibility/check` — via `EligibilityServiceClient`
- `GET http://campaign-service/campaigns/{id}/summary` — via `CampaignServiceClient` (for vendorShare when generating claims)

Both clients are configured via:
```yaml
promotionos.services.eligibility-url: ${ELIGIBILITY_SERVICE_URL:http://localhost:8082}
promotionos.services.campaign-url: ${CAMPAIGN_SERVICE_URL:http://localhost:8081}
```

## Key Interfaces You Must Implement

```java
// domain/service/IdempotencyGuard.java
boolean isDuplicate(IdempotencyKey key, TenantId tenantId);
void register(IdempotencyKey key, TenantId tenantId, UUID redemptionId);
UUID getOriginalRedemptionId(IdempotencyKey key, TenantId tenantId);
// CRITICAL: idempotency is per-tenant. Same key from two tenants is NOT a duplicate.

// domain/service/ClaimGenerator.java
Claim generate(Redemption redemption, double vendorShare, String vendorId);
// deduction = redemption.discountApplied.multiply(vendorShare / 100)
// scheduledAt = redemption.redeemedAt + 24 hours
void scheduleT24(UUID redemptionId);

// domain/repository/RedemptionRepository.java
Redemption save(Redemption redemption);
Optional<Redemption> findById(UUID id, TenantId tenantId);
List<Redemption> findByCustomer(UUID customerId, TenantId tenantId, int page, int size);
List<Redemption> findByCampaign(UUID campaignId, TenantId tenantId, int page, int size);
List<Redemption> findPendingClaims(Instant before);
```

## Running Locally

```bash
export DB_URL=postgresql://localhost:5432/redemption-db
export REDIS_URL=redis://localhost:6379
export ELIGIBILITY_SERVICE_URL=http://localhost:8082
export CAMPAIGN_SERVICE_URL=http://localhost:8081
export TENANT_ID=tenant-kroger-001
mvn spring-boot:run
# Health: http://localhost:8083/health
```

Run tests:
```bash
./scripts/test.sh
```

## Patterns In This Codebase

**Idempotency is per tenant, enforced before eligibility.** Check `isDuplicate` as the very first step in `redeem()`. If it is a duplicate, return 409 with the original `redemptionId` immediately — do not call Eligibility Service. A duplicate from a different tenant with the same key is NOT a duplicate: the guard checks `(key, tenantId)` together.

**Redemption is immutable after confirm().** The `Redemption.confirm()` factory method is the only way to create a Redemption. After that, the only allowed mutation is `attachClaim()` which moves status from `CONFIRMED` to `CLAIMED`. Do not add setters. Do not allow status rollback.

**redeemedAt is always server-side.** The POS system sends the redemption request; the server sets `redeemedAt = Instant.now()` inside `Redemption.confirm()`. Never accept a timestamp from the request body for this field. This prevents replay attacks with backdated redemptions.

## Do NOT Do This

**Do not generate claims synchronously inside `redeem()`.** Claims are generated by a scheduled job (`processPendingClaims`) that runs every minute. The T+24h delay is a business rule — it exists to allow time for returns/reversals before a claim is submitted to the vendor. Generating the claim immediately in `redeem()` violates this rule.

**Do not call Eligibility Service with just `cartTotal` — include `cartUPCs`.** The Eligibility Service needs the full UPC list to check item-level exclusions. A call with only `cartTotal` will miss exclusion checks and will pass eligibility incorrectly for carts containing excluded products.

**Do not trust `discountApplied` from the POS request.** The `discountApplied` value must come from the `EligibilityResult.discountApplied` field returned by the Eligibility Service — not from the incoming request body. The POS may send an incorrect or manipulated discount amount.
