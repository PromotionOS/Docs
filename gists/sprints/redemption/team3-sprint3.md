# Team 3 — Redemption Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, redemption-service-guide, adr-recorder, pr-reviewer, redemption-analytics-contract

---

## Context

Sprint 1 delivered idempotent redemption confirmation. Sprint 2 added claim generation and ClaimSubmitted events. Sprint 3 adds redemption history queries and the full tenant isolation validation pass.

---

## Sprint 3 Goal

By end of sprint — redemption history is queryable per customer and per campaign, tenant isolation is verified across all endpoints, and all redemption scenarios pass.

---

## Requirements

### Requirement 1 — Redemption History by Customer

Implement `GET /redemptions?customerId=X&tenantId=Y`:

1. Return all redemptions for a customer within the tenant
2. Sorted by redeemedAt descending (most recent first)
3. Include claim status for each redemption
4. Pagination: default 20 per page, support `?page=N`
5. Tenant isolation — NEVER return redemptions from another tenant

### Requirement 2 — Redemption History by Campaign

Implement `GET /redemptions?campaignId=X&tenantId=Y`:

1. Return all redemptions for a campaign within the tenant
2. Include: customerId, discountApplied, cartTotal, redeemedAt, storeId, division
3. Used by MX team to see who redeemed their campaign
4. Pagination: default 50 per page

### Requirement 3 — Redemption Statistics

Implement `GET /redemptions/stats?campaignId=X&tenantId=Y`:

1. Return aggregated stats for a campaign:
   - totalRedemptions: count
   - totalDiscountApplied: sum of all discountApplied
   - totalClaimsGenerated: count of claims with status SUBMITTED
   - averageCartTotal: average of cartTotal across all redemptions
2. Used by Analytics Service for lift calculation inputs

### Requirement 4 — Full Scenario Validation

Run and confirm ALL scenarios:
- Scenario 7: duplicate redemption blocked
- Scenario 17: cross-tenant idempotency
- Scenario 28: claim at T+24hrs
- Scenario 29: claim deduction calculation

---

## Domain Rules

From ubiquitous-language.md:
- Redemptions are immutable — history queries are read-only
- Tenant isolation is absolute — no cross-tenant data leakage
- Idempotency is per tenant

---

## Contracts Involved

No new contract changes this sprint.
Validate existing `contract-redemption-analytics.md` compliance for all emitted events.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 7 | Duplicate idempotency key | 409 DUPLICATE_REDEMPTION |
| 17 | Same key different tenant | 200 success |
| 28 | Claim at T+24 | ClaimSubmitted event emitted |
| 29 | camp-001 discount:6.99, vendorShare:60% | deduction:4.19 |
| — | GET /redemptions?customerId=cust-001 | returns redeem-001 and redeem-004 |
| — | GET /redemptions/stats?campaignId=camp-001 | totalRedemptions:1, totalDiscountApplied:6.99 |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Pagination approach — offset vs cursor based
- How stats are calculated — on-demand aggregation vs pre-computed
