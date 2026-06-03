# Team 4 — Catalog & Customer Service
## Type: Build via SDD

---

## Context

The Catalog & Customer Service owns two domains that the Eligibility Service depends on heavily. The Catalog context manages the UPC hierarchy and exclusion inheritance. The Customer context manages customer profiles, loyalty tiers, and segment membership.

The domain model and interfaces are defined. Your job is to implement the business logic.

**Your job:**
1. Read the domain model using the catalog-customer-guide skill
2. Read the catalog-eligibility-contract skill to understand what Eligibility expects
3. Write a spec using the sdd-spec-writer skill
4. Implement HierarchyResolver and SegmentCalculator
5. Record an ADR using the adr-recorder skill
6. Validate against catalog and customer scenarios
7. Open a PR

---

## Business Rules You Must Implement

### Exclusion Inheritance (Hierarchy Resolver)
1. A category can be explicitly excluded (e.g. `cat-alcohol` excluded for regulatory reasons)
2. Exclusion is inherited downward — if `cat-alcohol` is excluded, all child categories (e.g. `cat-wine`) are also excluded
3. All UPCs in an excluded category are excluded regardless of their own flags
4. When a category's exclusion status changes — publish `CatalogItemExcluded` event for all affected UPCs
5. `IsExcluded(upc)` must traverse the full category hierarchy upward to check for inherited exclusions

### UPC Lookup
6. Given a UPC code return its full details including category and whether it is excluded
7. Price is point-in-time — always return the current price, never cache

### Customer Profile
8. A customer's `loyaltyTier` is determined by `annualSpend`:
   - BASIC: < $500
   - SILVER: $500 - $1,999
   - GOLD: $2,000 - $4,999
   - PLATINUM: $5,000+
9. Tier must be recalculated on every profile fetch — never trust stored tier
10. A customer can belong to multiple segments simultaneously

### Segment Membership
11. Segment membership is based on 90-day rolling purchase history
12. `segment-wine-buyer`: purchased wine UPCs at least 2 times in 90 days
13. `segment-high-value`: annualSpend > $5,000
14. `segment-mid-value`: annualSpend $2,000 - $4,999
15. `segment-low-value`: annualSpend < $2,000
16. `segment-new-customer`: no purchases in last 90 days

---

## Validation Scenarios to Pass

| Scenario | Input | Expected |
|----------|-------|----------|
| — | GET /catalog/upc/upc-wine-cab | excluded: true, excludedFrom: cat-alcohol |
| — | GET /catalog/upc/upc-cola-12pk | excluded: false |
| — | GET /catalog/category/cat-wine | excluded: true, inheritedFrom: cat-alcohol |
| — | GET /customers/cust-001 | loyaltyTier: PLATINUM, annualSpend: 8500 |
| — | GET /customers/cust-004 | loyaltyTier: BASIC, segments: [segment-new-customer] |
| — | GET /customers/cust-001/segments | [segment-high-value, segment-wine-buyer, segment-midwest] |

---

## API Contracts You Must Honour

**GET /catalog/upc/:code**
```json
{
  "code": "string",
  "name": "string",
  "price": "number",
  "categoryId": "string",
  "excluded": "boolean",
  "excludedFrom": "string | null",
  "excludeReason": "string | null"
}
```

**GET /catalog/category/:id**
```json
{
  "id": "string",
  "name": "string",
  "parentId": "string | null",
  "excluded": "boolean",
  "inheritedFrom": "string | null",
  "childCategories": ["string"],
  "upcs": ["string"]
}
```

**GET /customers/:id**
```json
{
  "id": "string",
  "tenantId": "string",
  "loyaltyTier": "BASIC | SILVER | GOLD | PLATINUM",
  "segments": ["string"],
  "division": "string",
  "annualSpend": "number"
}
```

---

## Events You Must Publish

**CatalogItemExcluded**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "categoryId": "string",
  "affectedUPCs": ["string"],
  "reason": "string",
  "occurredAt": "timestamp",
  "schemaVersion": 1
}
```

**SegmentUpdated**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "customerId": "string",
  "previousSegments": ["string"],
  "currentSegments": ["string"],
  "occurredAt": "timestamp",
  "schemaVersion": 1
}
```

---

## Skills to Use

- `sdd-spec-writer` — write your spec before touching code
- `ddd-guide` — understand value objects (UPC, LoyaltyTier) and domain services
- `catalog-customer-guide` — understand the skeleton
- `adr-recorder` — record your exclusion inheritance algorithm decision
- `catalog-eligibility-contract` — understand what Eligibility Service expects from you
