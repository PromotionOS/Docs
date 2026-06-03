# Team 4 — Catalog & Customer Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, catalog-customer-guide, adr-recorder, pr-reviewer, catalog-eligibility-contract

---

## Context

Sprint 1 built catalog hierarchy and customer profiles. Sprint 2 added segment recalculation and exclusion management. Sprint 3 adds purchase history recording and price change tracking.

---

## Sprint 3 Goal

By end of sprint — purchase history can be recorded via API (simulating POS feed), price changes are tracked point-in-time, and all catalog/customer scenarios pass.

---

## Requirements

### Requirement 1 — Purchase History Recording

Implement `POST /customers/:id/purchases`:

1. Record a purchase event for a customer:
   - upc, quantity, spend, purchasedAt (server timestamp), storeId, tenantId
2. Append-only — never update or delete purchase records
3. After recording — trigger segment recalculation
4. If segment membership changes — publish SegmentUpdated event

### Requirement 2 — Price History

Implement price change tracking:

1. `PUT /catalog/upc/:code/price` — update a UPC's price
2. Record price history: `{ upc, previousPrice, newPrice, effectiveFrom: now }`
3. `GET /catalog/upc/:code/price?at=timestamp` — return the price at a specific point in time
4. Used by Redemption Service: price at redemption time must be the price when redemption occurred

### Requirement 3 — Bulk Catalog Lookup

Implement `POST /catalog/upcs/bulk`:

1. Accept list of UPC codes
2. Return full details for each (excluded status, price, category)
3. Used by Eligibility Service to check multiple UPCs in one call instead of N individual calls
4. Missing UPCs are returned with `found: false` — no error for individual missing UPCs

### Requirement 4 — Full Scenario Validation

Confirm all catalog and customer scenarios:
- Scenario 21: tier boundary at exactly $5000 → PLATINUM
- Scenario 22: tier boundary at $4999 → GOLD
- Scenario 23: exclusion inheritance for wine UPCs
- Scenario 24: tobacco buyer eligibility
- Scenario 25: geo miss for tobacco campaign

---

## Domain Rules

From ubiquitous-language.md:
- Price is point-in-time — always the price at time of redemption
- Purchase history is append-only
- Segment recalculation is triggered after every new purchase record

---

## Contracts Involved

`contract-catalog-eligibility.md` — SegmentUpdated (triggered by new purchases)
No schema changes needed.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 21 | GET /customers/cust-007 | loyaltyTier:PLATINUM (annualSpend:5000) |
| 22 | GET /customers/cust-006 | loyaltyTier:GOLD (annualSpend:4999) |
| 23 | POST /catalog/upcs/bulk [upc-wine-cab, upc-cola-12pk] | wine-cab excluded:true, cola excluded:false |
| — | POST /customers/cust-004/purchases {upc:upc-wine-cab, qty:2} | segment-wine-buyer added, SegmentUpdated published |
| — | PUT /catalog/upc/upc-cola-12pk/price {price:7.99} | price updated, history recorded |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Point-in-time price lookup — full history table vs effective-date ranges
- Bulk lookup performance — single query vs parallel individual queries
