# Team 4 — Catalog & Customer Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, catalog-customer-guide, adr-recorder, pr-reviewer, catalog-eligibility-contract

---

## Context

Sprint 1 delivered UPC lookup, exclusion inheritance, and customer profile. Sprint 2 adds segment recalculation, the POST /catalog/exclusions endpoint, and the SegmentUpdated event that keeps Eligibility Service in sync.

---

## Sprint 2 Goal

By end of sprint — segment membership is recalculated from purchase history, exclusion changes trigger events to Eligibility, and the full catalog mutation API is available.

---

## Requirements

### Requirement 1 — Segment Calculator

Implement `SegmentCalculator.calculate()`:

1. Calculate segment membership from 90-day rolling purchase history
2. Segment rules:
   - `segment-high-value`: annualSpend >= 5000
   - `segment-mid-value`: annualSpend >= 2000 AND < 5000
   - `segment-low-value`: annualSpend < 2000
   - `segment-new-customer`: zero purchases in last 90 days
   - `segment-wine-buyer`: purchased any wine UPC (cat-wine-red, cat-wine-white, cat-wine-rose) at least 2 times in last 90 days
   - `segment-vitamin-buyer`: purchased any vitamin UPC (cat-vitamins) at least 1 time in last 90 days
   - Geographic segments: `segment-midwest`, `segment-southeast`, `segment-west`, `segment-northeast`, `segment-southwest` — based on customer's division
3. A customer can belong to multiple segments simultaneously
4. Recalculation is triggered by `POST /customers/:id/recalculate-segments`

### Requirement 2 — SegmentUpdated Event

When segment membership changes after recalculation:

1. Compare new segments with previous segments
2. If any change — publish `SegmentUpdated` event to Redis
3. Event must match exact schema in `contract-catalog-eligibility.md`
4. Include both `previousSegments` and `currentSegments` in payload
5. If segments unchanged — do NOT publish event (no unnecessary noise)

### Requirement 3 — POST /catalog/exclusions

Implement exclusion management endpoint:

1. `POST /catalog/exclusions` — add or remove a category exclusion
2. Request: `{ categoryId, excluded: boolean, reason: string, tenantId }`
3. On change — resolve all affected UPCs using `HierarchyResolver.resolveInheritance()`
4. Update `excluded` flag on all affected categories and UPCs
5. Publish `CatalogItemExcluded` event with all affected UPCs listed

### Requirement 4 — Purchase History API

Implement `GET /customers/:id/purchases`:

1. Return 90-day rolling purchase history for a customer
2. Used internally by SegmentCalculator
3. Response: list of `{ upc, quantity, spend, date }`

---

## Domain Rules

From ubiquitous-language.md:
- Segment membership is recalculated from 90-day rolling purchase history
- A customer can belong to multiple segments simultaneously
- Geographic segment is determined by customer's division field
- SegmentUpdated is only published when segments actually change

---

## Contracts Involved

`contract-catalog-eligibility.md` — CatalogItemExcluded + SegmentUpdated
Both events are consumed by Eligibility Service.
Schema versions must remain at 1 unless breaking change is required.

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| — | POST /customers/cust-001/recalculate-segments | segments include segment-wine-buyer (4 wine purchases in 90 days) |
| — | POST /customers/cust-004/recalculate-segments | segments: [segment-new-customer, segment-low-value, segment-southeast] |
| — | POST /catalog/exclusions {categoryId:cat-dairy, excluded:true} | CatalogItemExcluded published with upc-milk-gallon, upc-cheese-cheddar |
| — | SegmentUpdated for cust-001 | previousSegments vs currentSegments shows actual diff |
| — | Recalculate with no change | SegmentUpdated NOT published |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Segment recalculation trigger — on-demand vs scheduled vs event-driven
- How you determine wine-buyer segment (minimum purchase count threshold)
