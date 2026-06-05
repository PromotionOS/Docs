---
name: vendor-service-guide
description: Codebase guide for Vendor Service — folder structure, pre-built vs stub, domain model, approval flow, event wiring, forecast patterns
metadata:
  type: reference
---

# Vendor Service — Codebase Guide

## Quick Facts
- Language: Java 17 / Spring Boot 3.x
- Framework: Spring Boot (Maven)
- Port: 8086
- DB Schema: `vendor` (PostgreSQL via Flyway)
- Status: Pure skeleton — all application service methods throw `NotImplementedException`

## Folder Structure

```
vendor-service/
├── src/main/java/com/promotionos/vendor/
│   ├── VendorServiceApplication.java
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Vendor.java                ← Aggregate Root — register(), suspend(), deactivate()
│   │   │   ├── VendorStatus.java          ← ACTIVE, INACTIVE, SUSPENDED
│   │   │   ├── FundingProposal.java       ← Entity — draft(), submit(), approve(), reject()
│   │   │   ├── ProposalStatus.java        ← DRAFT, SUBMITTED, APPROVED, REJECTED
│   │   │   ├── Approval.java              ← Entity — one approver decision per proposal
│   │   │   ├── ApprovalDecision.java      ← APPROVED, REJECTED
│   │   │   ├── Forecast.java              ← Value Object — predictedLift, burnVelocity, exhaustionDate
│   │   │   └── ApprovalThreshold.java     ← Value Object — $50,000 default; requires(Money)
│   │   ├── service/
│   │   │   ├── FundingProposalValidator.java  ← interface — validate(proposal, vendor)
│   │   │   ├── ForecastCalculator.java        ← interface — calculate(campaignId, tenantId)
│   │   │   └── ApprovalGateway.java           ← interface — requiresApproval(totalAmount)
│   │   ├── repository/
│   │   │   ├── VendorRepository.java
│   │   │   ├── FundingProposalRepository.java
│   │   │   ├── ApprovalRepository.java
│   │   │   └── ForecastRepository.java
│   │   └── event/
│   │       ├── FundingProposalSubmitted.java  ← record — published when proposal submitted
│   │       ├── FundingApproved.java           ← record — published when proposal approved
│   │       └── FundingRejected.java           ← record — published when proposal rejected
│   ├── application/
│   │   └── VendorApplicationService.java      ← ALL STUBS: registerVendor, submitFundingProposal,
│   │                                               approveProposal, rejectProposal, getForecast,
│   │                                               calculateBurnVelocity, handleCampaignPublished,
│   │                                               handleClaimSubmitted, handleBudgetExhausted
│   ├── infrastructure/
│   │   ├── repository/
│   │   │   ├── VendorRepositoryImpl.java
│   │   │   ├── FundingProposalRepositoryImpl.java
│   │   │   ├── ApprovalRepositoryImpl.java
│   │   │   └── ForecastRepositoryImpl.java
│   │   └── event/
│   │       ├── EventPublisher.java              ← publishes to 3 Redis channels
│   │       ├── CampaignPublishedConsumer.java   ← calls handleCampaignPublished
│   │       ├── ClaimSubmittedConsumer.java      ← calls handleClaimSubmitted
│   │       └── BudgetExhaustedConsumer.java     ← calls handleBudgetExhausted
│   └── api/
│       └── VendorController.java               ← stubs throw NotImplementedException
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V1__vendor_schema.sql               ← vendors, funding_proposals, approvals, forecasts
│       └── V2__seed_test_data.sql              ← seeds Frito-Lay + Coca-Cola vendors
└── src/test/java/com/promotionos/vendor/
    └── VendorContractTest.java
```

## What's Pre-built

`Vendor.java` — aggregate with `register()`, `suspend()`, `deactivate()` — fully implemented.

`FundingProposal.java` — entity with `draft()`, `submit()`, `approve()`, `reject(reason)` — fully implemented.

`Approval.java` and `ApprovalDecision.java` — entity and enum — fully defined.

`Forecast.java` — immutable value object — fields defined.

`ApprovalThreshold.java` — `defaultThreshold()` returns $50,000; `requires(Money)` is implemented.

`EventPublisher` — shell wired to three channels: `funding-proposal-submitted`, `funding-approved`, `funding-rejected`.

All three event consumers — wired in context config and running; handlers are stubs.

DB seed: two vendors pre-seeded — Frito-Lay (`a1b2c3d4-0000-0000-0000-000000000001`) and Coca-Cola (`a1b2c3d4-0000-0000-0000-000000000002`) for `tenant-kroger-001`.

`GET /health` returns `{"status":"ok","service":"vendor-service"}`.

## What You Implement (by sprint)

**Sprint 1**
- `VendorApplicationService.registerVendor(RegisterVendorCommand)`:
  - Call `Vendor.register(tenantId, name, contactEmail)`
  - Save via `vendorRepository`
  - Return saved vendor
- `VendorApplicationService.submitFundingProposal(SubmitProposalCommand)`:
  1. Load vendor, assert `VendorStatus.ACTIVE`
  2. `FundingProposal.draft(campaignId, vendorId, tenantId, vendorShare, krogerShare)`
  3. `proposalValidator.validate(proposal, vendor)`
  4. `proposal.submit()`
  5. `proposalRepository.save(proposal)`
  6. `eventPublisher.publish(new FundingProposalSubmitted(...))`
- `POST /vendors` — wire `registerVendor`
- `GET /vendors/:id` — wire `vendorRepository.findById()`
- `POST /vendors/:id/proposals` — wire `submitFundingProposal`
- Implement `FundingProposalValidator` — check `vendorShare + krogerShare == 100` and vendor is ACTIVE
- Implement `ApprovalGateway` — check `campaignTotal >= ApprovalThreshold.defaultThreshold().amount`
- `handleCampaignPublished(event)` — check `approvalGateway.requiresApproval(totalAmount)`; if true, initiate proposal draft and notify vendor (coordinate exact flow with Campaign Service Team 1)

**Sprint 2**
- `VendorApplicationService.approveProposal(proposalId, tenantId)`:
  1. Load proposal, assert `SUBMITTED`
  2. `proposal.approve()`
  3. Create `Approval` record with `ApprovalDecision.APPROVED`
  4. Save both
  5. Publish `FundingApproved` event — Campaign Service unblocks on this
- `VendorApplicationService.rejectProposal(proposalId, reason, tenantId)`:
  1. Load proposal, assert `SUBMITTED`
  2. `proposal.reject(reason)`
  3. Create `Approval` record with `REJECTED` + reason
  4. Save both
  5. Publish `FundingRejected` event
- `PUT /proposals/:id/approve` — wire `approveProposal`
- `PUT /proposals/:id/reject` — wire `rejectProposal`

**Sprint 3**
- Implement `ForecastCalculator` — `predictedLift = categoryBaseline × seasonFactor × promoTypeFactor`; `burnVelocityPerDay = burnedAmount / daysSinceStart`
- `VendorApplicationService.getForecast(campaignId, tenantId)` — load cached or call `forecastCalculator.calculate()`
- `VendorApplicationService.calculateBurnVelocity(campaignId, tenantId)`
- `GET /forecasts?campaignId=X` — wire `getForecast`
- `handleClaimSubmitted(event)` — update vendor claim records
- `handleBudgetExhausted(event)` — update vendor campaign status

## Domain Model

```
Vendor (Aggregate Root)
  vendorId: UUID
  tenantId: String
  name: String
  contactEmail: String
  status: VendorStatus      ← ACTIVE | INACTIVE | SUSPENDED
  createdAt: Instant

FundingProposal (Entity)
  id: UUID
  campaignId: UUID
  vendorId: UUID
  tenantId: String
  vendorShare: BigDecimal   ← percentage, e.g. 60.0
  krogerShare: BigDecimal   ← percentage, e.g. 40.0
  status: ProposalStatus    ← DRAFT → SUBMITTED → APPROVED | REJECTED
  rejectionReason: String
  submittedAt: Instant
  decidedAt: Instant

Approval (Entity)
  id: UUID
  proposalId: UUID
  tenantId: String
  approverRole: String      ← e.g. "FINANCE_DIRECTOR"
  decision: ApprovalDecision  ← APPROVED | REJECTED
  reason: String
  decidedAt: Instant

Forecast (Value Object — immutable)
  campaignId: UUID
  predictedLift: double
  predictedLiftPct: double
  burnVelocityPerDay: double
  estimatedExhaustionDate: LocalDate
  confidence: double         ← 0.0–1.0

ApprovalThreshold (Value Object)
  amount: Money             ← default $50,000
  requires(campaignTotal: Money): boolean
```

## Publishing Events

Three events are published, via `EventPublisher.publish(DomainEvent)`:

```java
// Channels (from application.yml):
// FundingProposalSubmitted → promotionos.vendor.proposal.submitted
// FundingApproved          → promotionos.vendor.funding.approved
// FundingRejected          → promotionos.vendor.funding.rejected

eventPublisher.publish(new FundingApproved(
    UUID.randomUUID().toString(),
    tenantId,
    campaignId,
    Instant.now().toString(),
    1,
    new FundingApproved.Payload(
        proposalId.toString(),
        vendorId.toString(),
        proposal.getVendorShare().doubleValue(),
        proposal.getKrogerShare().doubleValue(),
        Instant.now().toString()
    )
));
```

`FundingApproved` is the critical event — Campaign Service consumes it to transition a campaign from `PENDING_APPROVAL → ACTIVE` and then emit `CampaignPublished`.

## Consuming Events

Three consumers in `infrastructure/event/`, all following the Spring `MessageListener` pattern:

| Consumer file | Channel subscribed | Calls |
|---|---|---|
| `CampaignPublishedConsumer` | `promotionos.campaign.published` | `handleCampaignPublished(event)` |
| `ClaimSubmittedConsumer` | `promotionos.claim.submitted` | `handleClaimSubmitted(event)` |
| `BudgetExhaustedConsumer` | `promotionos.analytics.budget.exhausted` | `handleBudgetExhausted(event)` |

Channel names configured in `application.yml` under `promotionos.redis.channels` (consumed channels have `-in` suffix):
- `campaign-published-in: promotionos.campaign.published`
- `claim-submitted-in: promotionos.claim.submitted`
- `budget-exhausted-in: promotionos.analytics.budget.exhausted`

## Key Interfaces You Must Implement

```java
// domain/service/FundingProposalValidator.java
void validate(FundingProposal proposal, Vendor vendor);
// Throw DomainException("INVALID_SPLIT") if vendorShare + krogerShare != 100
// Throw DomainException("VENDOR_NOT_ACTIVE") if vendor.status != ACTIVE

// domain/service/ApprovalGateway.java
boolean requiresApproval(Money campaignTotal);
// Return true if campaignTotal.amount >= 50000 (uses ApprovalThreshold.defaultThreshold())

// domain/service/ForecastCalculator.java
Forecast calculate(UUID campaignId, String tenantId);
// predictedLift = categoryBaseline × seasonFactor × promoTypeFactor
// burnVelocityPerDay = burnedAmount / daysSinceStart (use Analytics data or redemption count)

// domain/repository/VendorRepository.java
Vendor findById(UUID vendorId);
List<Vendor> findByTenantId(String tenantId);
Vendor save(Vendor vendor);

// domain/repository/FundingProposalRepository.java
FundingProposal findByCampaignId(UUID campaignId);
List<FundingProposal> findByVendorId(UUID vendorId);
FundingProposal save(FundingProposal proposal);

// domain/repository/ApprovalRepository.java
List<Approval> findByProposalId(UUID proposalId);
Approval save(Approval approval);
```

## Running Locally

```bash
export DB_URL=jdbc:postgresql://localhost:5432/vendor-db
export REDIS_URL=redis://localhost:6379
export TENANT_ID=tenant-kroger-001
mvn spring-boot:run
# Health: http://localhost:8086/health
```

Run tests:
```bash
./scripts/test.sh
# or: mvn test -Dspring.profiles.active=test
```

## Patterns In This Codebase

**The approval flow is a two-service choreography.** Campaign Service transitions to `PENDING_APPROVAL` before calling `POST /vendors/:id/proposals`. Vendor Service creates the proposal, publishes `FundingProposalSubmitted`. Finance director calls `PUT /proposals/:id/approve`. Vendor Service publishes `FundingApproved`. Campaign Service consumes that event and transitions `PENDING_APPROVAL → ACTIVE`. The services do not call each other synchronously during approval — they communicate only via events. Coordinate with Team 1 on the exact trigger for this flow.

**Forecast is a value object — recalculate, never mutate.** `Forecast` is immutable (`@Value`). When burn velocity changes, calculate a new `Forecast` and replace the cached record. `ForecastRepository.save(campaignId, forecast)` is an upsert — it replaces any existing forecast for that campaign.

**vendorShare + krogerShare must always equal 100.** This invariant is enforced by `FundingProposalValidator` and is a domain rule stated in the ubiquitous language. It is not a UI validation — it is a domain constraint. Reject any proposal where the shares don't sum to 100 with a `DomainException("INVALID_SPLIT")`.

## Do NOT Do This

**Do not publish `FundingApproved` without saving the `Approval` record first.** The same publish-after-save rule applies here as in Campaign Service. If you publish before the DB write commits, Campaign Service may transition to ACTIVE for a proposal that was never persisted. Save `FundingProposal` (with `APPROVED` status) and `Approval` entity before calling `eventPublisher.publish()`.

**Do not confuse `FundingProposal` with `Funding`.** `Funding` is a Campaign Service concept — it is the confirmed funding agreement on a campaign. `FundingProposal` is a Vendor Service concept — it is a pending negotiation. When the Vendor Service publishes `FundingApproved`, Campaign Service uses that to confirm its internal `Funding` entity. These are different objects in different bounded contexts.

**Do not skip `ApprovalGateway` for campaigns below the threshold.** Campaigns with `totalAmount < $50,000` do not need approval and bypass the proposal flow entirely — Campaign Service handles them directly by calling `publish()`. Vendor Service should only be involved when `approvalGateway.requiresApproval()` returns true. Do not route all campaigns through the approval flow to "be safe" — it will block every campaign launch.
