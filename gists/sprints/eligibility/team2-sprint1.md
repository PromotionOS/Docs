# Team 2 — Eligibility Service — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, eligibility-service-guide, adr-recorder, pr-reviewer, campaign-eligibility-contract

---

## Context

The Eligibility Service determines whether a customer qualifies for a campaign's offer. The skeleton has the domain model and interfaces defined. Your job is to implement the two most fundamental checks — segment matching and campaign status filter.

These two together make the service useful. Everything else (exclusions, thresholds, stacking, geo) comes in Sprint 2.

---

## Sprint 1 Focus

**Segment matching + campaign status filter. Two requirements. Full SDD process.**

---

## Requirements

### Requirement 1 — Campaign Status Filter

Only ACTIVE campaigns are served:

1. Before any eligibility evaluation — check campaign status
2. PAUSED, DRAFT, SCHEDULED, EXPIRED → return `eligible: false, reason: CAMPAIGN_INACTIVE`
3. Only proceed with evaluation if status is ACTIVE

### Requirement 2 — Segment Matching

Implement `SegmentMatcher.match()`:

1. If campaign has no `segmentRestriction` → all customers pass
2. If campaign has `segmentRestriction: PLATINUM` → only PLATINUM customers qualify
3. Loyalty tier hierarchy: PLATINUM > GOLD > SILVER > BASIC
4. Customer's loyalty tier is fetched from Catalog & Customer Service `GET /customers/:id`
5. If customer tier is below restriction → return `eligible: false, reason: SEGMENT_MISMATCH`

### Requirement 3 — ACL for CampaignPublished

Implement `ACL.translate(CampaignPublished)`:

1. On receiving CampaignPublished event → translate into EligibilityRule
2. Store segmentRestriction and campaignId in EligibilityRule
3. See `contract-campaign-eligibility.md` for exact field mapping

---

## SDD Process — Do Not Skip

```
Read requirements
    ↓
Write Spec (sdd-spec-writer skill) — 20 min
    ↓
Review contract (campaign-eligibility-contract skill) — 10 min
    ↓
Record ADR (adr-recorder skill) — 10 min
    ↓
Implement — 45 min
    ↓
Validate against scenarios — 15 min
    ↓
PR (pr-reviewer skill) — 10 min
```

---

## Domain Rules

From ubiquitous-language.md:
- Only ACTIVE campaigns are served — never PAUSED, DRAFT, SCHEDULED, EXPIRED
- Loyalty tier hierarchy for segment restriction: PLATINUM > GOLD > SILVER > BASIC

---

## Contracts Involved

`contract-campaign-eligibility.md` — CampaignPublished event
Your ACL translates this into EligibilityRule. Read the exact field mappings before writing your spec.

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 2 | cust-002 (GOLD) + camp-004 (PLATINUM only) | eligible:false, SEGMENT_MISMATCH |
| 3 | cust-001 (PLATINUM) + camp-004 | eligible:true |
| 11 | any customer + camp-006 (PAUSED) | eligible:false, CAMPAIGN_INACTIVE |
| 12 | any customer + camp-007 (SCHEDULED) | eligible:false, CAMPAIGN_INACTIVE |

---

## Sprint 1 ADR Topics

Record an ADR for:
- How you fetch customer loyalty tier — call Catalog Service on every check vs cache
