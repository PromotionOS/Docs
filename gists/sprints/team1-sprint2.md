# Team 1 — Campaign Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, campaign-service-guide, adr-recorder, pr-reviewer, campaign-eligibility-contract

---

## Context

Sprint 1 fixed the budget threshold bug. The Campaign Service now correctly auto-pauses at 95.0%. Sprint 2 adds two new capabilities: conflict resolution improvements and the campaign scheduling lifecycle — allowing campaigns to transition from SCHEDULED to ACTIVE automatically when their start date arrives.

---

## Sprint 2 Goal

By end of sprint — campaign conflict resolution is robust across all edge cases, and SCHEDULED campaigns automatically transition to ACTIVE on their start date.

---

## Requirements

### Requirement 1 — Improved Conflict Resolution

Enhance `ConflictResolver.checkUPCOverlap()`:

1. A campaign cannot be published if any UPC in its scope overlaps with an ACTIVE campaign on the same tenant
2. Overlap check must also consider SCHEDULED campaigns — a campaign scheduled for tomorrow conflicts with one publishing today on the same UPC
3. Date range overlap must be checked — two campaigns on the same UPC but non-overlapping date ranges do NOT conflict
4. Return `409 UPC_OVERLAP` with the conflicting campaignId and the conflicting UPC codes

Date range overlap rule:
- Campaign A: June 1 - June 15
- Campaign B: June 10 - June 30
- These overlap (June 10-15) → conflict if same UPC

Non-overlapping:
- Campaign A: June 1 - June 15
- Campaign B: June 16 - June 30
- No overlap → no conflict

### Requirement 2 — SCHEDULED to ACTIVE Transition

Implement scheduled campaign activation:

1. A campaign with status SCHEDULED and `dateRange.startDate = today` must transition to ACTIVE
2. On transition — emit `CampaignPublished` event (same event as manual publish)
3. Transition happens at midnight on the startDate (use a scheduled job — simple polling is acceptable for the session)
4. If budget is zero or funding is missing at transition time — log error and do NOT activate

### Requirement 3 — Campaign Expiry

Implement campaign expiry:

1. A campaign with status ACTIVE and `dateRange.endDate < today` must transition to EXPIRED
2. On transition — emit `CampaignPaused` event with reason `CAMPAIGN_EXPIRED`
3. Expired campaigns must not serve offers (Eligibility will consume the CampaignPaused event)

---

## Domain Rules

From ubiquitous-language.md:
- Campaign lifecycle: DRAFT → ACTIVE → PAUSED | EXPIRED | SCHEDULED → ACTIVE
- A Campaign cannot be published if its UPCs overlap with another active/scheduled campaign
- Date range overlap determines UPC conflict

---

## Contracts Involved

`contract-campaign-eligibility.md` — CampaignPublished event
- SCHEDULED → ACTIVE transition must publish the same CampaignPublished event shape
- `schemaVersion: 1` unchanged
- No contract changes needed

---

## Validation Scenarios

| Scenario | Input | Expected |
|----------|-------|----------|
| 10 | camp-009 publish (UPC overlap with camp-001 on upc-cola-12pk) | 409 UPC_OVERLAP |
| — | camp-007 startDate = today | transitions to ACTIVE, CampaignPublished emitted |
| — | ACTIVE campaign endDate = yesterday | transitions to EXPIRED, CampaignPaused emitted |
| — | two campaigns same UPC non-overlapping dates | no conflict, both can publish |

---

## Sprint 2 ADR Topics

Record an ADR for:
- Date range overlap algorithm chosen
- Scheduled job approach for SCHEDULED → ACTIVE transition (polling vs cron vs event-driven)
