---
name: contract-analytics-campaign
description: Contract skill for Campaign Service consuming from Analytics Service — ACL translation, field mapping, blast radius guide
metadata:
  type: reference
---

# Contract: Analytics → Campaign

## What This Contract Covers

Analytics Service publishes a single `BudgetExhausted` event to Redis Pub/Sub when a Campaign's budget burn reaches or exceeds 95.0%. Campaign Service consumes this event and triggers `Campaign.pause(BUDGET_EXHAUSTED)`, which in turn emits `CampaignPaused` downstream to Eligibility Service and the Frontend. Both services are pre-built — neither side can change schema unilaterally.

## Events / APIs (Summary)

| Event | Channel | Purpose |
|-------|---------|---------|
| `BudgetExhausted` | `promotionos.analytics.budget.exhausted` | Trigger Campaign auto-pause when budget burn >= 95.0% |

## ACL Translation Guide

### BudgetExhausted

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "totalAmount": "number",
    "burnedAmount": "number",
    "budgetBurnPercent": "number",
    "redemptionCount": "integer",
    "exhaustedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/AnalyticsEventTranslator.java

public void handleBudgetExhausted(BudgetExhaustedEvent event) {
    String campaignId = event.getCampaignId();
    String tenantId = event.getTenantId();

    Campaign campaign = campaignRepository.findById(campaignId, tenantId)
        .orElseThrow(() -> new CampaignNotFoundException(campaignId));

    // Idempotency guard — if already PAUSED, skip silently.
    // Analytics emits BudgetExhausted exactly once, but Redemption events
    // arriving out of order could theoretically cause a duplicate.
    if (campaign.getStatus() == CampaignStatus.PAUSED) {
        log.warn("BudgetExhausted received for already-paused campaign={}", campaignId);
        return;
    }

    // The only required action: pause the campaign.
    campaign.pause(PauseReason.BUDGET_EXHAUSTED);
    campaignRepository.save(campaign);

    // Campaign.pause() emits CampaignPaused internally — no manual event publish needed here.
    log.info("Campaign auto-paused campaign={} burnPercent={} exhaustedAt={}",
        campaignId,
        event.getPayload().getBudgetBurnPercent(),
        event.getPayload().getExhaustedAt()
    );
}
```

**The threshold check that was buggy (Sprint 3):**
```java
// domain/service/BudgetTracker.java
// BUG: uses > 94 instead of >= 95.0
// FIXED version:
public boolean isExhausted(double budgetBurnPercent) {
    return budgetBurnPercent >= 95.0;   // correct: inclusive boundary
    // NOT: return budgetBurnPercent > 94;  ← this fires at 94.01%, too early
}
```

**Fields to watch:**
- `campaignId` → used to look up the `Campaign` aggregate; this is the primary correlation key
- `tenantId` → always scope the repository lookup to `campaignId + tenantId`; never look up by `campaignId` alone
- `payload.budgetBurnPercent` → the computed percentage at the moment of exhaustion; log it for the audit trail but do not re-evaluate the threshold — trust Analytics' calculation
- `payload.exhaustedAt` → ISO-8601 timestamp when the 95% threshold was crossed; store on the Campaign or an audit record if needed
- `payload.totalAmount` and `payload.burnedAmount` → informational context; Campaign Service does not need to update Budget from these fields — Budget state is owned by Analytics Service

## What Changes When The Contract Changes

### If producer adds a new optional field
Add the field to `BudgetExhaustedEvent` POJO with a null default. No action required beyond that — Campaign Service does not need to act on new analytics context fields unless the contract skill specifically says to.

### If producer renames a field
Both services are pre-built. Any rename requires:
1. Agreement between Team 1 and Team 5
2. `schemaVersion` bump to `2`
3. Update to this skill
4. Both services updated and deployed together — there is no transition window where only one side has deployed

### If the threshold changes (e.g., from 95.0% to 90.0%)
This is a domain rule change, not a schema change. The threshold is owned by Analytics Service (`BudgetBurnTracker`). Campaign Service does not enforce the threshold — it simply responds to `BudgetExhausted`. No schema change is needed; only Analytics needs to change.

### If the channel name changes
`promotionos.analytics.budget.exhausted` is the hardcoded subscription in Campaign Service. A channel rename is the most dangerous change — it requires both services to be updated in a single deployment window.

## Blast Radius Checklist

When you receive a `skill-contract-analytics-campaign` update — check these files in your repo:
- [ ] `infrastructure/acl/AnalyticsEventTranslator.java` — event parsing and `Campaign.pause()` call
- [ ] `infrastructure/messaging/AnalyticsEventListener.java` — channel subscription string
- [ ] `infrastructure/acl/BudgetExhaustedEvent.java` — event POJO field names and types
- [ ] `domain/model/Campaign.java` — `pause(PauseReason)` method signature
- [ ] `domain/service/BudgetTracker.java` — threshold comparison operator (the Sprint 3 bug lives here)
- [ ] `test/acl/AnalyticsEventTranslatorTest.java` — fixture JSON for `BudgetExhausted`; scenarios 8, 26, 27
- [ ] Any hardcoded comparison like `> 94` or `>= 95` in the codebase

## Common Mistakes

**Mistake 1 — Off-by-one on the threshold comparison.**
The known Sprint 3 bug: using `> 94` instead of `>= 95.0`. At exactly 95.0% burn, `> 94` is true (correct) but `> 94.9` would also be reached at 94.91% — the issue is that integer comparison truncates. Always compare `budgetBurnPercent >= 95.0` using floating-point comparison against the exact threshold. Scenario 26 (exactly 95.0%) must pass; 94.999% must not trigger.

**Mistake 2 — Not implementing the idempotency guard.**
The contract states Analytics emits `BudgetExhausted` exactly once per Campaign. But at-least-once delivery means Campaign Service could receive it twice. Without the `if (campaign.getStatus() == CampaignStatus.PAUSED) return;` guard, a duplicate event triggers `Campaign.pause()` again, which emits a second `CampaignPaused` and causes Eligibility Service to run its rule-removal logic twice. Implement the guard unconditionally.

**Mistake 3 — Reading `budgetBurnPercent` from the event and re-evaluating the threshold locally.**
Campaign Service should not re-check `budgetBurnPercent >= 95.0` before pausing. That check is Analytics Service's domain. Campaign Service's job is to pause the campaign when `BudgetExhausted` arrives — not to second-guess whether the threshold was actually crossed.

## Validation

| Scenario | Budget State | Expected |
|----------|-------------|----------|
| 8 | `camp-001`: `burnedAmount: 47500`, `totalAmount: 50000` (95.0%) | `BudgetExhausted` consumed, `Campaign.pause(BUDGET_EXHAUSTED)` called, `CampaignPaused` emitted |
| 26 | `camp-012`: `burnedAmount: 11400`, `totalAmount: 12000` (exactly 95.0%) | `BudgetExhausted` consumed, campaign paused |
| 27 | `camp-001`: subsequent redemption arrives after pause | `BudgetExhausted` NOT re-emitted by Analytics; idempotency guard skips duplicate if it somehow arrives |
| — | `camp-001`: `budgetBurnPercent: 94.998%` | `BudgetExhausted` NOT emitted — Campaign stays `ACTIVE` |
| — | `camp-002`: `budgetBurnPercent: 40.0%` | `BudgetExhausted` NOT emitted |
| Sprint 3 bug | `camp-012`: `budgetBurnPercent: 94.9%` | Campaign must remain `ACTIVE` — NOT paused at 94.9% |
