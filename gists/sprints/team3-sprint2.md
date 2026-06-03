# Team 3 — Redemption Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, redemption-service-guide, adr-recorder, pr-reviewer, redemption-analytics-contract

---

## Context

Sprint 1 delivered idempotent redemption confirmation and OfferRedeemed event publishing. Sprint 2 adds the claim lifecycle — automatic claim generation 24 hours post-redemption, deduction calculation, and ClaimSubmitted event publishing.

---

## Sprint 2 Goal

By end of sprint — claims are automatically generated 24 hours after redemption, deductions are correctly calculated from funding splits, and ClaimSubmitted events are published to Analytics.

---

## Requirements

### Requirement 1 — Claim Generation

Implement `ClaimGenerator.generate()`:

1. A Claim is automatically generated 24 hours after a Redemption is confirmed
2. Claim amount = `discountApplied` from the Redemption
3. Deduction amount = `discountApplied × (vendorShare / 100)`
   - Get vendorShare by calling Campaign Service `GET /campaigns/:id`
   - If vendorShare is 0 (100% Kroger funded) → deduction is 0.00
4. Claim status starts as PENDING
5. VendorId is taken from the Campaign's funding record

### Requirement 2 — Claim Scheduling

Implement `ClaimGenerator.scheduleT24()`:

1. After confirming a redemption — schedule claim generation for T+24 hours
2. Simple approach for session: store a `scheduledAt = redeemedAt + 24hrs` on the redemption record
3. A background job polls for redemptions where `scheduledAt <= now` and status is CONFIRMED with no claim
4. On trigger — generate the claim and update redemption status to CLAIMED

### Requirement 3 — ClaimSubmitted Event

After claim is generated:

1. Publish `ClaimSubmitted` event to Redis channel `promotionos.redemption.claim.submitted`
2. Event must match exact schema in `contract-redemption-analytics.md`
3. `schemaVersion: 1`
4. Analytics Service consumes this for funding cost tracking

### Requirement 4 — GET /redemptions/:id

Implement the redemption lookup endpoint:

1. Return full redemption record including claim status if it exists
2. Response must include: id, tenantId, customerId, campaignId, discountApplied, cartTotal, status, redeemedAt
3. If claim exists — include: claim.id, claim.status, claim.amount, claim.deduction
4. Tenant isolation — only return redemption if tenantId matches

---

## Domain Rules

From ubiquitous-language.md:
- Claim is generated automatically 24 hours after Redemption confirmation
- Deduction = discountApplied × vendorShare%
- If vendorShare is 0 — deduction is 0.00 (not an error)
- Redemption status: CONFIRMED → CLAIMED (after claim generated)
- Claim status: PENDING → SUBMITTED

---

## Contracts Involved

`contract-redemption-analytics.md` — ClaimSubmitted event
Analytics Service is pre-built. Your ClaimSubmitted event must match the schema exactly.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 28 | redeem-001 confirmed at T+0 | claim generated at T+24hrs |
| 29 | camp-001 vendorShare:60%, discount:6.99 | deduction: 4.19 |
| — | camp-003 vendorShare:0%, discount:10.00 | deduction: 0.00 |
| — | GET /redemptions/redeem-001 | includes claim with PENDING status |
| — | GET /redemptions/redeem-001 (after T+24) | claim status CLAIMED, ClaimSubmitted emitted |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Claim scheduling mechanism — polling vs event-driven scheduler
- How you handle the case where Campaign Service is unavailable when generating a claim
