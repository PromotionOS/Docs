---
name: contract-vendor-campaign
description: Contract skill for Campaign Service consuming from Vendor Service — ACL translation, field mapping, blast radius guide for the funding approval flow
metadata:
  type: reference
---

# Contract: Vendor → Campaign

## What This Contract Covers

Vendor Service drives the funding approval lifecycle for high-value Campaigns. Campaign Service consumes three domain events from Vendor Service and calls one REST endpoint on Vendor Service. Together these surfaces govern the `DRAFT → PENDING_APPROVAL → ACTIVE` (or back to `DRAFT`) lifecycle path required when `campaign.budget.totalAmount >= $50,000`.

## Events / APIs (Summary)

| Surface | Type | Channel / Endpoint | Purpose |
|---------|------|-------------------|---------|
| `FundingProposalSubmitted` | Redis event | `promotionos.vendor.proposal.submitted` | Record proposal ID on campaign |
| `FundingApproved` | Redis event | `promotionos.vendor.funding.approved` | Transition campaign `PENDING_APPROVAL → ACTIVE`, emit `CampaignPublished` |
| `FundingRejected` | Redis event | `promotionos.vendor.funding.rejected` | Revert campaign `PENDING_APPROVAL → DRAFT` |
| `POST /vendors/:id/proposals` | REST (outbound) | `/vendors/{vendorId}/proposals` | Initiate the approval flow at publish time |

## ACL Translation Guide

### FundingProposalSubmitted

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "uuid",
    "vendorId": "uuid",
    "vendorShare": "number",
    "krogerShare": "number",
    "submittedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/VendorEventTranslator.java

public void handleFundingProposalSubmitted(FundingProposalSubmittedEvent event) {
    String campaignId = event.getCampaignId();
    String tenantId = event.getTenantId();

    Campaign campaign = campaignRepository.findById(campaignId, tenantId)
        .orElseThrow(() -> new CampaignNotFoundException(campaignId));

    // Guard: only record the proposal if the campaign is still awaiting approval.
    if (campaign.getStatus() != CampaignStatus.PENDING_APPROVAL) {
        log.warn("FundingProposalSubmitted received for campaign={} with status={}; ignoring",
            campaignId, campaign.getStatus());
        return;
    }

    // Store proposalId for correlation with FundingApproved/FundingRejected.
    campaign.recordProposalId(event.getPayload().getProposalId());
    campaignRepository.save(campaign);
    // No status transition — campaign stays PENDING_APPROVAL.
}
```

**Fields to watch:**
- `payload.proposalId` → store as `Campaign.pendingProposalId`; this is the correlation key used to match `FundingApproved` and `FundingRejected` events
- `payload.vendorShare` / `payload.krogerShare` → do NOT update `Campaign.funding` from this event; wait for `FundingApproved` which carries the confirmed split
- No status transition fires on this event — campaign stays `PENDING_APPROVAL`

---

### FundingApproved

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "uuid",
    "vendorId": "uuid",
    "vendorShare": "number",
    "krogerShare": "number",
    "approvedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/VendorEventTranslator.java

public void handleFundingApproved(FundingApprovedEvent event) {
    String campaignId = event.getCampaignId();
    String tenantId = event.getTenantId();

    Campaign campaign = campaignRepository.findById(campaignId, tenantId)
        .orElseThrow(() -> new CampaignNotFoundException(campaignId));

    // Idempotency guard: if already ACTIVE, this is a duplicate — skip silently.
    if (campaign.getStatus() == CampaignStatus.ACTIVE) {
        log.warn("FundingApproved received for already-active campaign={}; skipping", campaignId);
        return;
    }

    var payload = event.getPayload();

    // Update funding split from the approved proposal.
    campaign.updateFunding(
        new Funding(
            payload.getVendorId(),
            Percentage.of(payload.getVendorShare()),
            Percentage.of(payload.getKrogerShare())
        )
    );

    // Transition: PENDING_APPROVAL → ACTIVE.
    // Campaign.publish() emits CampaignPublished to promotionos.campaign.published.
    campaign.publish();
    campaignRepository.save(campaign);
}
```

**Fields to watch:**
- `payload.proposalId` → optional: verify it matches `Campaign.pendingProposalId` for additional safety; if it does not match, log a warning — this indicates an event for a different proposal arrived
- `payload.vendorShare` + `payload.krogerShare` → update `Campaign.funding` from these values; the approved amounts may differ from what was originally submitted
- After `campaign.publish()` runs, `CampaignPublished` must fire on `promotionos.campaign.published` — Eligibility Service is waiting for it

---

### FundingRejected

**What arrives:**
```json
{
  "eventId": "uuid",
  "tenantId": "string",
  "campaignId": "uuid",
  "occurredAt": "ISO-8601 timestamp",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "uuid",
    "vendorId": "uuid",
    "rejectionReason": "string",
    "rejectedAt": "ISO-8601 timestamp"
  }
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/VendorEventTranslator.java

public void handleFundingRejected(FundingRejectedEvent event) {
    String campaignId = event.getCampaignId();
    String tenantId = event.getTenantId();

    Campaign campaign = campaignRepository.findById(campaignId, tenantId)
        .orElseThrow(() -> new CampaignNotFoundException(campaignId));

    if (campaign.getStatus() != CampaignStatus.PENDING_APPROVAL) {
        log.warn("FundingRejected received for campaign={} with status={}; ignoring",
            campaignId, campaign.getStatus());
        return;
    }

    var payload = event.getPayload();

    // Revert to DRAFT and store rejection reason for MX team visibility.
    campaign.rejectFunding(payload.getRejectionReason());
    campaignRepository.save(campaign);

    // Do NOT emit any event — campaign silently returns to DRAFT.
    log.info("Campaign reverted to DRAFT campaign={} reason={}",
        campaignId, payload.getRejectionReason());
}
```

**Fields to watch:**
- `payload.rejectionReason` → free-text string; store as a nullable field on Campaign so the MX team can see it on the campaign summary endpoint
- `payload.rejectedAt` → log for audit; no functional use in Campaign Service
- No event is emitted on rejection — `CampaignPublished` is not fired, Eligibility Service is not notified

---

### POST /vendors/:id/proposals (outbound REST call)

**What Campaign Service sends:**
```json
{
  "campaignId": "uuid",
  "tenantId": "string",
  "vendorShare": "number",
  "krogerShare": "number",
  "totalAmount": "number"
}
```

**How to build and call this in your ACL:**
```java
// infrastructure/acl/VendorServiceClient.java

public FundingProposalResponse submitProposal(Campaign campaign) {
    // Campaign must be in PENDING_APPROVAL before calling this endpoint.
    var request = FundingProposalRequest.builder()
        .campaignId(campaign.getId().toString())
        .tenantId(campaign.getTenantId())
        .vendorShare(campaign.getFunding().getVendorShare().getValue())
        .krogerShare(campaign.getFunding().getKrogerShare().getValue())
        .totalAmount(campaign.getBudget().getTotalAmount().getAmount())
        .build();

    try {
        var response = httpClient.post(
            "/vendors/" + campaign.getFunding().getVendorId() + "/proposals",
            request,
            FundingProposalResponse.class
        );
        return response; // contains proposalId, status: SUBMITTED
    } catch (HttpClientErrorException e) {
        return switch (e.getStatusCode().value()) {
            case 400 -> throw new InvalidProposalException(e.getResponseBodyAsString());
            case 404 -> throw new VendorNotFoundException(campaign.getFunding().getVendorId());
            case 409 -> throw new ProposalAlreadyExistsException(campaign.getId());
            default  -> throw new VendorServiceException(e);
        };
    }
}
```

**Fields to watch:**
- `vendorShare + krogerShare` must sum to `100`; Vendor Service returns `400 INVALID_PROPOSAL` if not
- `totalAmount` is the Campaign's budget `totalAmount`; the approval threshold check (`>= $50,000`) is a gate in `ApprovalGateway.requiresApproval()` before this call is made — do not re-check inside the client
- `409 PROPOSAL_ALREADY_EXISTS` means a proposal is already in `SUBMITTED` or `APPROVED` state; treat this as a duplicate call and do not transition the campaign status again

## What Changes When The Contract Changes

### If producer adds a new optional field to an event
Add the field to the event POJO. No action required if Campaign Service does not need to act on it. No `schemaVersion` bump needed.

### If producer renames a field
`schemaVersion` bumps from `1 → 2`. Update `VendorEventTranslator` to handle both versions during the transition window. Also notify Notification Service and Analytics Service — the contract doc notes that schema changes here may affect them.

### If the approval threshold changes in Vendor Service config
Campaign Service reads `ApprovalGateway.requiresApproval()` — the threshold is owned by Vendor Service's configuration (`approval-threshold-usd: 50000`). If Vendor Service changes this value, Campaign Service's gate logic must be updated to match. This is a configuration change, not a schema change.

### If the REST response adds a new field
Add the field to `FundingProposalResponse` POJO. Campaign Service currently only uses `proposalId` from this response — new fields are ignored unless the contract skill is updated to require them.

## Blast Radius Checklist

When you receive a `skill-contract-vendor-campaign` update — check these files in your repo:
- [ ] `infrastructure/acl/VendorEventTranslator.java` — all three event handlers
- [ ] `infrastructure/acl/VendorServiceClient.java` — REST call construction and error mapping
- [ ] `infrastructure/acl/FundingProposalRequest.java` — request POJO field names
- [ ] `infrastructure/acl/FundingProposalResponse.java` — response POJO field names
- [ ] `domain/model/Campaign.java` — `recordProposalId()`, `updateFunding()`, `publish()`, `rejectFunding()` method signatures
- [ ] `domain/model/CampaignStatus.java` — must include `PENDING_APPROVAL`
- [ ] `domain/service/ApprovalGateway.java` — threshold comparison
- [ ] `test/acl/VendorEventTranslatorTest.java` — fixture JSON for all three events; scenarios camp-013-01 through camp-013-05
- [ ] Any hardcoded channel name strings (`"promotionos.vendor.proposal.submitted"`, etc.)

## Common Mistakes

**Mistake 1 — Emitting `CampaignPublished` from `FundingProposalSubmitted` instead of from `FundingApproved`.**
`FundingProposalSubmitted` means the vendor has submitted a proposal — approval has not happened yet. `CampaignPublished` must only fire after `FundingApproved` is received and `campaign.publish()` succeeds. Emitting it from the wrong event loads Eligibility rules before funding is confirmed.

**Mistake 2 — Not implementing the idempotency guard on `FundingApproved`.**
Scenario camp-013-05: a duplicate `FundingApproved` arrives for an already-`ACTIVE` campaign. Without the `if (status == ACTIVE) return;` guard, `campaign.publish()` is called twice, emitting two `CampaignPublished` events. Eligibility Service will attempt to load rules twice, which may cause duplicates in the rule engine.

**Mistake 3 — Calling `POST /vendors/:id/proposals` before transitioning to `PENDING_APPROVAL`.**
The protocol is: transition to `PENDING_APPROVAL` first, then call the REST endpoint. If the REST call is made before the status transition and the call fails, the Campaign is still `DRAFT` and the MX team has no visibility that a proposal attempt was made. Transition first so the state is durable.

## Validation

| Scenario | Campaign | Action | Expected Result |
|----------|----------|--------|-----------------|
| camp-013-01 | `camp-013` (`totalAmount: $75,000`) | MX team calls publish | Campaign → `PENDING_APPROVAL`, `POST /vendors/:id/proposals` called, `FundingProposalSubmitted` received and `proposalId` stored |
| camp-013-02 | `camp-013` | Finance director approves | `FundingApproved` received, `funding` updated, Campaign → `ACTIVE`, `CampaignPublished` emitted |
| camp-013-03 | `camp-013` (reset to `PENDING_APPROVAL`) | Finance director rejects | `FundingRejected` received, Campaign → `DRAFT`, `rejectionReason` stored, no event emitted |
| camp-013-04 | `camp-013` (`totalAmount: $30,000`) | MX team calls publish | `ApprovalGateway` returns false, Campaign → `ACTIVE` directly, `CampaignPublished` emitted, no vendor call made |
| camp-013-05 | `camp-013` (already `ACTIVE`) | Duplicate `FundingApproved` arrives | Idempotency guard fires, `campaign.publish()` NOT called, no second `CampaignPublished` emitted |
