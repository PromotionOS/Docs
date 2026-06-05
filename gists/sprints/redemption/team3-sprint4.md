# Team 3 — Redemption Service — Sprint 4

> Type: Full Validation + Integration
> Duration: 2 hours
> Skills: sdd-spec-writer, redemption-service-guide, pr-reviewer

---

## Context

Sprints 1-3 built idempotent redemption, claim generation, and history queries. Sprint 4 is integration — validate the full redemption flow with live eligibility checks and confirm all scenarios pass.

---

## Sprint 4 Goal

All redemption scenarios pass end-to-end with live Eligibility Service. Claims are generated and ClaimSubmitted events flow to Analytics. Service is demo-ready.

---

## Requirements

### Requirement 1 — Live Eligibility Integration

Confirm live eligibility check is working:

1. `POST /redeem` calls live Eligibility Service (not a mock)
2. If eligibility check fails — redemption is rejected with the eligibility reason
3. If Eligibility Service is down — redemption is rejected with 503

### Requirement 2 — Full Scenario Pass

Run all redemption scenarios with live data:

| Scenario | Expected |
|----------|----------|
| 7 | Duplicate idempotency key → 409 |
| 17 | Same key different tenant → 200 |
| 28 | Claim at T+24hrs |
| 29 | Claim deduction: 60% of discount |
| — | Ineligible customer → 400 OFFER_INELIGIBLE |
| — | GET /redemptions/stats for camp-001 |

### Requirement 3 — Hardening

1. All endpoints validate tenantId presence — return 400 if missing
2. Redemption amount must be > 0 — return 400 if discountApplied is 0 or negative
3. CampaignId must reference an ACTIVE campaign — call Campaign Service to verify

### Requirement 4 — Demo Redemption

Prepare a demo redemption that works cleanly in Sprint 4 demo:

1. Use `cust-003` + `camp-003` (Spend $50 Get $10 Off)
2. cartTotal: 67.50 (above threshold)
3. idempotencyKey: `demo-sprint4-001`
4. Confirm redemption succeeds, OfferRedeemed published, Analytics updates

---

## Sprint 4 ADR Topics

Record an ADR for:
- Campaign status validation at redemption time — synchronous call vs cached state
