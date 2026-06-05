---
name: contract-catalog-eligibility
description: Contract skill for Eligibility Service consuming from Catalog & Customer Service — ACL translation, field mapping, blast radius guide
metadata:
  type: reference
---

# Contract: Catalog & Customer → Eligibility

## What This Contract Covers

Catalog & Customer Service exposes two REST APIs and publishes two Redis events that Eligibility Service consumes. The REST APIs are called synchronously during eligibility evaluation to fetch live customer profile and UPC exclusion status. The Redis events allow Eligibility Service to invalidate its caches without polling.

## Events / APIs (Summary)

| Surface | Type | Purpose |
|---------|------|---------|
| `CatalogItemExcluded` | Redis — `promotionos.catalog.item.excluded` | Update UPC exclusion lists in active rules |
| `SegmentUpdated` | Redis — `promotionos.customer.segment.updated` | Invalidate segment cache for a customer |
| `GET /customers/:id` | REST | Fetch current customer profile during eligibility check |
| `GET /catalog/upc/:code` | REST | Check if a specific UPC is excluded |

## ACL Translation Guide

### CatalogItemExcluded

**What arrives:**
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

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_event_translator.go

func (t *CatalogEventTranslator) HandleCatalogItemExcluded(event CatalogItemExcludedEvent) error {
    payload := event.Payload

    // Find all active EligibilityRules that reference this categoryId in their exclusions list.
    rules, err := t.ruleRepo.FindRulesWithExclusion(event.TenantID, payload.CategoryID)
    if err != nil {
        return fmt.Errorf("finding rules for exclusion update: %w", err)
    }

    for _, rule := range rules {
        if payload.Excluded {
            // Add each affected UPC to the rule's exclusion set.
            rule.AddExcludedUPCs(payload.AffectedUPCs)
        } else {
            // Remove UPCs — exclusion lifted, e.g. a MANUAL exclusion was reversed.
            rule.RemoveExcludedUPCs(payload.AffectedUPCs)
        }
        if err := t.ruleRepo.Save(rule); err != nil {
            return fmt.Errorf("saving rule %s: %w", rule.CampaignID, err)
        }
    }

    // Log inheritedFrom for audit — no functional impact on Eligibility.
    if payload.InheritedFrom != nil {
        t.log.Info("exclusion inherited", "categoryId", payload.CategoryID,
            "inheritedFrom", *payload.InheritedFrom)
    }

    return nil
}
```

**Fields to watch:**
- `payload.excluded` → boolean gate; `true` means add UPCs to exclusion lists, `false` means remove them — both directions must be handled
- `payload.affectedUPCs` → the already-resolved list of UPC codes; Eligibility does not need to walk the category tree itself
- `payload.inheritedFrom` → nullable; log it but do not use it for routing logic
- `payload.affectedChildCategories` → informational; Eligibility works at UPC level, not category level

---

### SegmentUpdated

**What arrives:**
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

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_event_translator.go

func (t *CatalogEventTranslator) HandleSegmentUpdated(event SegmentUpdatedEvent) error {
    payload := event.Payload

    // Invalidate the segment cache entry for this customer.
    // Do NOT update from the event payload directly — always re-fetch on next check.
    t.segmentCache.Invalidate(event.TenantID, payload.CustomerID)

    t.log.Info("segment cache invalidated",
        "customerId", payload.CustomerID,
        "tenantId", event.TenantID,
        "loyaltyTier", payload.LoyaltyTier,
    )

    return nil
}
```

**Fields to watch:**
- `payload.customerId` → cache invalidation key; do not store `currentSegments` or `loyaltyTier` from the event — the event is a cache-bust signal only, not the source of truth
- `payload.loyaltyTier` → do not cache this from the event; always re-fetch from `GET /customers/:id` on the next eligibility check because the ubiquitous language rule states tier must be recalculated on every fetch

---

### GET /customers/:id (REST)

**What arrives:**
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

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_rest_client.go

func (c *CatalogRestClient) GetCustomerProfile(tenantID, customerID string) (*CustomerProfile, error) {
    resp, err := c.httpClient.Get(
        fmt.Sprintf("/customers/%s?tenantId=%s", customerID, tenantID),
    )
    if err != nil {
        return nil, fmt.Errorf("catalog GET /customers/%s: %w", customerID, err)
    }

    switch resp.StatusCode {
    case 200:
        var body catalogCustomerResponse
        if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
            return nil, err
        }
        // Translate into Eligibility's domain language.
        return &CustomerProfile{
            CustomerID:   body.ID,
            TenantID:     body.TenantID,
            LoyaltyTier:  domain.LoyaltyTier(body.LoyaltyTier),
            Segments:     body.Segments,
            Division:     body.Division,
        }, nil
    case 404:
        return nil, ErrCustomerNotFound
    case 403:
        return nil, ErrTenantMismatch
    default:
        return nil, fmt.Errorf("unexpected status %d from catalog", resp.StatusCode)
    }
}
```

**Fields to watch:**
- `loyaltyTier` → maps to `domain.LoyaltyTier` enum; values are `BASIC | SILVER | GOLD | PLATINUM`
- `division` → maps to `CustomerProfile.Division`; compared against `EligibilityRule.geoScope` during geo check
- `annualSpend` → do not cache or use for tier decisions; tier is the canonical tier field

---

### GET /catalog/upc/:code (REST)

**What arrives:**
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

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_rest_client.go

func (c *CatalogRestClient) IsUPCExcluded(tenantID, upcCode string) (bool, error) {
    resp, err := c.httpClient.Get(
        fmt.Sprintf("/catalog/upc/%s?tenantId=%s", upcCode, tenantID),
    )
    if err != nil {
        return false, fmt.Errorf("catalog GET /catalog/upc/%s: %w", upcCode, err)
    }

    switch resp.StatusCode {
    case 200:
        var body catalogUPCResponse
        if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
            return false, err
        }
        // Eligibility only needs the boolean — price, categoryId, reason are for Catalog's context.
        return body.Excluded, nil
    case 404:
        return false, ErrUPCNotFound
    default:
        return false, fmt.Errorf("unexpected status %d from catalog", resp.StatusCode)
    }
}
```

**Fields to watch:**
- `excluded` → the only field Eligibility acts on; `true` means any rule that references this UPC in its scope should deny eligibility
- `excludedFrom` → nullable parent category — informational, do not use for routing
- `price` → belongs to Catalog's domain; Eligibility does not use it

## What Changes When The Contract Changes

### If producer adds a new field to a REST response
No action required if you use a lenient JSON decoder (unknown fields ignored). If the field is one Eligibility should act on, add it to the ACL client struct and map it into the domain object. No version bump needed.

### If producer renames a field
REST: Catalog Service bumps the path version to `/v2/customers/:id`. Update `CatalogRestClient` to call the new path. Keep the v1 client until Catalog Service retires it.
Event: `schemaVersion` bumps. Update both `catalogCustomerResponse` struct and the translator to handle `schemaVersion: 1` and `schemaVersion: 2` during the transition window.

### If producer removes a field
Breaking change on both REST and event surfaces. Do not deploy until Catalog Service has confirmed the field is absent from all responses and you have removed all references from your ACL, domain model, and tests.

## Blast Radius Checklist

When you receive a `skill-contract-catalog-eligibility` update — check these files in your repo:
- [ ] `infrastructure/acl/catalog_event_translator.go` — event field mappings
- [ ] `infrastructure/acl/catalog_rest_client.go` — REST response struct and mapping logic
- [ ] `domain/model/customer_profile.go` — domain struct that receives translated customer data
- [ ] `domain/model/eligibility_rule.go` — `ExcludedUPCs` field if exclusion shape changes
- [ ] `domain/engine/rule_engine.go` — geo check, tier check, UPC exclusion check logic
- [ ] `infrastructure/cache/segment_cache.go` — cache key format if `customerId` field name changes
- [ ] `test/acl/catalog_event_translator_test.go` — fixture JSON for both events
- [ ] `test/acl/catalog_rest_client_test.go` — stub response JSON for both REST endpoints

## Common Mistakes

**Mistake 1 — Populating `CustomerProfile` from `SegmentUpdated` instead of from the REST API.**
The `SegmentUpdated` event is a cache-bust signal. It tells you the cache is stale, not what the new values are. Always call `GET /customers/:id` on the next eligibility check — never write segment or tier values from the event into the domain model.

**Mistake 2 — Using `annualSpend` to recalculate loyalty tier locally.**
The ubiquitous language document states: "Tier is recalculated on every customer profile fetch — never trust stored tier." Do not implement tier derivation from `annualSpend` in Eligibility Service. Use the `loyaltyTier` field returned by `GET /customers/:id`.

**Mistake 3 — Failing the entire eligibility check when a single UPC lookup returns 404.**
`GET /catalog/upc/:code` may return 404 for an unknown UPC. Treat this as non-excluded (UPC not in Catalog = not excluded) and continue evaluation. Log the 404 for observability but do not surface it as an eligibility failure.

## Validation

| Scenario | Input | Expected |
|----------|-------|----------|
| Scenario 23 | `GET /catalog/upc/upc-wine-chard` | `excluded: true`, `excludedFrom: cat-alcohol` |
| Scenario 23 | `GET /catalog/upc/upc-cola-12pk` | `excluded: false` |
| — | `GET /customers/cust-001` | `loyaltyTier: PLATINUM`, `division: division-midwest` |
| — | `GET /customers/cust-006` | `loyaltyTier: GOLD`, `annualSpend: 4999.00` |
| — | `GET /customers/cust-007` | `loyaltyTier: PLATINUM`, `annualSpend: 5000.00` |
| Scenario 21 | `annualSpend = 5000.00` | `loyaltyTier: PLATINUM` (boundary — inclusive) |
| Scenario 22 | `annualSpend = 4999.00` | `loyaltyTier: GOLD` (boundary — exclusive) |
| `CatalogItemExcluded` | `excluded: true`, `affectedUPCs: [upc-wine-cab, upc-wine-chard]` | Both UPCs added to exclusion sets of all active rules referencing `cat-alcohol` |
| `CatalogItemExcluded` | `excluded: false`, `affectedUPCs: [upc-wine-cab]` | UPC removed from exclusion sets |
| `SegmentUpdated` | `customerId: cust-001` | Segment cache entry invalidated for cust-001; no domain model written |
