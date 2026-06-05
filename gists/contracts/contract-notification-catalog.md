# Contract — Notification ↔ Catalog & Customer

> Producer: Catalog Service (Team 4) + Customer Service (Team 5)
> Consumer: Notification Service (Team 6)
> Type: REST API (synchronous) + Domain Event (Redis Pub/Sub, one event)
> Status: LOCKED — changes require version bump + contract skill update

---

## Overview

Notification Service calls Catalog Service REST APIs to resolve store manager recipients
when routing geo-scoped notifications. It cannot build a recipient list from campaign events
alone because events carry division identifiers (`geoScope`), not individual store manager
contacts. Notification Service must call Catalog Service to expand a division into its
constituent stores and retrieve each store manager's contact details.

There is also one inbound event: `SegmentUpdated` (published by Customer Service),
which Notification Service consumes to invalidate and refresh recipient caches when
customer segment membership changes.

---

## REST APIs — Catalog Service

### GET /stores?divisionId=X&tenantId=Y

**Called by:** Notification Service
**Implemented by:** Catalog Service (Team 4)
**Purpose:** Expand a division identifier into the list of stores within it, so Notification
Service can retrieve each store manager's contact details.

**Request:**

```
GET /stores?divisionId={divisionId}&tenantId={tenantId}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `divisionId` | string | yes | Division identifier from campaign `geoScope` (e.g. `division-midwest`) |
| `tenantId` | string | yes | Tenant scoping — all responses are tenant-isolated |

**Response 200 OK:**

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

**Example — when CampaignPaused fires for camp-001 (geoScope: midwest):**

```
GET /stores?divisionId=division-midwest&tenantId=tenant-kroger-001
```

```json
{
  "divisionId": "division-midwest",
  "tenantId": "tenant-kroger-001",
  "stores": [
    { "storeId": "store-chicago-001",   "name": "Kroger Chicago Lincoln Park", "divisionId": "division-midwest", "city": "Chicago",    "state": "IL" },
    { "storeId": "store-columbus-001",  "name": "Kroger Columbus Easton",      "divisionId": "division-midwest", "city": "Columbus",   "state": "OH" },
    { "storeId": "store-indy-001",      "name": "Kroger Indianapolis Broad Ripple", "divisionId": "division-midwest", "city": "Indianapolis", "state": "IN" },
    { "storeId": "store-detroit-001",   "name": "Kroger Detroit Midtown",      "divisionId": "division-midwest", "city": "Detroit",    "state": "MI" }
  ]
}
```

Notification Service then calls `GET /stores/{storeId}/manager` for each `storeId`
in this list to retrieve store manager email addresses for the `CampaignPaused` notification.

**Error Responses:**

| HTTP Status | Error Code | Condition |
|-------------|------------|-----------|
| 400 | `MISSING_PARAM` | `divisionId` or `tenantId` is absent |
| 404 | `DIVISION_NOT_FOUND` | No division matches `divisionId` for the given `tenantId` |
| 500 | `INTERNAL_ERROR` | Catalog Service internal failure |

**400 Example:**

```json
{
  "error": "MISSING_PARAM",
  "message": "divisionId is required"
}
```

**404 Example:**

```json
{
  "error": "DIVISION_NOT_FOUND",
  "message": "No division found for divisionId=division-xyz and tenantId=tenant-kroger-001"
}
```

---

### GET /stores/:id/manager

**Called by:** Notification Service
**Implemented by:** Catalog Service (Team 4)
**Purpose:** Retrieve store manager contact details for a specific store, so Notification
Service can address an email or webhook notification to the correct recipient.

**Request:**

```
GET /stores/{storeId}/manager
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `storeId` | path param | yes | Store identifier (from GET /stores response) |

**Response 200 OK:**

```json
{
  "storeId": "string",
  "managerId": "string",
  "name": "string",
  "email": "string",
  "phone": "string | null"
}
```

**Example:**

```
GET /stores/store-chicago-001/manager
```

```json
{
  "storeId": "store-chicago-001",
  "managerId": "mgr-chicago-001",
  "name": "Jordan Michaels",
  "email": "j.michaels@kroger.com",
  "phone": null
}
```

Notification Service uses `email` from this response to populate the `recipientId` field
on the `Notification` aggregate before dispatch via `DeliveryService`.

**Error Responses:**

| HTTP Status | Error Code | Condition |
|-------------|------------|-----------|
| 404 | `STORE_NOT_FOUND` | `storeId` does not exist |
| 404 | `MANAGER_NOT_ASSIGNED` | Store exists but has no manager record |
| 500 | `INTERNAL_ERROR` | Catalog Service internal failure |

**404 Manager Not Assigned Example:**

```json
{
  "error": "MANAGER_NOT_ASSIGNED",
  "message": "Store store-chicago-002 has no manager assigned. Notification cannot be delivered."
}
```

> Notification Service must handle `MANAGER_NOT_ASSIGNED` gracefully: log a warning,
> skip this store, and continue delivering to other stores in the division.
> Do not fail the entire notification batch for one missing manager.

---

## End-to-End Example — CampaignPaused for camp-001

**Scenario:** camp-001 is paused with `reason: BUDGET_EXHAUSTED`. Campaign `geoScope` is `["midwest"]`.

Notification Service receives `CampaignPaused` on channel `promotionos.campaign.paused`.

**Step 1 — Resolve division to stores:**

```
GET /stores?divisionId=division-midwest&tenantId=tenant-kroger-001
```

Returns: chicago, columbus, indy, detroit stores.

**Step 2 — Resolve store manager contacts (4 calls, can be parallelised):**

```
GET /stores/store-chicago-001/manager  → j.michaels@kroger.com
GET /stores/store-columbus-001/manager → a.chen@kroger.com
GET /stores/store-indy-001/manager    → b.washington@kroger.com
GET /stores/store-detroit-001/manager → c.rodriguez@kroger.com
```

**Step 3 — Create Notification records and dispatch:**

Notification Service creates 4 `Notification` aggregate instances:

```
Notification { recipientId: "j.michaels@kroger.com",   recipientRole: STORE_MANAGER, channel: EMAIL, eventType: "CampaignPaused", status: PENDING }
Notification { recipientId: "a.chen@kroger.com",        recipientRole: STORE_MANAGER, channel: EMAIL, eventType: "CampaignPaused", status: PENDING }
Notification { recipientId: "b.washington@kroger.com",  recipientRole: STORE_MANAGER, channel: EMAIL, eventType: "CampaignPaused", status: PENDING }
Notification { recipientId: "c.rodriguez@kroger.com",   recipientRole: STORE_MANAGER, channel: EMAIL, eventType: "CampaignPaused", status: PENDING }
```

Each is passed to `DeliveryService.Send(notification)`.

**Step 4 — Update status:**

On successful delivery, each Notification's `status` transitions to `DELIVERED` and
`deliveredAt` is set. On failure, `status` transitions to `FAILED`.

---

## Domain Event — SegmentUpdated

**Channel:** `promotionos.customer.segment.updated`
**Published by:** Customer Service (Team 5)
**Consumed by:** Notification Service (Team 6)
**Purpose:** When customer segment membership changes, Notification Service must invalidate
any cached recipient lists that were derived from segment data, so the next notification
routes to the current segment members.

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

**Example:**

```json
{
  "eventId": "d4e5f6a7-bcde-f012-3456-789abcdef012",
  "tenantId": "tenant-kroger-001",
  "occurredAt": "2025-03-01T08:00:00Z",
  "schemaVersion": 1,
  "payload": {
    "segmentId": "seg-platinum-001",
    "segmentName": "PLATINUM",
    "changeType": "MEMBER_ADDED",
    "affectedCount": 142
  }
}
```

**ACL Translation (Notification Service):**
- `payload.segmentId` → invalidate recipient cache entries tagged with this segment
- `payload.changeType` → if `SEGMENT_DELETED`, remove all routing rules referencing this segment
- Notification Service does NOT query Customer Service on receipt of this event —
  it only invalidates the cache so the next routing call fetches fresh data

---

## Channel Summary

| Surface | Direction | Channel / Endpoint |
|---------|-----------|-------------------|
| GET /stores | Notification → Catalog | REST |
| GET /stores/:id/manager | Notification → Catalog | REST |
| SegmentUpdated | Customer → Notification | `promotionos.customer.segment.updated` |

---

## Breaking Change Protocol

If Catalog Service or Customer Service needs to change any response schema or event schema:

1. For REST responses — add a `version` path segment (e.g. `/v2/stores`) and maintain `/v1/stores` for the transition window. Notify Team 6 before deploying.
2. For event schemas — bump `schemaVersion` and update this file with what changed.
3. Both sides must support old + new schema during the transition window.
4. Notify facilitator — changes here may affect Eligibility Service if it also reads store data.

**Fields that are safe to add (non-breaking):**
- New optional fields in REST responses (Notification Service ignores unknown fields)
- New optional fields in events with null defaults

**Fields that are breaking changes:**
- Removing `email` or `managerId` from GET /stores/:id/manager response
- Renaming `storeId` in any response
- Removing any field from SegmentUpdated payload
- Changing `changeType` enum values
