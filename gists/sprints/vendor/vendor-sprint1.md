# Team 7 — Vendor Service — Sprint 1

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, vendor-service-guide, adr-recorder, pr-reviewer, campaign-vendor-contract

---

## Context

The Vendor Service is a new Java/Spring Boot service being built from scratch in Cohort 1. No skeleton exists — the team bootstraps the project this sprint.

The domain problem: Vendors are currently just a `vendorId` string on the Campaign aggregate. They have no service of their own, cannot submit Funding Proposals, and cannot see what campaigns they are funding or what Claims have been filed against them. This sprint establishes the Vendor as a first-class domain entity and delivers the two features Vendors need most: managing their profile and submitting a Funding Proposal.

---

## Sprint 1 Goal

Bootstrap the Vendor Service with Vendor profile CRUD, Funding Proposal submission, and a Vendor view of their active Campaigns and Claim history by consuming ClaimSubmitted events.

---

## Requirements

### Requirement 1 — Project Bootstrap

1. Initialise a Spring Boot 3.x project: `vendor-service` with Java 21
2. Connect to PostgreSQL (Supabase) — connection string from environment variable `DATABASE_URL`
3. Use Spring Data JPA with Flyway for schema migrations
4. Define the `vendors` table: `vendor_id` (PK), `tenant_id`, `name`, `contact_email`, `status` (`ACTIVE` | `INACTIVE` | `SUSPENDED`), `created_at`
5. Define the `funding_proposals` table: `proposal_id` (PK), `tenant_id`, `vendor_id`, `campaign_id`, `vendor_share` (decimal), `kroger_share` (decimal), `total_amount` (decimal), `status` (`DRAFT` | `SUBMITTED` | `APPROVED` | `REJECTED`), `created_at`, `updated_at`
6. Expose a health endpoint: `GET /health` → `{ "status": "ok" }`

### Requirement 2 — Vendor Profile CRUD

1. `POST /vendors` — create a Vendor: `{ name, contactEmail, tenantId }` → returns created Vendor with generated `vendorId`
2. `GET /vendors/:vendorId` — fetch Vendor by ID; return 404 if not found
3. `PUT /vendors/:vendorId` — update `name` and `contactEmail` only; `status` is immutable via this endpoint
4. `DELETE /vendors/:vendorId` — soft delete: set `status: INACTIVE`, do not remove the row
5. Vendor `status` values: `ACTIVE`, `INACTIVE`, `SUSPENDED` — only `ACTIVE` Vendors may submit Funding Proposals
6. Tenant isolation: all queries must filter by `tenantId`; a Vendor never sees another Tenant's data

### Requirement 3 — Submit Funding Proposal

1. `POST /vendors/:vendorId/funding-proposals` — body: `{ campaignId, vendorShare, totalAmount }`
2. Validate: `vendorShare` must be between 0 and 100 (exclusive of both extremes)
3. Derive: `krogerShare = 100 - vendorShare` — do not accept `krogerShare` as input
4. Validate: `vendorShare + krogerShare = 100` — domain invariant, enforce in the domain model
5. Validate: only `ACTIVE` Vendors may submit; return 422 with reason if Vendor is INACTIVE or SUSPENDED
6. Set initial `status: SUBMITTED` — there is no DRAFT → SUBMITTED transition in this sprint (record ADR)
7. Return created FundingProposal with `proposalId`, `status`, `vendorShare`, `krogerShare`, `totalAmount`
8. `GET /vendors/:vendorId/funding-proposals` — list all proposals for a Vendor, ordered by `created_at` desc

### Requirement 4 — Vendor Campaign and Claim View

1. Subscribe to the `claim.submitted` topic — published by Claim Processing Service
2. On receiving `ClaimSubmitted`: extract `tenantId`, `vendorId`, `claimId`, `campaignId`, `amount`, `deduction`, `submittedAt`
3. Persist a local `vendor_claim_events` table: `claim_id`, `vendor_id`, `campaign_id`, `amount`, `deduction`, `submitted_at` — this is the ACL's local read model
4. `GET /vendors/:vendorId/claims` — returns the Vendor's Claim history from the local read model; supports `campaignId` filter
5. `GET /vendors/:vendorId/campaigns` — calls Campaign Service `GET /campaigns/:id/summary` for each `campaignId` seen in the Vendor's Claims; aggregates and returns list
6. If Campaign Service is unavailable during the campaigns view — return the Claim data with `campaignSummary: null` rather than failing the request

---

## Domain Rules

From ubiquitous-language.md:
- Vendor is a first-class aggregate root — `vendorId`, `name`, `contactEmail`, `status (ACTIVE | INACTIVE | SUSPENDED)`
- Funding Proposal lifecycle: DRAFT → SUBMITTED → APPROVED | REJECTED — this sprint creates at SUBMITTED directly (record ADR)
- Funding invariant: `vendorShare + krogerShare = 100%` always — enforce in domain model, not just validation
- Claim: `deduction = discountApplied × vendorShare%` — the Vendor's Claim view must show both `amount` and `deduction`
- Anti-Corruption Layer: the `vendor_claim_events` table is the ACL's local projection; it does not replace the authoritative Claim record in Claim Processing Service
- Tenant isolation: every query filters by `tenantId`

---

## Contracts Involved

- `contract-claim-vendor.md` — ClaimSubmitted event shape; verify `vendorId`, `deduction`, `campaignId` are present before writing your spec
- `contract-campaign-vendor.md` — `GET /campaigns/:id/summary` response shape from Campaign Service; verify `fundingVendorShare`, `fundingVendorId`, `budgetTotal`, `budgetBurned` are present
- No outbound contracts published by this service yet

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 32 | POST /vendors with valid body | Vendor created with ACTIVE status, vendorId returned |
| 33 | POST /vendors/:vendorId/funding-proposals with vendorShare: 60 | FundingProposal created, krogerShare: 40, status: SUBMITTED |
| 34 | POST /vendors/:vendorId/funding-proposals where Vendor status is INACTIVE | 422 response with reason |
| 35 | POST /vendors/:vendorId/funding-proposals with vendorShare: 100 | 422 validation error — krogerShare would be 0 |
| 36 | ClaimSubmitted event received for vendor-v001 | vendor_claim_events row persisted; GET /vendors/v001/claims returns it |

---

## Sprint 1 ADR Topics

Record an ADR for:
- Funding Proposal initial status — why SUBMITTED directly vs requiring explicit DRAFT → SUBMITTED transition; what the UX implications are for Vendors
- Vendor Campaign view strategy — why call Campaign Service on-demand vs materialise campaign data locally alongside the claims ACL
