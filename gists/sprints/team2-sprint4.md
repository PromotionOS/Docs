# Team 2 — Eligibility Service — Sprint 4

> Type: Full Validation + Integration
> Duration: 2 hours
> Skills: sdd-spec-writer, eligibility-service-guide, pr-reviewer

---

## Context

Sprints 1-3 built the full eligibility engine. Sprint 4 is integration and validation — confirm all scenarios pass end-to-end with real data flowing from Campaign Service and Catalog Service.

---

## Sprint 4 Goal

All 30 eligibility-related validation scenarios pass with live services. Service is demo-ready.

---

## Requirements

### Requirement 1 — Integration Validation

With all services deployed — run the full validation suite end-to-end:

1. Start real campaigns via Campaign Service → confirm CampaignPublished arrives and rules load
2. Run all eligibility checks against live customer data from Catalog Service
3. Confirm exclusion inheritance works with live catalog data
4. Confirm segment matching works with live customer profiles

### Requirement 2 — Offer Ranking

Finalize `GET /offers` ranking:

1. Eligible offers ranked by `discountApplied` descending (highest value first)
2. Ties broken by campaign name alphabetically
3. Ineligible offers listed after eligible offers
4. CAMPAIGN_INACTIVE offers are filtered out entirely (not shown at all)

### Requirement 3 — Performance

Ensure eligibility checks complete in < 500ms:

1. Cache customer profiles for 60 seconds (invalidate on SegmentUpdated)
2. Load eligibility rules into memory on CampaignPublished — do not query DB per check
3. If performance target is not met — document the bottleneck in an ADR

### Requirement 4 — Full Scenario Pass

| Scenario | Expected |
|----------|----------|
| 1-6 | Basic eligibility |
| 11, 12 | Status filter |
| 13, 14 | Geo validation |
| 15, 16 | Stack validation |
| 21, 22 | Tier boundaries |
| 23-25 | Catalog exclusions |
| 30 | Multi-segment |

---

## Sprint 4 ADR Topics

Record an ADR for:
- Customer profile caching — TTL choice and invalidation strategy
