# Skill — DDD Guide

> Type: Generic Utility
> Used by: Every team, every sprint
> Purpose: Establish how Domain Driven Design is applied in PromotionOS

---

## Why DDD

Domain Driven Design is not about folder structure. It is about putting business logic where it belongs — in the domain layer — and keeping infrastructure concerns (databases, APIs, message queues) separate from domain concerns (rules, decisions, events).

In PromotionOS, this means:
- A `Campaign` knows how to publish itself — not a service method that reaches into Campaign's fields
- A `Budget` knows when it is exhausted — not a controller that checks a number
- A `Redemption` knows it is immutable — not a database constraint alone

**The domain layer is the source of truth. Everything else serves it.**

---

## Building Blocks — What Each One Is

### Aggregate Root
The single entry point into a bounded context's domain model. All changes to the aggregate and its children go through the aggregate root. External code never directly modifies entities inside an aggregate.

**In PromotionOS:**
- `Campaign` — owns Offer, Funding, Budget
- `EligibilityRule` — owns Exclusions, Threshold
- `Redemption` — owns Claim, IdempotencyKey

**How to identify it:** If two things always change together, they belong in the same aggregate. The aggregate root is the thing that enforces the invariants of the group.

**Example — correct:**
```java
// Campaign enforces its own rules
campaign.publish(fundingValidator, conflictResolver);
// publish() validates, transitions state, raises CampaignPublished event internally
```

**Example — wrong:**
```java
// Controller reaches into Campaign's internals
if (campaign.getFunding() != null && campaign.getBudget().getTotal() > 0) {
    campaign.setStatus("ACTIVE");
    eventPublisher.publish(new CampaignPublished(campaign.getId()));
}
```

---

### Entity
An object with a unique identity that persists over time. An entity's identity does not change even if its attributes change.

**In PromotionOS:**
- `Offer` (inside Campaign aggregate) — has an ID, its type and value can change before publish
- `Funding` (inside Campaign aggregate) — has an ID, shares can be updated
- `Claim` (inside Redemption aggregate) — has an ID, status changes from PENDING to SUBMITTED

---

### Value Object
An immutable object defined entirely by its attributes. Two value objects with the same attributes are equal. Value objects have no identity.

**In PromotionOS:**
- `Money` — `{amount: 6.99, currency: "USD"}` — no ID, immutable
- `Percentage` — `{value: 95.0}` — no ID, immutable
- `DateRange` — `{startDate, endDate}` — no ID, immutable
- `TenantId` — wraps a UUID, immutable
- `LoyaltyTier` — an enum value, immutable
- `IdempotencyKey` — wraps a string, immutable

**Why it matters:** When you represent Budget as a value object with `totalAmount: Money` and `burnedAmount: Money`, the compiler helps you — you cannot accidentally put a raw number where Money is expected.

---

### Domain Service
Business logic that does not naturally belong to a single aggregate. Domain services operate on multiple aggregates or require external information to make a decision.

**In PromotionOS:**
- `FundingValidator` — validates funding before Campaign publishes (needs to know the rule, not the Campaign's state)
- `ConflictResolver` — checks UPC overlap across multiple Campaigns
- `RuleEngine` — evaluates eligibility across rules, segments, thresholds
- `BudgetBurnTracker` — tracks burn across redemptions, not a single Campaign attribute

---

### Repository
The interface between the domain layer and the database. The domain layer only knows about the repository interface — never about SQL, JPA, or Gorm directly.

**Pattern:**
```java
// Domain layer — interface only
public interface CampaignRepository {
    Campaign findById(UUID id, TenantId tenantId);
    Campaign save(Campaign campaign);
}

// Infrastructure layer — implementation
public class CampaignRepositoryImpl implements CampaignRepository {
    // JPA / SQL here — domain layer never sees this
}
```

---

### Domain Event
Something that happened in the domain that other parts of the system may care about. Events are named in past tense. They are immutable. They carry everything a consumer needs to react — no callbacks, no queries.

**In PromotionOS:**
- `CampaignPublished` — happened when campaign went live
- `OfferRedeemed` — happened when customer used an offer
- `BudgetExhausted` — happened when campaign burned 95% of budget

**How events are raised:**
```java
// Campaign raises the event internally
public void publish(FundingValidator validator, ConflictResolver resolver) {
    validator.validate(this.funding);
    resolver.checkUPCOverlap(this, activeRepository.findAll(tenantId));
    this.status = CampaignStatus.ACTIVE;
    this.raiseDomainEvent(new CampaignPublished(this.id, this.tenantId, ...));
}
// Application service dispatches the event after saving
```

---

### Anti-Corruption Layer (ACL)
A translation layer that sits at the boundary between two bounded contexts. It converts the language of the producing context into the language of the consuming context. Without ACL, domain models leak across boundaries.

**In PromotionOS — Eligibility Service ACL:**
```java
// CampaignPublished arrives from Campaign Service
// ACL translates it into Eligibility's language
public EligibilityRule translate(CampaignPublished event) {
    return EligibilityRule.builder()
        .campaignId(event.getCampaignId())
        .stackLimit(event.getPayload().getStackLimit())
        .exclusions(event.getPayload().getExclusions()
            .stream()
            .map(cat -> new Exclusion(cat, ExclusionType.CATEGORY))
            .collect(toList()))
        .build();
}
```

---

## The Layers — What Goes Where

```
┌─────────────────────────────────────┐
│          API Layer                  │  HTTP in/out. No business logic.
│  Controllers / Handlers             │  Calls Application Service only.
├─────────────────────────────────────┤
│       Application Layer             │  Orchestrates. No business logic.
│  ApplicationService                 │  Calls domain, saves via repository,
│                                     │  dispatches events after commit.
├─────────────────────────────────────┤
│         Domain Layer                │  ALL business logic lives here.
│  Aggregates, Entities, Value Objects│  No imports from infrastructure.
│  Domain Services, Domain Events     │  No SQL. No Redis. No HTTP.
│  Repository Interfaces              │
├─────────────────────────────────────┤
│      Infrastructure Layer           │  Implements domain interfaces.
│  Repository Implementations         │  SQL, JPA, Gorm, Redis publisher,
│  Event Publisher/Consumer           │  HTTP clients, ACL translators.
│  ACL                                │
└─────────────────────────────────────┘
```

---

## Bounded Context Rules

1. **Never share a database** — each bounded context has its own DB
2. **Never call another context's repository** — only call their API or consume their events
3. **Always use ACL** — translate incoming events into your own domain language
4. **Ubiquitous Language is per context** — "Customer" in Eligibility may mean something slightly different than in Redemption
5. **Events are the contract** — not shared objects, not shared libraries

---

## How to Use This Skill with Claude

When implementing a feature say:
"Use the ddd-guide skill. I am implementing [feature]. The aggregate root is [X]. The domain rule is [Y]. Help me place this logic in the correct layer."

Claude will:
- Identify where the logic belongs (aggregate, domain service, application service)
- Flag if you are putting business logic in the wrong layer
- Suggest the correct value objects to use
- Remind you to raise domain events from the aggregate, not the controller

---

## Quick Reference

| Question | Answer |
|----------|--------|
| Where does validation go? | Domain layer — inside the aggregate or domain service |
| Where does persistence go? | Infrastructure layer — repository implementation |
| Where does event publishing go? | Application layer dispatches after domain raises and repository saves |
| Where does ACL translation go? | Infrastructure layer — consumed by application layer |
| Where does HTTP routing go? | API layer — calls application service only |
| Can a controller call a repository? | No — controllers call application services only |
| Can a domain service import JPA? | No — domain layer has zero infrastructure imports |
