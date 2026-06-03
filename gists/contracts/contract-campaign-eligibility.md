# Contract — Campaign → Eligibility

> Producer: Campaign Service (Team 1)
> Consumer: Eligibility Service (Team 2)
> Type: Domain Events (Redis Pub/Sub)
> Status: LOCKED — changes require schemaVersion bump + contract skill update

---

## Events Published by Campaign Service

### CampaignPublished

Emitted when a Campaign transitions from DRAFT to ACTIVE via `Campaign.publish()`.
Eligibility Service must load eligibility rules into its rule engine upon receiving this event.

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

**ACL Translation (Eligibility Service):**
- `payload.upcScope` → `EligibilityRule.upcScope`
- `payload.stackLimit` → `EligibilityRule.stackLimit`
- `payload.segmentRestriction` → `EligibilityRule.segmentRestriction`
- `payload.exclusions` → `EligibilityRule.exclusions` (list of categoryIds)
- `payload.geoScope` → `EligibilityRule.geoScope`
- `payload.dateRange` → `EligibilityRule.dateRange` (used to filter active rules)

---

### CampaignPaused

Emitted when a Campaign transitions to PAUSED (either by MX team or auto-pause via BudgetExhausted).
Eligibility Service must immediately remove this Campaign's rules from its rule engine.

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

**ACL Translation (Eligibility Service):**
- On receipt → remove all `EligibilityRule` records for `campaignId` from active rule engine
- No further offers served for this Campaign

---

### BudgetExhausted

Emitted by Analytics Service, consumed by Campaign Service which then emits CampaignPaused.
Eligibility Service does NOT consume BudgetExhausted directly — it consumes CampaignPaused.

> Note: This event flows Analytics → Campaign → (CampaignPaused) → Eligibility.
> Eligibility never directly handles BudgetExhausted.

---

## Redis Channel Names

| Event | Channel |
|-------|---------|
| CampaignPublished | `promotionos.campaign.published` |
| CampaignPaused | `promotionos.campaign.paused` |

---

## Breaking Change Protocol

If Team 1 needs to change any field in these event schemas:

1. Bump `schemaVersion` in the event (e.g. `1 → 2`)
2. Update `skill-contract-campaign-eligibility.md` with what changed
3. Team 2 uses the updated contract skill to identify ACL blast radius
4. Team 2 updates their ACL translation before merging
5. Both services must support old + new schema version during transition

**Fields that are safe to add (non-breaking):**
- New optional fields with null defaults

**Fields that are breaking changes:**
- Renaming any existing field
- Changing a field's type
- Removing any field
- Changing enum values

---

## Validation

Team 2 must validate their ACL against these test data campaigns:

| Campaign | Event | Expected ACL output |
|----------|-------|-------------------|
| camp-001 | CampaignPublished | stackLimit:1, exclusions:[cat-alcohol, cat-tobacco], geo:[midwest,southeast] |
| camp-004 | CampaignPublished | segmentRestriction:PLATINUM, exclusions:[], stackLimit:1 |
| camp-006 | CampaignPaused | rules removed from engine, BUDGET_EXHAUSTED reason |
| camp-002 | CampaignPublished | stackLimit:2, stackPermission:true, geo:[division-all] |
