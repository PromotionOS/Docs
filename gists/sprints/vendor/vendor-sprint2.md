# Team 7 — Vendor Service — Sprint 2

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, vendor-service-guide, adr-recorder, pr-reviewer, campaign-vendor-contract, vendor-notification-contract

---

## Context

Sprint 1 delivered Vendor profile CRUD, Funding Proposal submission, and a Vendor's Claim history view via ACL from ClaimSubmitted events. Funding Proposals sit in SUBMITTED status indefinitely — there is no approval workflow yet.

This sprint introduces the Approval Workflow: an authorised MX role approves or rejects a Funding Proposal, which publishes FundingApproved or FundingRejected events. The Campaign Service (Team 1) consumes FundingApproved to unblock camp-013, enabling the DRAFT → PENDING_APPROVAL → ACTIVE Campaign lifecycle for high-budget campaigns.

---

## Sprint 2 Goal

Implement the Approval Workflow — publish FundingApproved and FundingRejected events so the Campaign Service can unblock camp-013 and advance through the PENDING_APPROVAL → ACTIVE transition.

---

## Requirements

### Requirement 1 — Approval Entity

1. Define the `approvals` table: `approval_id` (PK), `tenant_id`, `proposal_id` (FK → funding_proposals), `approved_by` (userId string), `decision` (`APPROVED` | `REJECTED`), `rejection_reason` (nullable), `decided_at`
2. `Approval` is an entity inside the Vendor Service — it records the decision made by an authorised MX user
3. One FundingProposal can have at most one Approval — enforce as a unique constraint on `proposal_id` in the `approvals` table
4. An Approval cannot be reversed once recorded — immutable after creation

### Requirement 2 — Approve Funding Proposal

1. `POST /funding-proposals/:proposalId/approve` — body: `{ approvedBy: string }`
2. Validate: proposal must be in `SUBMITTED` status — return 422 if already APPROVED, REJECTED, or DRAFT
3. Validate: `approvedBy` must be a non-empty string (role enforcement is out of scope this sprint — record ADR)
4. Transition proposal `status` → `APPROVED`
5. Create an `Approval` record with `decision: APPROVED`
6. Publish `FundingApproved` domain event: `{ tenantId, vendorId, campaignId, proposalId, approvedAmount: totalAmount, vendorShare, krogerShare, approvedAt, schemaVersion: 1 }`
7. Return 200 with the updated FundingProposal

### Requirement 3 — Reject Funding Proposal

1. `POST /funding-proposals/:proposalId/reject` — body: `{ approvedBy: string, rejectionReason: string }`
2. Validate: proposal must be in `SUBMITTED` status
3. Validate: `rejectionReason` is required and non-empty
4. Transition proposal `status` → `REJECTED`
5. Create an `Approval` record with `decision: REJECTED`, `rejection_reason` set
6. Publish `FundingRejected` domain event: `{ tenantId, vendorId, campaignId, proposalId, rejectionReason, rejectedAt, schemaVersion: 1 }`
7. Return 200 with the updated FundingProposal

### Requirement 4 — Campaign Service Consumes FundingApproved

This requirement is a cross-team integration task. Team 7 (Vendor Service) publishes the event; Team 1 (Campaign Service) must consume it. Coordinate with Team 1 on:

1. Confirm the `funding.approved` topic name matches what Campaign Service subscribes to
2. Confirm FundingApproved payload carries `campaignId` — Campaign Service uses this to look up and unblock the campaign
3. Campaign Service transitions camp-013: `PENDING_APPROVAL → ACTIVE` on FundingApproved receipt
4. Vendor Service does not call Campaign Service directly — the integration is event-driven only
5. Document the agreed event contract in `contract-campaign-vendor.md` before implementing

### Requirement 5 — Campaign Lifecycle Update (Vendor Service perspective)

1. `GET /funding-proposals/:proposalId` — return full proposal including `status` and linked `Approval` (if any)
2. `GET /vendors/:vendorId/funding-proposals` — update to include `status` in each row
3. Vendor Service must reflect the PENDING_APPROVAL state for campaigns above the Approval Threshold in its own records — add `campaign_status` column to `vendor_claim_events` or document why not (ADR)

---

## Domain Rules

From ubiquitous-language.md:
- Funding Proposal lifecycle: DRAFT → SUBMITTED → APPROVED | REJECTED — transitions are one-way and immutable once terminal
- Approval Threshold: `campaign.budget.totalAmount >= $50,000` → Campaign requires Approval before publication; camp-013 is the reference scenario
- Campaign lifecycle with Approval Threshold: `DRAFT → PENDING_APPROVAL → ACTIVE` — Vendor Service does not own Campaign status but must publish the event that drives this transition
- Approval is an entity in Vendor Service — not called Sign-off or Authorisation
- Domain Events are past tense and immutable: FundingApproved, FundingRejected — `schemaVersion` must be set
- `vendorShare + krogerShare = 100%` invariant must still hold in the FundingApproved event payload

---

## Contracts Involved

- `contract-campaign-vendor.md` — FundingApproved event published by Vendor Service, consumed by Campaign Service; align on topic name and all fields before coding
- `contract-vendor-notification.md` — FundingApproved and FundingRejected also consumed by Notification Service (Team 8); ensure `vendorContactEmail` is in the payload
- Vendor Service now publishes events for the first time — this is a new outbound contract

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 36 | POST /funding-proposals/:id/approve for a SUBMITTED proposal | status → APPROVED; FundingApproved event published |
| 37 | POST /funding-proposals/:id/reject with rejectionReason | status → REJECTED; FundingRejected event published with reason |
| 38 | POST /funding-proposals/:id/approve for an already-APPROVED proposal | 422 response |
| 39 | Campaign Service receives FundingApproved for camp-013 | camp-013 transitions PENDING_APPROVAL → ACTIVE |
| 40 | POST /funding-proposals/:id/reject with empty rejectionReason | 422 validation error |

---

## Sprint 2 ADR Topics

Record an ADR for:
- MX role enforcement for Approve/Reject — why `approvedBy` is a plain string in Sprint 2, what the production enforcement mechanism should be (JWT claims, RBAC middleware), and when to add it
- Event publishing strategy — transactional outbox vs direct publish: risk of publishing the event before the database transaction commits; what the failure mode is
- Cross-team contract coordination — how Vendor Service and Campaign Service agreed on the FundingApproved event shape without a shared schema registry
