# Contract — Vendor ↔ Campaign

> Producer: Vendor Service (Team 7)
> Consumer: Campaign Service (Team 1)
> Type: Domain Events (Redis Pub/Sub) + REST API
> Status: LOCKED — changes require schemaVersion bump + contract skill update

---

## Overview

This contract governs two integration surfaces between Vendor Service and Campaign Service:

1. **Domain Events** — Vendor Service publishes three events that Campaign Service consumes to drive campaign status transitions.
2. **REST API** — Campaign Service calls `POST /vendors/:id/proposals` on Vendor Service to initiate the approval flow when an MX team member tries to publish a campaign with `totalAmount >= $50,000`.

### New Campaign Status

Vendor Service requires Campaign Service to add `PENDING_APPROVAL` to `CampaignStatus`. This must be in place before Sprint 2 begins.

| Status | Meaning |
|--------|---------|
| `PENDING_APPROVAL` | Campaign has been submitted for publication but is blocked pending vendor funding approval. No eligibility rules are loaded. `CampaignPublished` is NOT emitted until approval is received. |

---

## Domain Events

### 1. FundingProposalSubmitted

**Channel:** `promotionos.vendor.proposal.submitted`

Emitted by Vendor Service when a vendor submits a funding proposal for a campaign.
Campaign Service must confirm the campaign is in `PENDING_APPROVAL` and record the `proposalId`
for subsequent approve/reject correlation.

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

**Example:**

```json
{
  "eventId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "tenantId": "tenant-kroger-001",
  "campaignId": "camp-013",
  "occurredAt": "2025-02-14T10:30:00Z",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "p-9a1b2c3d-0001",
    "vendorId": "a1b2c3d4-0000-0000-0000-000000000001",
    "vendorShare": 60.0,
    "krogerShare": 40.0,
    "submittedAt": "2025-02-14T10:30:00Z"
  }
}
```

**ACL Translation (Campaign Service):**
- `payload.proposalId` → store on Campaign as `pendingProposalId` for correlation
- Confirm campaign status is `PENDING_APPROVAL` — if not, log warning and skip
- No status transition on this event — campaign remains `PENDING_APPROVAL`

---

### 2. FundingApproved

**Channel:** `promotionos.vendor.funding.approved`

Emitted by Vendor Service when a finance director approves a funding proposal.
Campaign Service must transition the campaign from `PENDING_APPROVAL` to `ACTIVE`
and emit `CampaignPublished` to unblock the rest of the platform.

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

**Example:**

```json
{
  "eventId": "a3f9c1e2-1234-5678-abcd-ef0123456789",
  "tenantId": "tenant-kroger-001",
  "campaignId": "camp-013",
  "occurredAt": "2025-02-15T09:00:00Z",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "p-9a1b2c3d-0001",
    "vendorId": "a1b2c3d4-0000-0000-0000-000000000001",
    "vendorShare": 60.0,
    "krogerShare": 40.0,
    "approvedAt": "2025-02-15T09:00:00Z"
  }
}
```

**ACL Translation (Campaign Service):**
- Look up campaign by `campaignId` + `tenantId`
- Assert status is `PENDING_APPROVAL` — if already `ACTIVE`, skip (idempotent guard)
- Update `funding.vendorShare` and `funding.krogerShare` from event payload
- Transition campaign status: `PENDING_APPROVAL → ACTIVE`
- Emit `CampaignPublished` event to `promotionos.campaign.published`
- Eligibility Service will load rules on receipt of `CampaignPublished`

---

### 3. FundingRejected

**Channel:** `promotionos.vendor.funding.rejected`

Emitted by Vendor Service when a finance director rejects a funding proposal.
Campaign Service must keep the campaign in `DRAFT` status (revert from `PENDING_APPROVAL`)
and record the rejection reason for MX team visibility.

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

**Example:**

```json
{
  "eventId": "b7d2e4f1-9abc-def0-1234-56789abcdef0",
  "tenantId": "tenant-kroger-001",
  "campaignId": "camp-013",
  "occurredAt": "2025-02-15T11:30:00Z",
  "schemaVersion": 1,
  "payload": {
    "proposalId": "p-9a1b2c3d-0001",
    "vendorId": "a1b2c3d4-0000-0000-0000-000000000001",
    "rejectionReason": "Vendor share exceeds category cap for snack foods",
    "rejectedAt": "2025-02-15T11:30:00Z"
  }
}
```

**ACL Translation (Campaign Service):**
- Look up campaign by `campaignId` + `tenantId`
- Assert status is `PENDING_APPROVAL`
- Revert status: `PENDING_APPROVAL → DRAFT`
- Store `rejectionReason` on campaign (new nullable field — non-breaking addition)
- Do NOT emit any event — campaign silently returns to draft for MX team to revise
- MX team sees rejection reason on campaign summary endpoint

---

## Redis Channel Names

| Event | Channel |
|-------|---------|
| FundingProposalSubmitted | `promotionos.vendor.proposal.submitted` |
| FundingApproved | `promotionos.vendor.funding.approved` |
| FundingRejected | `promotionos.vendor.funding.rejected` |

---

## REST API

### POST /vendors/:id/proposals

**Called by:** Campaign Service
**Implemented by:** Vendor Service
**Sprint:** 1 (both sides)

Campaign Service calls this endpoint when an MX team member attempts to publish a campaign
with `totalAmount >= $50,000`. Campaign Service must transition the campaign to `PENDING_APPROVAL`
before calling this endpoint.

**Request:**

```
POST /vendors/{vendorId}/proposals
Content-Type: application/json
```

```json
{
  "campaignId": "uuid",
  "tenantId": "string",
  "vendorShare": "number",
  "krogerShare": "number",
  "totalAmount": "number"
}
```

**Response 201 Created:**

```json
{
  "proposalId": "uuid",
  "campaignId": "uuid",
  "vendorId": "uuid",
  "tenantId": "string",
  "vendorShare": "number",
  "krogerShare": "number",
  "status": "SUBMITTED",
  "submittedAt": "ISO-8601 timestamp"
}
```

**Response 400 Bad Request** — vendorShare + krogerShare != 100, or vendor is INACTIVE/SUSPENDED:

```json
{
  "error": "INVALID_PROPOSAL",
  "message": "vendorShare and krogerShare must sum to 100"
}
```

**Response 404 Not Found** — vendor not found for tenantId:

```json
{
  "error": "VENDOR_NOT_FOUND",
  "message": "No vendor found for id={vendorId} and tenantId={tenantId}"
}
```

**Response 409 Conflict** — a proposal already exists for this campaign in SUBMITTED/APPROVED state:

```json
{
  "error": "PROPOSAL_ALREADY_EXISTS",
  "message": "A proposal for campaign={campaignId} is already in status=SUBMITTED"
}
```

---

## Approval Threshold

Campaigns with `totalAmount >= $50,000 USD` require vendor approval before publishing.
This threshold is configured in Vendor Service `application.yml`:

```yaml
promotionos:
  vendor:
    approval-threshold-usd: 50000
```

Campaign Service must check this gate before transitioning to `PENDING_APPROVAL`.
The reference implementation is `ApprovalGateway.requiresApproval(Money campaignTotal)`.

---

## Validation Scenarios

Teams must validate their implementations against these scenarios referencing camp-013:

| Scenario | Campaign | Action | Expected Result |
|----------|----------|--------|-----------------|
| camp-013-01 | camp-013 (totalAmount: $75,000) | MX team calls publish | Campaign transitions to PENDING_APPROVAL, POST /vendors/:id/proposals called, FundingProposalSubmitted emitted |
| camp-013-02 | camp-013 | Finance director approves | FundingApproved emitted, Campaign transitions PENDING_APPROVAL → ACTIVE, CampaignPublished emitted |
| camp-013-03 | camp-013 (reset to PENDING_APPROVAL) | Finance director rejects | FundingRejected emitted, Campaign reverts to DRAFT, rejectionReason stored |
| camp-013-04 | camp-013 (totalAmount: $30,000) | MX team calls publish | No approval required, Campaign transitions DRAFT → ACTIVE directly, CampaignPublished emitted |
| camp-013-05 | camp-013 | Duplicate FundingApproved received | Campaign Service skips (idempotent guard), no second CampaignPublished emitted |

---

## Breaking Change Protocol

If either team needs to change any field in these event schemas or the REST request/response:

1. Bump `schemaVersion` in the event (`1 → 2`)
2. Update this contract file with what changed and why
3. The consuming team uses the updated contract to identify ACL blast radius
4. Both services must support old + new schema version during the transition window
5. Notify facilitator — schema changes affect Notification Service and Analytics Service too

**Fields that are safe to add (non-breaking):**
- New optional fields with null defaults on events
- New optional query params on REST endpoints
- New optional fields in REST response bodies

**Fields that are breaking changes:**
- Renaming any existing field
- Changing a field's type (e.g. string → number)
- Removing any field
- Changing enum values (e.g. renaming `APPROVED` to `ACCEPTED`)
- Changing the channel name for an existing event
