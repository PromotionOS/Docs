# Team 3 — Redemption Service
## Type: Build via SDD

---

## Context

The Redemption Service records the confirmed application of an offer to a customer transaction. It is the system of record for what actually happened — every redemption is immutable once confirmed. It also generates vendor claims automatically 24 hours post-redemption.

The domain model and interfaces are defined. Your job is to implement the business logic.

**Your job:**
1. Read the domain model using the redemption-service-guide skill
2. Read the contracts from eligibility-redemption-contract and redemption-analytics-contract skills
3. Write a spec using the sdd-spec-writer skill
4. Implement IdempotencyGuard, Redemption.confirm(), and ClaimGenerator
5. Record an ADR using the adr-recorder skill
6. Validate against redemption scenarios
7. Open a PR

---

## Business Rules You Must Implement

### Idempotency
1. Every redemption request carries an `idempotencyKey` from the POS system
2. If a request arrives with a key already registered for this tenant — return `409 DUPLICATE_REDEMPTION`
3. The key must be checked **before** any redemption is written — use a database-level unique constraint
4. Race conditions must be handled — two simultaneous requests with the same key must result in exactly one redemption

### Redemption Confirmation
5. A confirmed redemption is **immutable** — no updates, no deletes, only reads
6. Record: customerId, campaignId, discountApplied, cartTotal, storeId, division, redeemedAt, tenantId
7. `redeemedAt` is the server timestamp at confirmation — never trust the client timestamp
8. Status starts as `CONFIRMED` — moves to `CLAIMED` only when claim is submitted

### Claim Generation
9. A claim is automatically generated 24 hours after redemption confirmation
10. Claim amount = `discountApplied`
11. Deduction amount = `discountApplied × vendorShare%` (get funding split from Campaign Service)
12. Claim status starts as `PENDING` → moves to `SUBMITTED` when sent to vendor

### Event Publishing
13. On successful redemption → publish `OfferRedeemed` event
14. On claim submission → publish `ClaimSubmitted` event

---

## Validation Scenarios to Pass

| Scenario | Input | Expected |
|----------|-------|----------|
| 7 | redeem-003 (duplicate idempotencyKey of redeem-001) | 409 DUPLICATE_REDEMPTION |
| — | Two simultaneous requests with same key | Exactly one 200, one 409 |
| — | Valid redemption request | 200, redemption persisted, OfferRedeemed published |
| — | Valid redemption for camp-001 (60% vendor) | Claim generated at T+24hrs, deduction = 60% of discount |

---

## API Contract You Must Honour

**POST /redeem**
```json
Request:
{
  "tenantId": "string",
  "idempotencyKey": "string",
  "customerId": "string",
  "campaignId": "string",
  "discountApplied": "number",
  "cartTotal": "number",
  "storeId": "string",
  "division": "string"
}

Response (success):
{
  "redemptionId": "string",
  "status": "CONFIRMED",
  "redeemedAt": "timestamp"
}

Response (duplicate):
{
  "error": "DUPLICATE_REDEMPTION",
  "originalRedemptionId": "string"
}
```

**GET /redemptions/:id**
```json
Response:
{
  "id": "string",
  "tenantId": "string",
  "customerId": "string",
  "campaignId": "string",
  "discountApplied": "number",
  "cartTotal": "number",
  "status": "CONFIRMED | CLAIMED",
  "redeemedAt": "timestamp",
  "claim": { "id": "string", "status": "PENDING | SUBMITTED", "amount": "number" }
}
```

---

## Events You Must Publish

**OfferRedeemed**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "string",
  "customerId": "string",
  "redemptionId": "string",
  "discountApplied": "number",
  "occurredAt": "timestamp",
  "schemaVersion": 1
}
```

**ClaimSubmitted**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "string",
  "claimId": "string",
  "vendorId": "string",
  "amount": "number",
  "deduction": "number",
  "occurredAt": "timestamp",
  "schemaVersion": 1
}
```

---

## Skills to Use

- `sdd-spec-writer` — write your spec before touching code
- `ddd-guide` — understand immutability and idempotency as domain concepts
- `redemption-service-guide` — understand the skeleton
- `adr-recorder` — record your idempotency implementation decision
- `eligibility-redemption-contract` — understand what you receive from Eligibility
- `redemption-analytics-contract` — understand what Analytics expects from you
