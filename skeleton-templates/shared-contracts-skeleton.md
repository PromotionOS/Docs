# Shared Contracts Repo — Skeleton Build Template

> Language: JSON Schema
> Purpose: Single source of truth for all event and API contracts
> All services reference these schemas for contract validation
> Deploy: GitHub repo only — no service deployment

---

## Repo Structure

```
shared-contracts/
├── events/
│   ├── CampaignPublished.json
│   ├── CampaignPaused.json
│   ├── BudgetExhausted.json
│   ├── BudgetUpdated.json
│   ├── CatalogItemExcluded.json
│   ├── SegmentUpdated.json
│   ├── OfferRedeemed.json
│   └── ClaimSubmitted.json
├── api/
│   ├── campaign-service-api.json
│   ├── eligibility-service-api.json
│   ├── redemption-service-api.json
│   ├── catalog-customer-api.json
│   └── analytics-service-api.json
├── CHANGELOG.md
└── README.md
```

---

## events/CampaignPublished.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CampaignPublished",
  "description": "Emitted when a Campaign transitions from DRAFT to ACTIVE. Consumed by Eligibility and Analytics.",
  "version": "1.0.0",
  "channel": "promotionos.campaign.published",
  "publisher": "campaign-service",
  "consumers": ["eligibility-service", "analytics-service", "event-store"],
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["offerId", "offerType", "dateRange", "stackPermission", "stackLimit", "geoScope"],
      "properties": {
        "offerId": { "type": "string", "format": "uuid" },
        "offerType": { "type": "string", "enum": ["PCT_OFF", "AMT_OFF", "BOGO", "THRESHOLD"] },
        "offerValue": { "type": ["number", "null"] },
        "thresholdAmount": { "type": ["number", "null"] },
        "discountAmount": { "type": ["number", "null"] },
        "upcScope": { "type": "array", "items": { "type": "string" } },
        "dateRange": {
          "type": "object",
          "required": ["startDate", "endDate"],
          "properties": {
            "startDate": { "type": "string", "format": "date" },
            "endDate": { "type": "string", "format": "date" }
          }
        },
        "stackPermission": { "type": "boolean" },
        "stackLimit": { "type": "integer", "minimum": 1, "default": 1 },
        "segmentRestriction": { "type": ["string", "null"] },
        "geoScope": { "type": "array", "items": { "type": "string" }, "minItems": 1 },
        "exclusions": { "type": "array", "items": { "type": "string" }, "default": [] }
      }
    }
  }
}
```

---

## events/CampaignPaused.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CampaignPaused",
  "channel": "promotionos.campaign.paused",
  "publisher": "campaign-service",
  "consumers": ["eligibility-service", "frontend-dashboard", "event-store"],
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["reason", "pausedAt"],
      "properties": {
        "reason": { "type": "string", "enum": ["BUDGET_EXHAUSTED", "MANUAL_PAUSE", "CAMPAIGN_EXPIRED"] },
        "pausedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## events/BudgetExhausted.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BudgetExhausted",
  "channel": "promotionos.analytics.budget.exhausted",
  "publisher": "analytics-service",
  "consumers": ["campaign-service", "event-store"],
  "note": "Published EXACTLY ONCE per campaign when budgetBurnPercent >= 95.0",
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["totalAmount", "burnedAmount", "budgetBurnPercent", "redemptionCount", "exhaustedAt"],
      "properties": {
        "totalAmount": { "type": "number", "minimum": 0 },
        "burnedAmount": { "type": "number", "minimum": 0 },
        "budgetBurnPercent": { "type": "number", "minimum": 95.0 },
        "redemptionCount": { "type": "integer", "minimum": 0 },
        "exhaustedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## events/BudgetUpdated.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BudgetUpdated",
  "channel": "promotionos.campaign.budget.updated",
  "publisher": "campaign-service",
  "consumers": ["analytics-service", "event-store"],
  "note": "Published when MX team tops up a campaign budget. reactivated=true resets BudgetExhausted flag.",
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["previousTotalAmount", "newTotalAmount", "burnedAmount", "newBudgetBurnPercent", "reactivated"],
      "properties": {
        "previousTotalAmount": { "type": "number" },
        "newTotalAmount": { "type": "number" },
        "burnedAmount": { "type": "number" },
        "newBudgetBurnPercent": { "type": "number" },
        "reactivated": { "type": "boolean" }
      }
    }
  }
}
```

---

## events/CatalogItemExcluded.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CatalogItemExcluded",
  "channel": "promotionos.catalog.item.excluded",
  "publisher": "catalog-customer-service",
  "consumers": ["eligibility-service", "event-store"],
  "type": "object",
  "required": ["eventId", "tenantId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["categoryId", "excluded", "excludeReason", "affectedUPCs"],
      "properties": {
        "categoryId": { "type": "string" },
        "categoryName": { "type": "string" },
        "excluded": { "type": "boolean" },
        "excludeReason": { "type": "string", "enum": ["REGULATORY", "REGULATORY_INHERITED", "MANUAL"] },
        "inheritedFrom": { "type": ["string", "null"] },
        "affectedUPCs": { "type": "array", "items": { "type": "string" } },
        "affectedChildCategories": { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

---

## events/SegmentUpdated.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "SegmentUpdated",
  "channel": "promotionos.customer.segment.updated",
  "publisher": "catalog-customer-service",
  "consumers": ["eligibility-service", "event-store"],
  "note": "Published ONLY when segments actually change. Not published if unchanged.",
  "type": "object",
  "required": ["eventId", "tenantId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["customerId", "previousSegments", "currentSegments", "loyaltyTier"],
      "properties": {
        "customerId": { "type": "string", "format": "uuid" },
        "previousSegments": { "type": "array", "items": { "type": "string" } },
        "currentSegments": { "type": "array", "items": { "type": "string" } },
        "loyaltyTier": { "type": "string", "enum": ["BASIC", "SILVER", "GOLD", "PLATINUM"] },
        "annualSpend": { "type": "number" },
        "changedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## events/OfferRedeemed.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OfferRedeemed",
  "channel": "promotionos.redemption.redeemed",
  "publisher": "redemption-service",
  "consumers": ["analytics-service", "event-store"],
  "warning": "Analytics Service is pre-built against this schema. Any deviation breaks the burn tracker.",
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["redemptionId", "customerId", "discountApplied", "cartTotal", "redeemedAt"],
      "properties": {
        "redemptionId": { "type": "string", "format": "uuid" },
        "customerId": { "type": "string", "format": "uuid" },
        "discountApplied": { "type": "number", "minimum": 0 },
        "cartTotal": { "type": "number", "minimum": 0 },
        "storeId": { "type": "string" },
        "division": { "type": "string" },
        "redeemedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## events/ClaimSubmitted.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ClaimSubmitted",
  "channel": "promotionos.redemption.claim.submitted",
  "publisher": "redemption-service",
  "consumers": ["analytics-service", "event-store"],
  "type": "object",
  "required": ["eventId", "tenantId", "campaignId", "occurredAt", "schemaVersion", "payload"],
  "properties": {
    "eventId": { "type": "string", "format": "uuid" },
    "tenantId": { "type": "string" },
    "campaignId": { "type": "string", "format": "uuid" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "schemaVersion": { "type": "integer", "const": 1 },
    "payload": {
      "type": "object",
      "required": ["claimId", "redemptionId", "vendorId", "claimAmount", "deductionAmount", "submittedAt"],
      "properties": {
        "claimId": { "type": "string", "format": "uuid" },
        "redemptionId": { "type": "string", "format": "uuid" },
        "vendorId": { "type": "string" },
        "claimAmount": { "type": "number", "minimum": 0 },
        "deductionAmount": { "type": "number", "minimum": 0 },
        "submittedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## CHANGELOG.md

```markdown
# Contract Changelog

## v1.0.0 — Initial release

All 8 events at schemaVersion 1.

### Breaking change protocol
1. Bump schemaVersion in the JSON schema file
2. Update the relevant contract skill gist
3. Notify consuming teams via GitHub PR on this repo
4. Both producer and consumer must be updated before deploying

### Safe changes (non-breaking)
- Adding optional fields with null/default values

### Breaking changes (require version bump)
- Renaming any field
- Changing a field type
- Removing any field
- Changing enum values
- Changing channel name
```

---

## README.md

```markdown
# Shared Contracts

Single source of truth for all PromotionOS event and API contracts.

## Event Schemas
All events are in `/events/`. Each schema includes:
- Channel name
- Publisher
- Consumers
- Required fields
- Schema version

## API Contracts
REST API contracts are in `/api/`. Match these exactly in your controllers.

## Contract Skill
Each boundary has a corresponding contract skill gist that explains:
- What changed between versions
- Blast radius of changes
- ACL translation rules for consumers

## Rule
If you change a contract — open a PR here first.
No service deploys a contract change before this repo is updated.
```
