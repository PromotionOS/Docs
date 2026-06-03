# Team 2 — Eligibility Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, eligibility-service-guide, adr-recorder, pr-reviewer, campaign-eligibility-contract, catalog-eligibility-contract

---

## Context

Sprint 1 delivered core eligibility — segment matching, threshold evaluation, exclusion check, geo validation, and campaign status filter. Sprint 2 adds stack validation and the full ACL for all domain events the Eligibility Service consumes.

---

## Sprint 2 Goal

By end of sprint — stack validation works correctly for all stacking scenarios, and the ACL correctly handles all four incoming event types.

---

## Requirements

### Requirement 1 — Stack Validation

Implement `RuleEngine.validateStack()`:

1. When a customer qualifies for multiple campaigns in a single transaction — check if stacking is permitted
2. Stack limit is the MINIMUM `stackLimit` across ALL campaigns being applied
3. If number of eligible campaigns > stack limit → reject the excess campaigns
4. Rejection order: lower-value offers are rejected first (preserve highest discount for customer)
5. Return `STACK_EXCEEDED` for rejected campaigns
6. Default stack limit is 1 — no stacking unless at least one campaign has `stackPermission: true`

Stacking rules from test data:
- camp-002 (stackLimit:2) + camp-011 (stackLimit:2) → both eligible (min stack limit is 2)
- camp-002 + camp-011 + camp-003 (stackLimit:2) → camp-003 eligible too (still within limit of 2)
- camp-001 (stackLimit:1) + any other → second campaign rejected STACK_EXCEEDED

### Requirement 2 — Full ACL Implementation

Implement the complete ACL for all four event types:

**CampaignPublished → EligibilityRule:**
- Already partially implemented in Sprint 1
- Ensure ALL fields are translated including stackLimit and segmentRestriction

**CampaignPaused → remove rules:**
- Remove ALL EligibilityRule records for this campaignId from active engine
- Idempotent — calling twice must not error

**CatalogItemExcluded → update exclusions:**
- For each affected UPC — update exclusion status in all active EligibilityRules that include this category
- If category added to exclusions → add exclusion to all rules that scope this UPC
- If category removed from exclusions → remove exclusion from all rules

**SegmentUpdated → refresh segment cache:**
- Invalidate any cached customer profile for this customerId
- Next eligibility check fetches fresh profile from Catalog & Customer Service

### Requirement 3 — GET /offers endpoint

Implement `GET /offers?customerId=X&tenantId=Y&cartTotal=Z`:

1. Fetch all ACTIVE campaigns for tenant
2. Run eligibility check for each campaign against the customer
3. Return eligible offers ranked by discount value (highest first)
4. Return ineligible offers with reason
5. Response must match contract-eligibility-redemption.md schema exactly

---

## Domain Rules

From ubiquitous-language.md:
- Stack limit is the MINIMUM across all campaigns being applied
- Default stack limit is 1
- Lower-value offers are rejected first when stack limit is exceeded

---

## Contracts Involved

`contract-campaign-eligibility.md` — all four events
`contract-catalog-eligibility.md` — CatalogItemExcluded + SegmentUpdated
`contract-eligibility-redemption.md` — GET /offers response shape

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 15 | cust-011 + camp-002 + camp-011 (both stackLimit:2) | both eligible |
| 16 | cust-011 + camp-002 + camp-011 + camp-003 | all three eligible (stackLimit:2, 3 campaigns all stackable) |
| — | cust-001 + camp-001 (stackLimit:1) + camp-002 | camp-002 rejected STACK_EXCEEDED |
| 30 | cust-011 GET /offers | all eligible campaigns returned ranked by discount |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Stack limit resolution when campaigns have different limits
- Offer ranking algorithm — how you order eligible offers for GET /offers response
