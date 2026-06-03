# Contract — Analytics → Campaign

> Producer: Analytics Service (Team 5 — pre-built)
> Consumer: Campaign Service (Team 1 — pre-built with bug)
> Type: Domain Events (Redis Pub/Sub)
> Status: LOCKED — both services are pre-built against this contract

---

## Events Published by Analytics Service

### BudgetExhausted

Emitted by Analytics Service when a Campaign's budget burn reaches or exceeds 95.0%.
Campaign Service consumes this event and triggers `Campaign.pause()`.

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

**Campaign Service ACL Translation:**
- `campaignId` → look up Campaign aggregate
- `tenantId` → tenant isolation
- Call `Campaign.pause(reason: BUDGET_EXHAUSTED)`
- Emit `CampaignPaused` event downstream

**Critical rules:**
- `BudgetExhausted` is emitted **exactly once** per Campaign — not on every subsequent redemption after 95%
- Analytics Service tracks a `budgetExhaustedEmitted: boolean` flag per Campaign
- Campaign Service must be idempotent — duplicate `BudgetExhausted` events must not cause double-pause

---

## Redis Channel Names

| Event | Channel |
|-------|---------|
| BudgetExhausted | `promotionos.analytics.budget.exhausted` |

---

## The Auto-Pause Flow

```
OfferRedeemed (Redemption → Analytics)
    ↓
Analytics: budgetBurnPercent >= 95.0
    ↓
Analytics: emit BudgetExhausted (once)
    ↓
Campaign: consume BudgetExhausted
    ↓
Campaign: Campaign.pause(BUDGET_EXHAUSTED)
    ↓
Campaign: emit CampaignPaused
    ↓
Eligibility: consume CampaignPaused → remove rules
Frontend: consume CampaignPaused → show alert
```

---

## The Bug (Team 1 Sprint 3)

Campaign Service has a bug in how it handles `BudgetExhausted`:

**Bug location:** `BudgetTracker.checkThreshold()`
**Symptom:** Campaign pauses at 94.9% burn instead of 95.0%
**Root cause:** Off-by-one — uses `> 94` instead of `>= 95.0`

Team 1's Sprint 3 job is to find this bug, write an RCA, fix it, and validate that:
- Scenario 26 (camp-012 at exactly 95.0%) → CampaignPaused fires
- Scenario 8 (camp-001 at 95.2%) → CampaignPaused fires
- At 94.9% → Campaign stays ACTIVE

---

## Breaking Change Protocol

Both services are pre-built. Any schema change to `BudgetExhausted`:
1. Both Team 1 and Team 5 must agree
2. Bump `schemaVersion` to 2
3. Update `skill-contract-analytics-campaign.md`
4. Both services must be updated and deployed together

**Safe to add:** New optional payload fields
**Breaking:** Any field rename, type change, removal

---

## Validation

| Scenario | Budget State | Expected |
|----------|-------------|----------|
| 8 | camp-001: burnedAmount 47500 (95.0%) | BudgetExhausted emitted, CampaignPaused follows |
| 26 | camp-012: burnedAmount 11400 (95.0% exactly) | BudgetExhausted emitted |
| 27 | camp-001: subsequent redemption after pause | BudgetExhausted NOT re-emitted |
| — | camp-001: burnedAmount 47499 (94.998%) | BudgetExhausted NOT emitted, Campaign stays ACTIVE |
| — | camp-002: burnedAmount 12000 (40.0%) | BudgetExhausted NOT emitted |
