---
name: contract-campaign-eligibility
description: Contract skill for Eligibility Service consuming from Campaign Service — ACL translation, field mapping, blast radius guide
metadata:
  type: reference
---

# Contract: Campaign → Eligibility

## What This Contract Covers

Campaign Service publishes domain events to Redis Pub/Sub when campaign lifecycle transitions occur. Eligibility Service consumes these events to load and unload eligibility rules in its rule engine — this is the only mechanism by which Eligibility learns which campaigns are active and what their rules are.

## Events / APIs (Summary)

| Event | Channel | Purpose |
|-------|---------|---------|
| `CampaignPublished` | `promotionos.campaign.published` | Load eligibility rules into rule engine |
| `CampaignPaused` | `promotionos.campaign.paused` | Remove rules from rule engine immediately |

Note: Eligibility Service does NOT consume `BudgetExhausted` directly. That flows Analytics → Campaign → `CampaignPaused` → Eligibility.

## ACL Translation Guide

### CampaignPublished

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "offerId": "uuid",
    "offerType": "PCT_OFF | AMT_OFF | BOGO | THRESHOLD",
    "offerValue": "number | null",
    "thresholdAmount": "number | null",
    "discountAmount": "number | null",
    "upcScope": ["string"],
    "dateRange": {
      "startDate": "YYYY-MM-DD",
      "endDate": "YYYY-MM-DD"
    },
    "stackPermission": "boolean",
    "stackLimit": "integer",
    "segmentRestriction": "BASIC | SILVER | GOLD | PLATINUM | null",
    "geoScope": ["string"],
    "exclusions": ["string"]
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/CampaignEventTranslator.java

public EligibilityRule translate(CampaignPublishedEvent event) {
    var payload = event.getPayload();

    DateRange dateRange = new DateRange(
        LocalDate.parse(payload.getDateRange().getStartDate()),
        LocalDate.parse(payload.getDateRange().getEndDate())
    );

    return EligibilityRule.builder()
        .campaignId(event.getCampaignId())
        .tenantId(event.getTenantId())
        .offerType(OfferType.valueOf(payload.getOfferType()))
        .upcScope(payload.getUpcScope())                         // List<String>
        .dateRange(dateRange)
        .stackPermission(payload.isStackPermission())
        .stackLimit(payload.getStackLimit())
        .segmentRestriction(                                     // nullable
            payload.getSegmentRestriction() != null
                ? LoyaltyTier.valueOf(payload.getSegmentRestriction())
                : null
        )
        .geoScope(payload.getGeoScope())                         // List<String> of division ids
        .exclusions(payload.getExclusions())                     // List<String> of categoryIds
        .build();
}
```

**Fields to watch:**
- `payload.upcScope` → `EligibilityRule.upcScope` — the list of UPC codes the offer applies to; an empty list means all UPCs qualify (not zero UPCs)
- `payload.stackLimit` → `EligibilityRule.stackLimit` — default is `1`; a value of `2` means the customer may stack this offer with one other
- `payload.segmentRestriction` → `EligibilityRule.segmentRestriction` — nullable; `null` means all loyalty tiers are eligible, not that no customers are eligible
- `payload.exclusions` → `EligibilityRule.exclusions` — list of `categoryId` strings (e.g. `cat-alcohol`); exclusion inheritance (parent → child categories) is resolved by Catalog Service before this event is emitted
- `payload.geoScope` → `EligibilityRule.geoScope` — list of division identifiers; `division-all` matches any customer regardless of their division
- `payload.dateRange` → `EligibilityRule.dateRange` — used by `RuleEngine` to filter which rules are active at evaluation time; do not pre-filter on ingest

---

### CampaignPaused

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "reason": "BUDGET_EXHAUSTED | MANUAL_PAUSE",
    "pausedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/CampaignEventTranslator.java

public void handleCampaignPaused(CampaignPausedEvent event) {
    // Remove ALL EligibilityRule records for this campaignId from the active engine.
    // The reason field is not used for routing logic — log it for audit only.
    ruleEngine.removeRulesForCampaign(event.getCampaignId(), event.getTenantId());

    log.info("Removed rules for campaign={} reason={} pausedAt={}",
        event.getCampaignId(),
        event.getPayload().getReason(),
        event.getPayload().getPausedAt()
    );
}
```

**Fields to watch:**
- `payload.reason` → `BUDGET_EXHAUSTED | MANUAL_PAUSE` — log for audit but do not branch logic on it; removal is unconditional
- `campaignId` — scope the removal to tenant + campaignId pair; never remove rules for a different tenant's campaign

## What Changes When The Contract Changes

### If producer adds a new optional field
Add the field to `CampaignPublishedEvent` POJO, map it in `CampaignEventTranslator`, and add a default in `EligibilityRule` so old events without the field still parse. No `schemaVersion` bump required for optional additions.

### If producer renames a field
`schemaVersion` bumps from `1 → 2`. Update `CampaignEventTranslator` to handle both versions during the transition window:
```java
if (event.getSchemaVersion() == 1) {
    // old field name
} else {
    // new field name
}
```
Remove the v1 branch only after Campaign Service stops publishing v1 events.

### If producer removes a field
This is a breaking change. Do not merge until:
1. Campaign Service has deployed with `schemaVersion: 2` and stopped publishing the removed field
2. `EligibilityRule` no longer requires that field for rule evaluation
3. `CampaignEventTranslator` is updated and all tests pass

## Blast Radius Checklist

When you receive a `skill-contract-campaign-eligibility` update — check these files in your repo:
- [ ] `infrastructure/acl/CampaignEventTranslator.java` — field mappings and schemaVersion branching
- [ ] `domain/model/EligibilityRule.java` — new/removed/renamed fields on the domain model
- [ ] `domain/engine/RuleEngine.java` — if offer evaluation logic depends on a changed field
- [ ] `infrastructure/messaging/CampaignEventListener.java` — channel name if it changed
- [ ] `test/acl/CampaignEventTranslatorTest.java` — all fixture JSON files that embed the old schema

## Common Mistakes

**Mistake 1 — Treating `null` segmentRestriction as "no customers eligible".**
A `null` `segmentRestriction` means the offer is open to all loyalty tiers. Guarding with `if (segmentRestriction != null) checkTier(...)` is correct. Blocking all customers when it is null is the opposite of intended behavior.

**Mistake 2 — Filtering `upcScope` on ingest instead of at evaluation time.**
`upcScope` defines which UPCs the offer applies to, but it must be evaluated against the customer's actual cart UPCs at check time. Do not resolve UPCs or filter the list when the `CampaignPublished` event arrives.

**Mistake 3 — Consuming `BudgetExhausted` directly to remove rules.**
Eligibility Service must only remove rules on `CampaignPaused`. If you subscribe to `promotionos.analytics.budget.exhausted`, you will remove rules before Campaign Service has a chance to confirm the pause — creating a race condition.

## Validation

Run these to confirm your consumer is correct:

| Scenario | Event | Expected ACL output |
|----------|-------|-------------------|
| camp-001 | `CampaignPublished` | `stackLimit: 1`, `exclusions: [cat-alcohol, cat-tobacco]`, `geoScope: [division-midwest, division-southeast]` |
| camp-002 | `CampaignPublished` | `stackLimit: 2`, `stackPermission: true`, `geoScope: [division-all]` |
| camp-004 | `CampaignPublished` | `segmentRestriction: PLATINUM`, `exclusions: []`, `stackLimit: 1` |
| camp-006 | `CampaignPaused` | Rules removed from engine, `reason: BUDGET_EXHAUSTED` logged |

Test strategy: feed the raw JSON fixture through `CampaignEventTranslator` and assert each field on the resulting `EligibilityRule`. Do not test `RuleEngine` behavior in the translator test — keep the translation and the evaluation separate.
