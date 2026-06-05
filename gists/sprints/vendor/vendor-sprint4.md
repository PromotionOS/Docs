# Team 7 — Vendor Service — Sprint 4

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, vendor-service-guide, adr-recorder, pr-reviewer, campaign-vendor-contract, vendor-notification-contract, analytics-vendor-contract

---

## Context

Sprints 1–3 delivered: Vendor profile CRUD, Funding Proposal submission, Approval Workflow (FundingApproved/FundingRejected events published), Claim history via ACL, Predicted Lift forecasting from category benchmarks, Burn Velocity from Analytics Service, and the pre-publish health check.

This final sprint assembles the complete camp-013 approval flow end-to-end and builds the Vendor Portal view — a single API response giving a Vendor full visibility into their funded campaigns, Claims, and Deductions.

---

## Sprint 4 Goal

Demonstrate the camp-013 complete approval flow end-to-end and deliver the Vendor Portal view showing funded campaigns, Claims, and Deductions.

---

## Requirements

### Requirement 1 — camp-013 End-to-End Approval Flow

Wire and verify the complete flow for camp-013 (the reference campaign requiring Approval):

1. camp-013 has `totalAmount >= $50,000` → Campaign Service places it in `PENDING_APPROVAL`
2. Vendor submits a FundingProposal for camp-013: `POST /vendors/:vendorId/funding-proposals` with `campaignId: camp-013`
3. MX role approves: `POST /funding-proposals/:proposalId/approve` → `status: APPROVED` → `FundingApproved` event published
4. Campaign Service receives `FundingApproved` → camp-013 transitions `PENDING_APPROVAL → ACTIVE`
5. Notification Service receives `FundingApproved` → EMAIL to MX_TEAM and VENDOR_CONTACT; WebSocket push to MX_TEAM
6. Document the complete sequence diagram in the ADR — every service involved, every event, every state transition

### Requirement 2 — Vendor Portal View

1. `GET /vendors/:vendorId/portal` — returns a comprehensive view for the Vendor's portal dashboard
2. Response shape:
   ```json
   {
     "vendorId": "...",
     "name": "...",
     "status": "ACTIVE",
     "fundingProposals": [...],
     "activeCampaigns": [...],
     "claims": [...],
     "totalDeductions": { "amount": 0.00, "currency": "USD" }
   }
   ```
3. `fundingProposals`: all proposals with `status`, `vendorShare`, `krogerShare`, `totalAmount`
4. `activeCampaigns`: call Campaign Service `GET /campaigns/:id/summary` for each `campaignId` where the FundingProposal is `APPROVED`; include `status`, `budgetBurned`, `budgetTotal`
5. `claims`: from the local `vendor_claim_events` ACL; include `claimId`, `campaignId`, `amount`, `deduction`, `submittedAt`
6. `totalDeductions`: sum of all `deduction` values across all Claims for this Vendor
7. If Campaign Service is unavailable — return campaign entries with `campaignSummary: null`; do not fail the portal response

### Requirement 3 — Deduction Summary

1. `GET /vendors/:vendorId/deductions` — returns a breakdown of Deductions per Campaign
2. Response: `{ vendorId, deductions: [{ campaignId, claimCount: int, totalDeduction: Money }], grandTotal: Money }`
3. Deduction formula enforced in the read model: `deduction = discountApplied × vendorShare%` — if a stored `deduction` value is zero but `amount` and `vendorShare` are non-zero, recompute it (log a warning)
4. Supports `fromDate` and `toDate` query params to filter by `submittedAt`
5. This endpoint satisfies the Vendor's reconciliation use case — they can match it against their accounts payable system

### Requirement 4 — Integration Smoke Test

Document (in the ADR) the manual integration test sequence for the full demo:

1. Create Vendor: `POST /vendors`
2. Submit FundingProposal for camp-013: `POST /vendors/:vendorId/funding-proposals`
3. Run health check: `GET /vendors/:vendorId/health-check?campaignId=camp-013` → `readyToPublish: false` (proposal not yet approved)
4. Approve: `POST /funding-proposals/:proposalId/approve` → FundingApproved published
5. Confirm camp-013 is ACTIVE (via Campaign Service)
6. Confirm Notification Service delivered EMAIL and WebSocket
7. Simulate a Redemption → ClaimSubmitted event → Vendor portal shows Claim and Deduction
8. `GET /vendors/:vendorId/portal` → shows ACTIVE campaign, Claim, total Deduction

---

## Domain Rules

From ubiquitous-language.md:
- Approval Threshold: `totalAmount >= $50,000` → Campaign requires Approval; camp-013 is the canonical example
- Campaign lifecycle with threshold: `DRAFT → PENDING_APPROVAL → ACTIVE` — Vendor Service drives the PENDING_APPROVAL → ACTIVE transition via FundingApproved event
- Deduction: `deduction = discountApplied × vendorShare%` — must be shown in the Vendor Portal and Deduction Summary
- Claim status: `PENDING → SUBMITTED` — Vendor Service ACL consumes ClaimSubmitted; it does not own the Claim aggregate
- Funding invariant: `vendorShare + krogerShare = 100%` — must hold in all portal and summary responses
- Vendor status `ACTIVE` is required for `readyToPublish: true` on the health check
- Tenant isolation: portal data must be filtered by `tenantId` at every query

---

## Contracts Involved

- `contract-campaign-vendor.md` — camp-013 FundingApproved flow; confirm Campaign Service transition is triggered by the event, not a direct API call
- `contract-vendor-notification.md` — final review: FundingApproved carries `vendorContactEmail` so Notification Service can deliver without a separate lookup
- `contract-claim-vendor.md` — ClaimSubmitted ACL: confirm `deduction` field is present and computed correctly upstream
- Vendor Portal API (`GET /vendors/:vendorId/portal`) is a new outbound API contract — document its shape for any frontend consumer

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 32 | Full camp-013 flow: submit proposal → approve → FundingApproved published | camp-013 transitions to ACTIVE in Campaign Service |
| 33 | GET /vendors/:vendorId/health-check before approval | readyToPublish: false, fundingProposalStatus: SUBMITTED |
| 34 | GET /vendors/:vendorId/health-check after approval | readyToPublish: true, fundingProposalStatus: APPROVED |
| 35 | GET /vendors/:vendorId/portal after one Redemption → Claim | Portal shows claim with deduction; totalDeductions is non-zero |
| 36 | GET /vendors/:vendorId/deductions with fromDate/toDate range | Returns only claims submitted in that window |
| 37 | Campaign Service unavailable during portal request | Portal returns with campaignSummary: null; no 500 error |

---

## Sprint 4 ADR Topics

Record an ADR for:
- camp-013 sequence diagram — full event chain from DRAFT to ACTIVE including every bounded context involved; decisions about where failures are caught and retried
- Vendor Portal data aggregation strategy — why Vendor Service aggregates on-demand vs a pre-materialised view; when the materialised approach becomes necessary at scale
- Deduction recomputation guard — why the portal recomputes deduction when stored value is zero, and whether this indicates an upstream data quality issue that should be surfaced as an alert
