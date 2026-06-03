# Contract — Redemption → Analytics

> Producer: Redemption Service (Team 3)
> Consumer: Analytics Service (Team 5 — pre-built)
> Type: Domain Events (Redis Pub/Sub)
> Status: LOCKED — Analytics Service is pre-built against this contract. Any change breaks Analytics.

---

## Events Published by Redemption Service

### OfferRedeemed

Emitted immediately after a Redemption is confirmed. This is the primary event that drives Analytics — every metric update (burn, lift, ROI) is triggered by this event.

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

**Critical fields for Analytics:**
- `campaignId` — used to look up campaign metrics record
- `tenantId` — all analytics are tenant-scoped
- `payload.discountApplied` — added to `burnedAmount` in budget burn tracker
- `payload.redeemedAt` — used for time-series aggregation

**Analytics ACL Translation:**
- `discountApplied` → increment `burnedAmount` on campaign budget
- recalculate `budgetBurnPercent = (burnedAmount / totalAmount) × 100`
- if `budgetBurnPercent >= 95.0` → emit `BudgetExhausted` (exactly once)
- increment `redemptionCount`
- update `actualSalesPerDay` rolling average
- recalculate `lift` and `roi`

---

### ClaimSubmitted

Emitted 24 hours after Redemption confirmation when the Claim is submitted to the Vendor.
Analytics Service uses this for funding cost tracking.

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

**Analytics ACL Translation:**
- `payload.claimAmount` → increment `totalFundingCost` on campaign metrics
- `payload.deductionAmount` → tracked for vendor reconciliation reporting
- recalculate `roi` after funding cost update

---

## Redis Channel Names

| Event | Channel |
|-------|---------|
| OfferRedeemed | `promotionos.redemption.redeemed` |
| ClaimSubmitted | `promotionos.redemption.claim.submitted` |

---

## CRITICAL — Analytics is Pre-Built

Analytics Service (Team 5) is pre-built and already consuming these events with the exact schema above. Team 3 must publish events that match this contract exactly.

**Before merging any Redemption Service PR:**
1. Run the contract validation test in `shared-contracts` repo
2. Confirm `OfferRedeemed` event shape matches this spec field-by-field
3. Confirm `schemaVersion: 1` is set correctly

**If Team 3 must change the schema:**
1. Discuss with Team 5 first
2. Bump `schemaVersion` to 2
3. Update `skill-contract-redemption-analytics.md`
4. Team 5 must update their consumer before Team 3 deploys

---

## Breaking Change Protocol

**Safe to add:** New optional payload fields with null defaults
**Breaking:** Any field rename, type change, removal, or channel name change

---

## Validation

| Scenario | Redemption Event | Expected Analytics Effect |
|----------|-----------------|--------------------------|
| — | redeem-001: camp-001, discount:6.99 | burnedAmount += 6.99, redemptionCount++ |
| 8 | camp-001 burn reaches 95.0% | BudgetExhausted emitted by Analytics |
| 26 | camp-012 burn reaches exactly 95.0% | BudgetExhausted emitted |
| 27 | camp-001 subsequent redemption after pause | BudgetExhausted NOT re-emitted |
| 28 | redeem-001 at T+0 | ClaimSubmitted at T+24hrs |
| 29 | camp-001 funding: vendorShare 60%, discount:6.99 | deductionAmount: 4.19 |
