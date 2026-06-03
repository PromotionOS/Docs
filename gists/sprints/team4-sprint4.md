# Team 4 — Catalog & Customer Service — Sprint 4

> Type: Full Validation + Integration
> Duration: 2 hours
> Skills: sdd-spec-writer, catalog-customer-guide, pr-reviewer

---

## Context

Sprints 1-3 built catalog hierarchy, customer profiles, segment recalculation, and purchase history. Sprint 4 is integration and validation — confirm all catalog and customer scenarios pass with live data flowing to Eligibility.

---

## Sprint 4 Goal

All catalog and customer scenarios pass. Eligibility Service is successfully consuming events. Service is demo-ready.

---

## Requirements

### Requirement 1 — Live Event Integration

Confirm events are flowing to Eligibility Service:

1. Trigger a CatalogItemExcluded event for a new category
2. Confirm Eligibility Service ACL receives and processes it
3. Trigger a SegmentUpdated event by recording a purchase
4. Confirm Eligibility Service refreshes customer profile cache

### Requirement 2 — Full Scenario Pass

| Scenario | Expected |
|----------|----------|
| 21 | cust-007 annualSpend:5000 → PLATINUM |
| 22 | cust-006 annualSpend:4999 → GOLD |
| 23 | upc-wine-chard excluded:true, inheritedFrom:cat-alcohol |
| 24 | cust-012 tobacco buyer midwest → eligible for camp-008 |
| 25 | cust-002 southeast → GEO_MISMATCH for camp-008 |
| — | Bulk UPC lookup returns correct exclusion status |

### Requirement 3 — Hardening

1. All endpoints validate tenantId
2. UPC lookup returns 404 for unknown UPCs — never 500
3. Customer lookup returns 404 for unknown customers — never 500
4. Category hierarchy traversal must not infinite-loop on circular references (add cycle detection)

### Requirement 4 — Demo Data Confirmation

Verify all test data is correctly seeded and queryable:

1. All 48 UPCs return correct exclusion status
2. All 12 customers return correct loyalty tier from annualSpend
3. All 6 tier boundary customers (cust-006, cust-007, cust-008) are correct

---

## Sprint 4 ADR Topics

Record an ADR for:
- Cycle detection in category hierarchy — why it matters and how implemented
