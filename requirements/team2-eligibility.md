# Team 2 — Eligibility Service
## Type: Build via SDD

---

## Context

The Eligibility Service determines whether a customer qualifies for a campaign's offer. It is the most complex bounded context in the system — it enforces stacking rules, exclusions, thresholds, geo-restrictions, and segment matching.

The domain model and interfaces are defined. Your job is to implement the business logic.

**Your job:**
1. Read the domain model using the eligibility-service-guide skill
2. Read the contracts from campaign-eligibility-contract and catalog-eligibility-contract skills
3. Write a spec using the sdd-spec-writer skill
4. Implement the RuleEngine and SegmentMatcher
5. Record an ADR using the adr-recorder skill
6. Validate against all eligibility scenarios
7. Open a PR

---

## Business Rules You Must Implement

### Segment Matching
1. A customer's `loyaltyTier` must meet or exceed `segmentRestriction` if set
   - Tier hierarchy: PLATINUM > GOLD > SILVER > BASIC
   - If campaign has `segmentRestriction: PLATINUM` only PLATINUM customers qualify
2. A customer can belong to multiple segments — any match qualifies

### Threshold Evaluation
3. If campaign has a `SPEND` threshold — customer's cart total must meet or exceed the threshold amount
4. Cart total is evaluated **after exclusions are removed** from the cart
5. If campaign has no threshold — all customers pass threshold check

### Exclusion Check
6. If a UPC's category is excluded — that UPC is excluded from the campaign
7. Exclusion is inherited — if parent category `cat-alcohol` is excluded, all child categories and their UPCs are excluded
8. Excluded UPCs are removed from cart before threshold evaluation

### Stack Validation
9. A customer cannot redeem more offers in a single transaction than the lowest `stackLimit` across all campaigns being applied
10. Default stack limit is 1 — customers can only use one offer unless explicitly permitted

### Geo Validation
11. Customer's `division` must be in the campaign's `geoScope`
12. If geoScope contains `division-all` — all customers qualify regardless of division

### Campaign Status
13. Only `ACTIVE` campaigns are eligible — never serve offers from PAUSED, DRAFT, SCHEDULED, or EXPIRED campaigns

---

## Validation Scenarios to Pass

| Scenario | Input | Expected |
|----------|-------|----------|
| 1 | cust-002 (GOLD) + camp-002 (no restriction) | eligible: true |
| 2 | cust-002 (GOLD) + camp-004 (PLATINUM only) | eligible: false, SEGMENT_MISMATCH |
| 3 | cust-001 (PLATINUM) + camp-004 (PLATINUM only) | eligible: true |
| 4 | cust-001 + camp-001 + upc-wine-cab in cart | eligible: false, UPC_EXCLUDED |
| 5 | cust-003 + camp-003 + cartTotal: 35.00 | eligible: false, THRESHOLD_NOT_MET |
| 6 | cust-003 + camp-003 + cartTotal: 67.50 | eligible: true, discount: 10.00 |

---

## API Contract You Must Honour

**POST /eligibility/check**
```json
Request:
{
  "tenantId": "string",
  "customerId": "string",
  "campaignId": "string",
  "cartTotal": "number",
  "cartUPCs": ["string"]
}

Response (eligible):
{
  "eligible": true,
  "campaignId": "string",
  "discount": "number",
  "discountType": "PCT_OFF | AMT_OFF | BOGO | THRESHOLD"
}

Response (ineligible):
{
  "eligible": false,
  "campaignId": "string",
  "reason": "SEGMENT_MISMATCH | THRESHOLD_NOT_MET | UPC_EXCLUDED | GEO_MISMATCH | STACK_EXCEEDED | CAMPAIGN_INACTIVE"
}
```

---

## Events You Must Consume

Your Anti-Corruption Layer must handle:
- `CampaignPublished` → load eligibility rules into your rule engine
- `CampaignPaused` → remove campaign rules, stop serving
- `CatalogItemExcluded` → update exclusion list
- `SegmentUpdated` → refresh segment cache

Use the `campaign-eligibility-contract` skill to understand the exact event schemas.

---

## Skills to Use

- `sdd-spec-writer` — write your spec before touching code
- `ddd-guide` — understand how to implement RuleEngine as a domain service
- `eligibility-service-guide` — understand the skeleton
- `adr-recorder` — record key decisions (especially stacking and exclusion inheritance)
- `campaign-eligibility-contract` — understand what Campaign Service sends you
- `catalog-eligibility-contract` — understand what Catalog Service sends you
