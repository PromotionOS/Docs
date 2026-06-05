# Team 7 — Vendor Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, vendor-service-guide, adr-recorder, pr-reviewer, analytics-vendor-contract, campaign-vendor-contract

---

## Context

Sprint 1 delivered Vendor profile CRUD, Funding Proposal submission, and Claim history via ACL. Sprint 2 delivered the Approval Workflow — FundingApproved and FundingRejected events are now published and camp-013 can be unblocked.

Vendors currently have no visibility into how their funded campaigns are performing before or during a campaign run. This sprint adds forecasting — Predicted Lift calculated from category benchmarks — and Burn Velocity derived from Analytics Service data. A pre-publish health check endpoint lets the Campaign Service (or a human operator) confirm a Vendor's position is sound before a Campaign goes ACTIVE.

---

## Sprint 3 Goal

Add Predicted Lift forecasting from category benchmarks, Burn Velocity from Analytics Service data, and a pre-publish health check endpoint for Vendor readiness.

---

## Requirements

### Requirement 1 — Category Benchmark Seed Data

1. Define the `category_benchmarks` table: `category_id` (PK), `baseline_sales_per_day` (decimal), `season_factor` (decimal, default 1.0), `promo_type_factor` (decimal), `promo_type` (`PCT_OFF` | `AMT_OFF` | `BOGO` | `THRESHOLD`)
2. Seed at minimum 5 benchmark rows covering the categories referenced by the test Campaigns (consult Campaign test data for category references)
3. `GET /category-benchmarks` — returns all benchmark rows; supports `promoType` filter
4. Benchmarks are managed data, not user input this sprint — no POST/PUT/DELETE endpoints yet

### Requirement 2 — Predicted Lift Calculation

1. `POST /vendors/:vendorId/funding-proposals/:proposalId/predicted-lift` — triggers a Predicted Lift calculation
2. Body: `{ categoryId, promoType }` — the category and promotion type the Campaign targets
3. Look up the matching `category_benchmarks` row for `{ categoryId, promoType }`
4. Apply the formula: `predictedLift = categoryBaseline × seasonFactor × promoTypeFactor`
5. Return: `{ proposalId, categoryId, promoType, predictedLift, baselineSalesPerDay, seasonFactor, promoTypeFactor, calculatedAt }`
6. If no benchmark exists for the given `{ categoryId, promoType }` → return 404 with message: `"No benchmark data for this category and promotion type"`
7. Persist the result in a `predicted_lifts` table: `proposal_id`, `category_id`, `promo_type`, `predicted_lift`, `calculated_at`
8. `GET /vendors/:vendorId/funding-proposals/:proposalId/predicted-lift` — returns the most recent calculation

### Requirement 3 — Burn Velocity from Analytics Service

1. `GET /vendors/:vendorId/campaigns/:campaignId/burn-velocity` — returns the current Burn Velocity for a Vendor's Campaign
2. Call Analytics Service `GET /analytics/campaigns/:campaignId/metrics` to fetch `burnedAmount` and `daysSinceStart`
3. Apply the formula: `burnVelocity = burnedAmount / daysSinceStart`
4. If `daysSinceStart` is 0 (campaign started today) → return `burnVelocity: null` with `reason: "Campaign started today — no burn velocity yet"`
5. Return: `{ campaignId, burnedAmount, daysSinceStart, burnVelocity, fetchedAt }`
6. If Analytics Service is unavailable → return 503 with message: `"Analytics Service unavailable — burn velocity cannot be calculated"`
7. Do not cache the result this sprint — call Analytics Service on each request (record ADR)

### Requirement 4 — Pre-Publish Health Check

1. `GET /vendors/:vendorId/health-check?campaignId={campaignId}` — returns a readiness summary for a Campaign before publication
2. Checks to perform and report:
   - `vendorStatus`: ACTIVE | INACTIVE | SUSPENDED
   - `fundingProposalStatus`: SUBMITTED | APPROVED | REJECTED | MISSING
   - `predictedLiftAvailable`: true | false (true if a Predicted Lift record exists for this proposalId)
   - `burnVelocityAvailable`: true | false (true if Analytics Service responded successfully)
3. Response shape: `{ campaignId, vendorId, checks: { vendorStatus, fundingProposalStatus, predictedLiftAvailable, burnVelocityAvailable }, readyToPublish: boolean }`
4. `readyToPublish: true` only when: `vendorStatus: ACTIVE` AND `fundingProposalStatus: APPROVED`
5. This endpoint is informational — it does not block publication. The Campaign Service enforces the actual gate via FundingApproved event.

---

## Domain Rules

From ubiquitous-language.md:
- Predicted Lift: `predictedLift = categoryBaseline × seasonFactor × promoTypeFactor` — use these exact variable names in code
- Predicted Lift is distinct from Lift — Lift is actual measured post-launch; Predicted Lift is a forecast before launch
- Burn Velocity: `burnVelocity = burnedAmount / daysSinceStart` — derived from Analytics Service data, not owned by Vendor Service
- Offer types: `PCT_OFF`, `AMT_OFF`, `BOGO`, `THRESHOLD` — use these exact values in benchmark lookup
- Vendor status `ACTIVE` is the only status that allows proposal submission and publication readiness
- Approval Threshold: `totalAmount >= $50,000` requires Approval — the health check reflects whether this gate has been cleared

---

## Contracts Involved

- `contract-analytics-vendor.md` — Analytics Service `GET /analytics/campaigns/:campaignId/metrics` response shape; confirm `burnedAmount` and `daysSinceStart` fields are present before writing your spec
- Pre-publish health check is a new endpoint — share the response contract with Campaign Service (Team 1) if they plan to call it
- No new events published this sprint

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 36 | POST predicted-lift with categoryId matching a benchmark row | Predicted Lift calculated and returned using formula |
| 37 | POST predicted-lift with categoryId with no benchmark | 404 with message |
| 38 | GET burn-velocity for a campaign active 5 days with $10,000 burned | burnVelocity: 2000.00 returned |
| 39 | GET burn-velocity when Analytics Service is unavailable | 503 returned, no crash |
| 40 | GET health-check for vendor with ACTIVE status and APPROVED proposal | readyToPublish: true |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Burn Velocity caching decision — why call Analytics Service on every request in Sprint 3, what the TTL and cache invalidation strategy should be in production (campaign metrics change frequently)
- Benchmark data management — why benchmarks are seeded data vs user-managed this sprint, and what the admin interface would look like
- Pre-publish health check as informational vs enforcing gate — why the Vendor Service does not block Campaign publication directly and why the Campaign Service event-driven approach is preferred
