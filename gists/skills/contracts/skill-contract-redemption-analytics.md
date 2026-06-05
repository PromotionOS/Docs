---
name: contract-redemption-analytics
description: Contract skill for Analytics Service consuming from Redemption Service — ACL translation, field mapping, blast radius guide
metadata:
  type: reference
---

# Contract: Redemption → Analytics

## What This Contract Covers

Redemption Service publishes two domain events to Redis Pub/Sub after a Redemption is confirmed. Analytics Service (pre-built) consumes both events to drive all budget burn, lift, ROI, and funding cost metrics. This is the most critical event contract in the platform — Analytics is pre-built and cannot change to accommodate Redemption Service schema drift.

## Events / APIs (Summary)

| Event | Channel | Purpose |
|-------|---------|---------|
| `OfferRedeemed` | `promotionos.redemption.redeemed` | Primary metric driver — budget burn, lift, ROI |
| `ClaimSubmitted` | `promotionos.redemption.claim.submitted` | Funding cost tracking for ROI calculation |

## ACL Translation Guide

### OfferRedeemed

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "redemptionId": "uuid",
    "customerId": "uuid",
    "discountApplied": "number",
    "cartTotal": "number",
    "storeId": "string",
    "division": "string",
    "redeemedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL — Redemption Service (producer side):**
```java
// infrastructure/events/RedemptionEventPublisher.java

public void publishOfferRedeemed(Redemption redemption) {
    var payload = OfferRedeemedPayload.builder()
        .redemptionId(redemption.getId().toString())
        .customerId(redemption.getCustomerId().toString())
        .discountApplied(redemption.getDiscountApplied().getAmount())   // BigDecimal → double
        .cartTotal(redemption.getCartTotal().getAmount())
        .storeId(redemption.getStoreId())
        .division(redemption.getDivision())
        .redeemedAt(redemption.getRedeemedAt().toString())              // ISO-8601
        .build();

    var event = DomainEvent.<OfferRedeemedPayload>builder()
        .eventId(UUID.randomUUID().toString())
        .tenantId(redemption.getTenantId())
        .campaignId(redemption.getCampaignId().toString())
        .occurredAt(Instant.now().toString())
        .schemaVersion(1)                                               // must be exactly 1
        .payload(payload)
        .build();

    redisPublisher.publish("promotionos.redemption.redeemed", event);
}
```

**How Analytics Service consumes this (ACL inside Analytics):**
```go
// internal/acl/redemption_event_translator.go

func (t *RedemptionEventTranslator) HandleOfferRedeemed(event OfferRedeemedEvent) error {
    metrics, err := t.metricsRepo.FindByCampaignID(event.TenantID, event.CampaignID)
    if err != nil {
        return fmt.Errorf("finding metrics for campaign %s: %w", event.CampaignID, err)
    }

    // Increment burned amount and redemption count.
    metrics.BurnedAmount += event.Payload.DiscountApplied
    metrics.RedemptionCount++
    metrics.BudgetBurnPercent = (metrics.BurnedAmount / metrics.TotalAmount) * 100

    // Update sales rolling average.
    metrics.ActualSalesPerDay = t.rollingSales.Update(
        event.CampaignID,
        event.Payload.CartTotal,
        event.Payload.RedeemedAt,
    )

    // Recalculate lift and ROI.
    metrics.Lift = metrics.ActualSalesPerDay - metrics.BaselineSalesPerDay
    if metrics.TotalFundingCost > 0 {
        metrics.ROI = (metrics.Lift * avgMarginPct) / metrics.TotalFundingCost
    }

    if err := t.metricsRepo.Save(metrics); err != nil {
        return fmt.Errorf("saving metrics: %w", err)
    }

    // Emit BudgetExhausted exactly once when burn crosses 95%.
    if metrics.BudgetBurnPercent >= 95.0 && !metrics.BudgetExhaustedEmitted {
        metrics.BudgetExhaustedEmitted = true
        t.metricsRepo.Save(metrics) // save flag before emitting
        t.budgetExhaustedEmitter.Emit(event.TenantID, event.CampaignID, metrics)
    }

    return nil
}
```

**Fields to watch:**
- `payload.discountApplied` → added to `burnedAmount`; this drives the 95% threshold check — must be a positive number representing the dollar amount of the discount
- `payload.redeemedAt` → ISO-8601; used for time-series aggregation of `actualSalesPerDay`; must be the redemption timestamp, not `occurredAt`
- `campaignId` → used to look up the campaign metrics record; must match the UUID stored when `CampaignPublished` was processed
- `schemaVersion` → must be `1`; Analytics Service will reject events with an unknown schema version

---

### ClaimSubmitted

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "claimId": "uuid",
    "redemptionId": "uuid",
    "vendorId": "string",
    "claimAmount": "number",
    "deductionAmount": "number",
    "submittedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL — Redemption Service (producer side):**
```java
// infrastructure/events/RedemptionEventPublisher.java

public void publishClaimSubmitted(Claim claim, Redemption redemption) {
    var payload = ClaimSubmittedPayload.builder()
        .claimId(claim.getId().toString())
        .redemptionId(redemption.getId().toString())
        .vendorId(claim.getVendorId())
        .claimAmount(claim.getAmount().getAmount())           // total claim value
        .deductionAmount(claim.getDeduction().getAmount())   // vendorShare portion
        .submittedAt(claim.getSubmittedAt().toString())
        .build();

    var event = DomainEvent.<ClaimSubmittedPayload>builder()
        .eventId(UUID.randomUUID().toString())
        .tenantId(redemption.getTenantId())
        .campaignId(redemption.getCampaignId().toString())
        .occurredAt(Instant.now().toString())
        .schemaVersion(1)
        .payload(payload)
        .build();

    redisPublisher.publish("promotionos.redemption.claim.submitted", event);
}
```

**Fields to watch:**
- `payload.claimAmount` → added to `totalFundingCost` on campaign metrics; triggers ROI recalculation
- `payload.deductionAmount` → tracked separately for vendor reconciliation reporting; formula is `discountApplied × vendorShare%` — this is `Claim.deduction` in the domain model
- `payload.submittedAt` → must be T+24h from the original Redemption's `redeemedAt`; Analytics uses this for funding cost timing reports
- `payload.redemptionId` → used by Analytics for deduplication; publishing the same `ClaimSubmitted` twice for the same redemption must be guarded by an idempotency check

## What Changes When The Contract Changes

### If producer adds a new optional field
Add the field to the event POJO and publish it. Analytics Service (Go) uses a lenient JSON decoder — new optional fields will be ignored safely. No `schemaVersion` bump required.

### If producer renames a field
**Do not rename any field without coordinating with Team 5 first.** Analytics is pre-built. A renamed field means Analytics silently loses the data. Protocol:
1. Discuss with Team 5
2. Bump `schemaVersion` to `2`
3. Update this skill
4. Team 5 deploys updated consumer before Team 3 deploys updated producer

### If producer removes a field
Same protocol as rename — coordinate, bump version, Team 5 deploys first.

### If channel name changes
Both `promotionos.redemption.redeemed` and `promotionos.redemption.claim.submitted` are hardcoded in Analytics Service. Channel renames are the most dangerous change — treat as a breaking deployment requiring both services to be updated in the same deployment window.

## Blast Radius Checklist

When you receive a `skill-contract-redemption-analytics` update — check these files in your repo:
- [ ] `infrastructure/events/RedemptionEventPublisher.java` — event construction and field values
- [ ] `domain/model/Redemption.java` — `discountApplied`, `cartTotal`, `storeId`, `division` fields
- [ ] `domain/model/Claim.java` — `amount`, `deduction`, `vendorId` fields
- [ ] `infrastructure/events/OfferRedeemedPayload.java` — POJO field names match schema exactly
- [ ] `infrastructure/events/ClaimSubmittedPayload.java` — POJO field names match schema exactly
- [ ] `test/events/RedemptionEventPublisherTest.java` — verify event JSON serialization matches contract exactly
- [ ] Any hardcoded channel name strings (`"promotionos.redemption.redeemed"`)

## Common Mistakes

**Mistake 1 — Publishing `ClaimSubmitted` immediately at confirmation instead of at T+24h.**
The domain rule is: Claim is submitted 24 hours after Redemption is confirmed. Implementing this as a synchronous call in `RedemptionService.confirm()` publishes the event at T+0, which causes Analytics to record funding costs before the claim is actually submitted to the Vendor.

**Mistake 2 — Using `occurredAt` instead of `payload.redeemedAt` for time-series data.**
`occurredAt` is the event publication timestamp; `payload.redeemedAt` is when the transaction occurred at the POS. Analytics uses `redeemedAt` for `actualSalesPerDay` calculation. They will be close in practice but are semantically different — always populate `redeemedAt` from `Redemption.redeemedAt`.

**Mistake 3 — Not setting `schemaVersion: 1` on the event.**
Analytics Service validates `schemaVersion`. An event published without this field (or with `null`) will be rejected or silently dropped by the Analytics consumer. Set it explicitly — do not rely on a default.

## Validation

| Scenario | Redemption Event | Expected Analytics Effect |
|----------|-----------------|--------------------------|
| — | `redeem-001`: `camp-001`, `discountApplied: 6.99` | `burnedAmount += 6.99`, `redemptionCount++` |
| 8 | `camp-001` burn reaches 95.0% | `BudgetExhausted` emitted exactly once, `budgetExhaustedEmitted` flag set to true |
| 26 | `camp-012` burn reaches exactly 95.0% | `BudgetExhausted` emitted |
| 27 | Subsequent redemption against `camp-001` after pause | `BudgetExhausted` NOT re-emitted (`budgetExhaustedEmitted` flag guards it) |
| 28 | `redeem-001` at T+0 | `ClaimSubmitted` event appears at T+24h |
| 29 | `camp-001` funding: `vendorShare: 60%`, `discountApplied: 6.99` | `deductionAmount: 4.19` |
