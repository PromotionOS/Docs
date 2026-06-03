# Contract — Catalog & Customer → Eligibility

> Producer: Catalog & Customer Service (Team 4)
> Consumer: Eligibility Service (Team 2)
> Type: Domain Events (Redis Pub/Sub) + REST API
> Status: LOCKED — changes require schemaVersion bump + contract skill update

---

## Events Published by Catalog & Customer Service

### CatalogItemExcluded

Emitted when a Category's exclusion status changes — either directly excluded or inherited from a parent.
Eligibility Service must update its exclusion list for all affected UPCs upon receiving this event.

```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "categoryId": "string",
    "categoryName": "string",
    "excluded": "boolean",
    "excludeReason": "REGULATORY | REGULATORY_INHERITED | MANUAL",
    "inheritedFrom": "string | null",
    "affectedUPCs": ["string"],
    "affectedChildCategories": ["string"]
  }
}
```

**ACL Translation (Eligibility Service):**
- `payload.affectedUPCs` → add/remove from `Exclusion` list in all active `EligibilityRule` records that have `cat-alcohol` or `cat-tobacco` in their exclusions
- `payload.excluded: true` → add exclusions
- `payload.excluded: false` → remove exclusions
- `payload.inheritedFrom` → log for audit, no functional impact on Eligibility

---

### SegmentUpdated

Emitted when a Customer's segment membership changes (nightly recalculation or triggered update).
Eligibility Service uses this to refresh its segment cache for real-time eligibility checks.

```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "customerId": "uuid",
    "previousSegments": ["string"],
    "currentSegments": ["string"],
    "loyaltyTier": "BASIC | SILVER | GOLD | PLATINUM",
    "annualSpend": "number",
    "changedAt": "ISO-8601 timestamp"
  }
}
```

**ACL Translation (Eligibility Service):**
- `payload.customerId` → invalidate segment cache entry for this customer
- `payload.currentSegments` → update cached segments
- `payload.loyaltyTier` → update cached loyalty tier
- Eligibility Service fetches fresh profile from Catalog & Customer API on next eligibility check

---

## REST API Consumed by Eligibility Service

### GET /customers/:id

Called by Eligibility Service during eligibility check to get current customer profile.

**Request:**
```
GET /customers/{customerId}?tenantId={tenantId}
```

**Response:**
```json
{
  "id": "uuid",
  "tenantId": "string",
  "loyaltyTier": "BASIC | SILVER | GOLD | PLATINUM",
  "segments": ["string"],
  "division": "string",
  "annualSpend": "number"
}
```

**Error responses:**
```json
{ "error": "CUSTOMER_NOT_FOUND", "customerId": "uuid" }      // 404
{ "error": "TENANT_MISMATCH" }                                // 403
```

### GET /catalog/upc/:code

Called by Eligibility Service to check if a UPC is excluded.

**Request:**
```
GET /catalog/upc/{upcCode}?tenantId={tenantId}
```

**Response:**
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

**Error responses:**
```json
{ "error": "UPC_NOT_FOUND", "code": "string" }   // 404
```

---

## Redis Channel Names

| Event | Channel |
|-------|---------|
| CatalogItemExcluded | `promotionos.catalog.item.excluded` |
| SegmentUpdated | `promotionos.customer.segment.updated` |

---

## Breaking Change Protocol

If Team 4 needs to change any field in these schemas:

1. Bump `schemaVersion` in the event
2. Update `skill-contract-catalog-eligibility.md` with what changed and blast radius
3. Team 2 uses the updated skill to refactor their ACL
4. Both services must support old + new schema version during transition

**Safe to add:** New optional fields with null defaults
**Breaking:** Rename, type change, removal, enum changes

---

## Validation

| Scenario | Input | Expected |
|----------|-------|----------|
| 23 | GET /catalog/upc/upc-wine-chard | excluded:true, excludedFrom:cat-alcohol |
| 23 | GET /catalog/upc/upc-cola-12pk | excluded:false |
| — | GET /customers/cust-001 | loyaltyTier:PLATINUM, division:division-midwest |
| — | GET /customers/cust-006 | loyaltyTier:GOLD, annualSpend:4999.00 |
| — | GET /customers/cust-007 | loyaltyTier:PLATINUM, annualSpend:5000.00 |
| 21 | tier boundary at exactly $5000 | PLATINUM |
| 22 | tier boundary at $4999 | GOLD |
