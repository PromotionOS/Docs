---
name: contract-notification-catalog
description: Contract skill for Notification Service consuming from Catalog & Customer Service — ACL translation, field mapping, blast radius guide for store manager resolution and segment cache invalidation
metadata:
  type: reference
---

# Contract: Catalog & Customer → Notification

## What This Contract Covers

Notification Service cannot route geo-scoped notifications to individual store managers from campaign event data alone — campaign events carry division identifiers (`geoScope`), not store manager contacts. This contract covers the two REST APIs Notification Service calls to resolve a division into store manager email addresses, and the one Redis event it consumes to invalidate recipient caches when customer segment membership changes.

## Events / APIs (Summary)

| Surface | Type | Direction | Purpose |
|---------|------|-----------|---------|
| `GET /stores?divisionId=X&tenantId=Y` | REST | Notification → Catalog | Expand a division into its constituent stores |
| `GET /stores/:id/manager` | REST | Notification → Catalog | Get store manager contact details for a specific store |
| `SegmentUpdated` | Redis — `promotionos.customer.segment.updated` | Customer → Notification | Invalidate cached recipient lists when segment membership changes |

## ACL Translation Guide

### GET /stores (REST)

**What arrives:**
```json
{
  "divisionId": "string",
  "tenantId": "string",
  "stores": [
    {
      "storeId": "string",
      "name": "string",
      "divisionId": "string",
      "city": "string",
      "state": "string"
    }
  ]
}
```

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_client.go

func (c *CatalogClient) GetStoresByDivision(tenantID, divisionID string) ([]Store, error) {
    resp, err := c.httpClient.Get(
        fmt.Sprintf("/stores?divisionId=%s&tenantId=%s", divisionID, tenantID),
    )
    if err != nil {
        return nil, fmt.Errorf("GET /stores divisionId=%s: %w", divisionID, err)
    }

    switch resp.StatusCode {
    case 200:
        var body catalogStoresResponse
        if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
            return nil, err
        }
        // Translate into Notification's domain language.
        stores := make([]Store, len(body.Stores))
        for i, s := range body.Stores {
            stores[i] = Store{
                StoreID:    s.StoreID,
                Name:       s.Name,
                DivisionID: s.DivisionID,
            }
            // city and state are Catalog domain fields — not needed by Notification Service.
        }
        return stores, nil
    case 400:
        return nil, ErrMissingParam
    case 404:
        return nil, ErrDivisionNotFound
    default:
        return nil, fmt.Errorf("unexpected status %d from catalog GET /stores", resp.StatusCode)
    }
}
```

**Fields to watch:**
- `stores[].storeId` → the key used to call `GET /stores/:id/manager` for each store; do not assume storeId format
- `divisionId` in the response → should match what was sent in the query; if it does not, log a warning
- `city` / `state` → Catalog domain fields; Notification Service should not store these — they are not part of the `Store` entity in Notification's domain

---

### GET /stores/:id/manager (REST)

**What arrives:**
```json
{
  "storeId": "string",
  "managerId": "string",
  "name": "string",
  "email": "string",
  "phone": "string | null"
}
```

**How to translate in your ACL:**
```go
// infrastructure/acl/catalog_client.go

func (c *CatalogClient) GetStoreManager(storeID string) (*StoreManagerContact, error) {
    resp, err := c.httpClient.Get(
        fmt.Sprintf("/stores/%s/manager", storeID),
    )
    if err != nil {
        return nil, fmt.Errorf("GET /stores/%s/manager: %w", storeID, err)
    }

    switch resp.StatusCode {
    case 200:
        var body catalogManagerResponse
        if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
            return nil, err
        }
        return &StoreManagerContact{
            StoreID:   body.StoreID,
            ManagerID: body.ManagerID,
            Email:     body.Email,     // → Notification.recipientId
            // phone is optional — Notification Service uses email only for now
        }, nil
    case 404:
        // Check which 404 variant it is.
        var errBody catalogErrorResponse
        json.NewDecoder(resp.Body).Decode(&errBody)
        if errBody.Error == "MANAGER_NOT_ASSIGNED" {
            return nil, ErrManagerNotAssigned  // log warning, skip this store
        }
        return nil, ErrStoreNotFound
    default:
        return nil, fmt.Errorf("unexpected status %d from catalog GET /stores/%s/manager",
            resp.StatusCode, storeID)
    }
}
```

**How Notification Service uses the contact to build a Notification:**
```go
// domain/service/notification_router.go

func (r *NotificationRouter) RouteToStoreManagers(
    tenantID string,
    divisionIDs []string,
    eventType string,
) ([]*Notification, error) {
    var notifications []*Notification

    for _, divisionID := range divisionIDs {
        stores, err := r.catalogClient.GetStoresByDivision(tenantID, divisionID)
        if err != nil {
            return nil, fmt.Errorf("resolving division %s: %w", divisionID, err)
        }

        // Manager calls can be parallelised — one goroutine per store.
        for _, store := range stores {
            contact, err := r.catalogClient.GetStoreManager(store.StoreID)
            if errors.Is(err, ErrManagerNotAssigned) {
                // Graceful skip — log and continue, do not fail the batch.
                r.log.Warn("no manager assigned", "storeId", store.StoreID)
                continue
            }
            if err != nil {
                return nil, fmt.Errorf("resolving manager for store %s: %w", store.StoreID, err)
            }

            notifications = append(notifications, &Notification{
                RecipientID:   contact.Email,                    // from manager response
                RecipientRole: RecipientRoleStoreManager,
                Channel:       DeliveryChannelEmail,
                EventType:     eventType,
                Status:        NotificationStatusPending,
            })
        }
    }

    return notifications, nil
}
```

**Fields to watch:**
- `email` → maps to `Notification.recipientId`; this is the critical field — if it changes name in the response, the entire notification dispatch breaks
- `managerId` → store for audit; not used for dispatch
- `phone` → nullable; Notification Service ignores it currently — do not crash if absent
- `404 MANAGER_NOT_ASSIGNED` vs `404 STORE_NOT_FOUND` → these are distinct errors in the response body; Notification Service must handle `MANAGER_NOT_ASSIGNED` gracefully (skip and continue) but should surface `STORE_NOT_FOUND` as an error since it indicates a data integrity problem

---

### SegmentUpdated (Redis Event)

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "segmentId": "string",
    "segmentName": "string",
    "changeType": "MEMBER_ADDED | MEMBER_REMOVED | SEGMENT_DELETED",
    "affectedCount": "integer"
  }
}
```

**How to translate in your ACL:**
```go
// infrastructure/acl/customer_event_translator.go

func (t *CustomerEventTranslator) HandleSegmentUpdated(event SegmentUpdatedEvent) error {
    payload := event.Payload

    switch payload.ChangeType {
    case "SEGMENT_DELETED":
        // Remove all routing rules that reference this segment.
        t.recipientCache.RemoveRoutingRulesForSegment(event.TenantID, payload.SegmentID)
        t.log.Info("routing rules removed for deleted segment",
            "segmentId", payload.SegmentID,
            "tenantId", event.TenantID,
        )

    case "MEMBER_ADDED", "MEMBER_REMOVED":
        // Invalidate cached recipient lists tagged with this segment.
        // Do NOT query Customer Service — just invalidate.
        t.recipientCache.InvalidateBySegment(event.TenantID, payload.SegmentID)
        t.log.Info("recipient cache invalidated",
            "segmentId", payload.SegmentID,
            "changeType", payload.ChangeType,
            "affectedCount", payload.AffectedCount,
        )

    default:
        t.log.Warn("unknown changeType in SegmentUpdated", "changeType", payload.ChangeType)
    }

    return nil
}
```

**Fields to watch:**
- `payload.segmentId` → cache invalidation key; tag cached recipient lists with this segment ID so they can be invalidated efficiently
- `payload.changeType` → three variants; `SEGMENT_DELETED` requires routing rule removal in addition to cache invalidation; the other two are cache-bust only
- `payload.affectedCount` → informational; log for observability but do not use for routing logic
- `payload.segmentName` → informational label; do not use as a cache key — `segmentId` is the stable identifier

## What Changes When The Contract Changes

### If producer adds a new field to a REST response
Notification Service's Go JSON decoder ignores unknown fields. No action required unless the new field is one Notification Service should act on. Check this skill for an updated field guide before shipping.

### If `email` is renamed in GET /stores/:id/manager
This is the most dangerous possible change in this contract — `email` maps directly to `Notification.recipientId` which is the dispatch address. Protocol:
1. Catalog Service bumps path to `/v2/stores/:id/manager`
2. Update `CatalogClient.GetStoreManager()` to call `/v2/` path
3. Update the response struct to use the new field name
4. Keep `/v1/` support in place until Catalog Service retires it
5. Notify facilitator — this change also affects Eligibility Service if it reads store data

### If `changeType` enum gets a new value in `SegmentUpdated`
Add a case to the `switch` in `CustomerEventTranslator.HandleSegmentUpdated()`. The `default` arm already logs unknown values — but a new value that requires action (e.g. `SEGMENT_RENAMED`) must not silently fall through. Review the contract skill update carefully before adding the default fallthrough.

### If `divisionId` parameter is renamed in GET /stores
Update the query string construction in `CatalogClient.GetStoresByDivision()`. This must be coordinated — Catalog Service must deploy first and support both parameter names during the transition window.

## Blast Radius Checklist

When you receive a `skill-contract-notification-catalog` update — check these files in your repo:
- [ ] `infrastructure/acl/catalog_client.go` — both REST methods, response structs, error mapping
- [ ] `infrastructure/acl/customer_event_translator.go` — `SegmentUpdated` handler and `changeType` switch
- [ ] `domain/model/notification.go` — `RecipientID` field sourced from `email`; `RecipientRole`; `DeliveryChannel`
- [ ] `domain/service/notification_router.go` — how it calls the ACL client and builds `Notification` aggregates
- [ ] `infrastructure/cache/recipient_cache.go` — cache key format (`segmentId`, `divisionId`)
- [ ] `infrastructure/delivery/delivery_service.go` — if `DeliveryService.Send()` depends on a field that changed
- [ ] `test/acl/catalog_client_test.go` — stub responses for 200, 400, 404 (both variants), 500
- [ ] `test/acl/customer_event_translator_test.go` — fixture JSON for all three `changeType` values

## Common Mistakes

**Mistake 1 — Failing the entire notification batch when one store returns `MANAGER_NOT_ASSIGNED`.**
The contract explicitly requires Notification Service to skip stores with no manager assigned and continue delivering to other stores in the division. A single missing manager must not prevent the other 3 stores in a division from receiving their notifications. Log a warning, skip, and continue.

**Mistake 2 — Querying Customer Service on receipt of `SegmentUpdated`.**
This event is a cache-bust signal. Notification Service must only invalidate its recipient cache — it does not call Customer Service to fetch the updated segment data. The next routing operation will fetch fresh data naturally.

**Mistake 3 — Making the store manager calls sequentially instead of in parallel.**
For a division with 4 stores, sequential `GET /stores/:id/manager` calls add 4× the latency of a single call. The contract end-to-end example explicitly notes these calls "can be parallelised". Use goroutines with a WaitGroup or errgroup — this is especially important for the `division-all` scope where all stores in the tenant are in scope.

## Validation

| Scenario | Input | Expected |
|----------|-------|----------|
| `CampaignPaused` for `camp-001` (`geoScope: midwest`) | `GET /stores?divisionId=division-midwest&tenantId=tenant-kroger-001` | Returns 4 stores: chicago, columbus, indy, detroit |
| — | `GET /stores/store-chicago-001/manager` | `email: j.michaels@kroger.com`, `managerId: mgr-chicago-001` |
| — | `GET /stores/store-chicago-002/manager` | `404 MANAGER_NOT_ASSIGNED` → log warning, skip, continue |
| Full flow | `CampaignPaused` for `camp-001` | 4 `Notification` aggregates created with `recipientId` = manager emails, `status: PENDING`, dispatched via `DeliveryService.Send()` |
| `SegmentUpdated` | `changeType: MEMBER_ADDED`, `segmentId: seg-platinum-001` | `recipientCache.InvalidateBySegment(tenantID, "seg-platinum-001")` called; no Customer Service call made |
| `SegmentUpdated` | `changeType: SEGMENT_DELETED`, `segmentId: seg-platinum-001` | Routing rules referencing `seg-platinum-001` removed; cache invalidated |
| `GET /stores` | `divisionId` missing from query | `400 MISSING_PARAM` → `ErrMissingParam` returned to caller |
| `GET /stores` | Unknown `divisionId` | `404 DIVISION_NOT_FOUND` → `ErrDivisionNotFound` returned to caller |
