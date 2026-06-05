# Team 8 — Notification Service — Sprint 3

> Type: Build via SDD
> Duration: 2 hours
> Skills: sdd-spec-writer, ddd-guide, notification-service-guide, adr-recorder, pr-reviewer, vendor-notification-contract, campaign-notification-contract

---

## Context

Sprint 1 delivered email for CampaignPaused and BudgetExhausted. Sprint 2 added the WebSocket hub and Store Manager routing via Division lookup. Vendors and the MX team are still not notified of funding decisions, and external vendor systems cannot receive webhooks yet.

This sprint expands the event surface to FundingApproved and FundingRejected (published by the Vendor Service being built in parallel by Team 7), wires ClaimSubmitted to a webhook Delivery Channel, and exposes a Notification History API for the vendor portal view.

---

## Sprint 3 Goal

Deliver FundingApproved/FundingRejected notifications to the MX team and Vendor contacts, add ClaimSubmitted webhook delivery, and expose a Notification History API.

---

## Requirements

### Requirement 1 — FundingApproved Notifications

1. Subscribe to the `funding.approved` topic — published by Vendor Service when an Approval is confirmed
2. On receiving `FundingApproved`: extract `tenantId`, `vendorId`, `campaignId`, `approvedAmount`, `approvedAt`
3. Load Notification Preferences for `role: MX_TEAM` and `eventType: FUNDING_APPROVED`
4. Load Notification Preferences for `role: VENDOR_CONTACT` and `eventType: FUNDING_APPROVED`
5. Deliver EMAIL to enabled MX_TEAM recipients
6. Deliver EMAIL to enabled VENDOR_CONTACT recipients — use `vendorContactEmail` from the event payload
7. Push WebSocket alert to connected MX_TEAM clients for the matching `tenantId`
8. Log all delivery attempts to the `notifications` table

### Requirement 2 — FundingRejected Notifications

1. Subscribe to the `funding.rejected` topic
2. On receiving `FundingRejected`: extract `tenantId`, `vendorId`, `campaignId`, `rejectionReason`, `rejectedAt`
3. Deliver EMAIL to MX_TEAM and VENDOR_CONTACT per their Notification Preferences
4. Push WebSocket alert to connected MX_TEAM clients
5. Email body must include `rejectionReason` — this is the primary actionable field for the Vendor
6. Log all delivery attempts

### Requirement 3 — ClaimSubmitted Webhook Delivery

1. Subscribe to the `claim.submitted` topic — published by Claim Processing Service
2. On receiving `ClaimSubmitted`: extract `tenantId`, `vendorId`, `claimId`, `amount`, `deduction`, `submittedAt`
3. Load Notification Preferences for `role: VENDOR_CONTACT` and `eventType: CLAIM_SUBMITTED`
4. For each enabled preference with `channel: WEBHOOK` — look up the vendor's registered webhook URL from the `notification_preferences` table (`webhookUrl` field)
5. POST the ClaimSubmitted payload to the webhook URL with a 5-second timeout
6. On HTTP 2xx response — log `status: SENT`
7. On timeout or non-2xx — log `status: FAILED`; do not retry this sprint (record ADR for retry strategy)
8. Sign the webhook body with an HMAC-SHA256 signature using a per-vendor secret stored in Supabase; include as `X-PromotionOS-Signature` header

### Requirement 4 — Notification History API

1. `GET /notifications/history` — returns a filtered, paginated history
2. Required query param: `tenantId`
3. Optional params: `vendorId`, `eventType`, `channel`, `status`, `fromDate`, `toDate`
4. Response: `{ data: [NotificationLog], total: int, page: int, limit: int }`
5. This endpoint is consumed by the Vendor Portal — ensure `vendorId` filter works correctly so a Vendor sees only their own notification history
6. Add `GET /notifications/:id` — single log entry lookup by ID

---

## Domain Rules

From ubiquitous-language.md:
- Claim: `deduction = discountApplied × vendorShare%` — the webhook payload must carry both `amount` and `deduction` so the Vendor system can reconcile
- Funding Proposal lifecycle: DRAFT → SUBMITTED → APPROVED | REJECTED — FundingApproved and FundingRejected are terminal transitions
- Approval Threshold: campaigns with `totalAmount >= $50,000` require Approval — FundingApproved unblocks Campaign publication
- Vendor is a first-class entity: `VENDOR_CONTACT` role is distinct from `MX_TEAM` and `STORE_MANAGER`
- Delivery Channel `WEBHOOK` is for external system push — Notification Service must never expose internal system details in the webhook payload

---

## Contracts Involved

- `contract-vendor-notification.md` — FundingApproved and FundingRejected event shapes from Vendor Service; verify `vendorContactEmail` is present in the payload
- `contract-claim-notification.md` — ClaimSubmitted event shape from Claim Processing Service; verify `deduction` field is included
- Notification History API is a new outbound contract this sprint — agree shape with Vendor Service (Team 7) before coding

---

## Validation Scenarios This Sprint

| # | Input | Expected |
|---|-------|----------|
| 36 | FundingApproved received for camp-013 | EMAIL to MX_TEAM and VENDOR_CONTACT; WebSocket push to MX_TEAM |
| 37 | FundingRejected received for camp-013 with rejectionReason: "Budget exceeds category cap" | EMAIL includes rejection reason; VENDOR_CONTACT receives it |
| 38 | ClaimSubmitted for vendor-v001, webhook URL registered, webhook returns 200 | Log status SENT; X-PromotionOS-Signature header present |
| 39 | ClaimSubmitted, webhook URL returns 503 | Log status FAILED; no crash; no retry |
| 40 | GET /notifications/history?tenantId=t-001&vendorId=v-001 | Returns only notifications for that vendor |

---

## Sprint 3 ADR Topics

Record an ADR for:
- Webhook retry strategy — why no retry in Sprint 3, what the plan is (exponential backoff queue, dead-letter) for a future sprint
- HMAC signature scheme — key rotation approach, how vendors verify the signature, why SHA-256 over alternatives
- Notification History API ownership — Notification Service owns the read API vs Vendor Service aggregating its own view: why Notification Service is the authority
