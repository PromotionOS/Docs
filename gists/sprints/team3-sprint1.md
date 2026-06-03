# Team 3 — Redemption Service — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, redemption-service-guide, adr-recorder, pr-reviewer, redemption-analytics-contract

---

## Context

The Redemption Service is the system of record for confirmed redemptions. Sprint 1 focuses on the single most critical piece — idempotent redemption confirmation. Get this right and everything else builds cleanly on top.

---

## Sprint 1 Focus

**Idempotency guard + redemption confirmation. One flow. Full SDD process.**

---

## Requirements

### Requirement 1 — Idempotency Guard

Implement `IdempotencyGuard.check()` and `IdempotencyGuard.register()`:

1. Every request carries an `idempotencyKey` from the POS system
2. Check if key already exists for this tenant → if yes, return `409 DUPLICATE_REDEMPTION` with original redemptionId
3. Register the key atomically before confirming
4. Race condition: use DB-level unique constraint on `(idempotency_key, tenant_id)`
5. Same key from different tenant → NOT a duplicate → proceed normally

### Requirement 2 — Redemption Confirmation

Implement `Redemption.confirm()`:

1. Record: customerId, campaignId, discountApplied, cartTotal, storeId, division, tenantId
2. `redeemedAt` = server timestamp — never client's timestamp
3. Redemption is immutable once written — no updates, no deletes
4. Return 200 with redemptionId, status: CONFIRMED, redeemedAt

### Requirement 3 — OfferRedeemed Event

After confirmation:

1. Publish `OfferRedeemed` to Redis channel `promotionos.redemption.redeemed`
2. Must match exact schema in `contract-redemption-analytics.md`
3. Publish AFTER redemption is persisted — never before

---

## Domain Rules

From ubiquitous-language.md:
- A Redemption is immutable once confirmed
- Idempotency is enforced per Tenant — same key different tenant is NOT a duplicate
- `redeemedAt` is server-side always

---

## Contracts Involved

`contract-redemption-analytics.md` — OfferRedeemed event schema
Analytics Service is pre-built against this. Do not deviate from the schema.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 7 | idempotencyKey: pos-store-chi-001-txn-88821 (duplicate) | 409 DUPLICATE_REDEMPTION |
| 17 | same key, tenantId: tenant-other-001 | 200 success |
| — | valid new redemption | 200, OfferRedeemed published |
| — | two simultaneous requests same key | exactly one 200, one 409 |

---

## Sprint 1 ADR Topics

Record an ADR for:
- Race condition handling — DB constraint vs application-level lock — why DB constraint wins
