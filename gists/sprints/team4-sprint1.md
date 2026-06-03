# Team 4 — Catalog & Customer Service — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, catalog-customer-guide, adr-recorder, pr-reviewer, catalog-eligibility-contract

---

## Context

The Catalog & Customer Service provides two things every other service depends on — product exclusion status and customer loyalty tier. Sprint 1 focuses on these two foundations. Get them right and every downstream service works correctly.

---

## Sprint 1 Focus

**UPC exclusion lookup + customer loyalty tier. Two endpoints. Full SDD process.**

---

## Requirements

### Requirement 1 — UPC Exclusion Lookup

Implement `GET /catalog/upc/:code`:

1. Return UPC details: code, name, price, categoryId
2. Resolve exclusion by traversing category hierarchy upward:
   - If UPC's own category is excluded → excluded: true
   - If any parent category is excluded → excluded: true, excludedFrom: that category's id
   - If no excluded ancestor → excluded: false
3. Traversal: UPC → category → parent → grandparent until root
4. Return first excluded ancestor found

Key test cases:
- `upc-wine-chard` → cat-wine-white → cat-wine → **cat-alcohol (excluded)** → `excluded: true, excludedFrom: cat-alcohol`
- `upc-cola-12pk` → cat-soft-drinks → cat-beverages (not excluded) → `excluded: false`
- `upc-cigarettes-marlboro` → cat-cigarettes → **cat-tobacco (excluded)** → `excluded: true, excludedFrom: cat-tobacco`

### Requirement 2 — Customer Loyalty Tier

Implement `GET /customers/:id`:

1. Return: id, tenantId, loyaltyTier, segments, division, annualSpend
2. Loyalty tier MUST be calculated from annualSpend at fetch time:
   - BASIC: annualSpend < 500
   - SILVER: 500 <= annualSpend < 2000
   - GOLD: 2000 <= annualSpend < 5000
   - PLATINUM: annualSpend >= 5000
3. Never trust the stored tier — always recalculate

---

## Domain Rules

From ubiquitous-language.md:
- Exclusion is inherited downward through category hierarchy
- LoyaltyTier is ALWAYS calculated from annualSpend at fetch time
- PLATINUM threshold is >= 5000 (not > 4999)

---

## Critical Boundary Cases

| Customer | annualSpend | Must return |
|----------|------------|-------------|
| cust-006 | 4999.00 | GOLD |
| cust-007 | 5000.00 | PLATINUM |
| cust-008 | 499.00 | BASIC |

---

## Contracts Involved

`contract-catalog-eligibility.md` — Eligibility Service calls your APIs.
Read the exact response shapes before writing your spec.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 21 | GET /customers/cust-007 | loyaltyTier: PLATINUM |
| 22 | GET /customers/cust-006 | loyaltyTier: GOLD |
| 23 | GET /catalog/upc/upc-wine-chard | excluded:true, excludedFrom:cat-alcohol |
| 23 | GET /catalog/upc/upc-cola-12pk | excluded:false |

---

## Sprint 1 ADR Topics

Record an ADR for:
- Why loyalty tier is recalculated at fetch time vs stored — correctness over performance
