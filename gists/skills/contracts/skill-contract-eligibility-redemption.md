---
name: contract-eligibility-redemption
description: Contract skill for Redemption Service consuming from Eligibility Service — ACL translation, field mapping, blast radius guide
metadata:
  type: reference
---

# Contract: Eligibility → Redemption

## What This Contract Covers

Eligibility Service exposes a synchronous REST API that Redemption Service must call before confirming any Redemption. This is a blocking gate — a Redemption cannot be confirmed without a successful eligibility check. The contract governs both the request Redemption Service must send and the response shapes it must handle.

## Events / APIs (Summary)

| Surface | Type | Purpose |
|---------|------|---------|
| `POST /eligibility/check` | REST | Gate call before confirming a Redemption |
| `GET /offers` | REST | Frontend offer display — not called by Redemption Service |

## ACL Translation Guide

### POST /eligibility/check

**What Redemption Service sends:**
```json
{
  "tenantId": "string",
  "customerId": "uuid",
  "campaignId": "uuid",
  "cartTotal": "number",
  "cartUPCs": ["string"]
}
```

**What arrives back (eligible):**
```json
{
  "eligible": true,
  "tenantId": "string",
  "customerId": "uuid",
  "campaignId": "uuid",
  "offerType": "PCT_OFF | AMT_OFF | BOGO | THRESHOLD",
  "discountApplied": "number",
  "evaluatedAt": "ISO-8601 timestamp"
}
```

**What arrives back (ineligible):**
```json
{
  "eligible": false,
  "tenantId": "string",
  "customerId": "uuid",
  "campaignId": "uuid",
  "reason": "SEGMENT_MISMATCH | THRESHOLD_NOT_MET | UPC_EXCLUDED | GEO_MISMATCH | STACK_EXCEEDED | CAMPAIGN_INACTIVE",
  "evaluatedAt": "ISO-8601 timestamp"
}
```

**How to translate in your ACL:**
```java
// infrastructure/acl/EligibilityClient.java

public EligibilityResult checkEligibility(RedemptionRequest request) {
    var body = EligibilityCheckRequest.builder()
        .tenantId(request.getTenantId())
        .customerId(request.getCustomerId())
        .campaignId(request.getCampaignId())
        .cartTotal(request.getCartTotal())
        .cartUPCs(request.getCartUPCs())   // pass raw UPCs — no pre-filtering
        .build();

    try {
        var response = httpClient.post("/eligibility/check", body, EligibilityCheckResponse.class);

        if (response.isEligible()) {
            return EligibilityResult.eligible(
                response.getCampaignId(),
                response.getOfferType(),
                response.getDiscountApplied()
            );
        } else {
            return EligibilityResult.ineligible(
                response.getCampaignId(),
                IneligibilityReason.valueOf(response.getReason())
            );
        }

    } catch (HttpClientErrorException e) {
        return switch (e.getStatusCode().value()) {
            case 404 -> throw new EligibilityResourceNotFoundException(e.getMessage());
            case 403 -> throw new TenantMismatchException();
            case 400 -> throw new InvalidRedemptionRequestException(e.getResponseBodyAsString());
            default  -> throw new EligibilityServiceException(e);
        };
    } catch (ResourceAccessException e) {
        // Eligibility Service is unreachable — caller returns 503
        throw new EligibilityUnavailableException(e);
    }
}
```

**How Redemption Service uses the result:**
```java
// domain/service/RedemptionService.java

public Redemption confirm(RedemptionRequest request) {
    EligibilityResult result = eligibilityClient.checkEligibility(request);

    if (!result.isEligible()) {
        throw new OfferIneligibleException(result.getReason()); // → 400 OFFER_INELIGIBLE
    }

    return Redemption.confirm(
        request.getIdempotencyKey(),
        request.getCustomerId(),
        request.getCampaignId(),
        result.getDiscountApplied(),   // from eligibility response
        request.getCartTotal(),
        request.getStoreId()
    );
}
```

**Fields to watch:**
- `discountApplied` → the computed discount amount from Eligibility; Redemption Service must use this value verbatim — do not recompute the discount locally
- `reason` → only present when `eligible: false`; map to `IneligibilityReason` enum — the values are fixed (`SEGMENT_MISMATCH`, `THRESHOLD_NOT_MET`, `UPC_EXCLUDED`, `GEO_MISMATCH`, `STACK_EXCEEDED`, `CAMPAIGN_INACTIVE`)
- `evaluatedAt` → ISO-8601 timestamp; store this on the `Redemption` aggregate for audit trail
- `offerType` → present only when `eligible: true`; do not attempt to read `offerType` from an ineligible response

## What Changes When The Contract Changes

### If producer adds a new optional response field
Add the field to `EligibilityCheckResponse` POJO with a null default. No API version bump needed. Map to `EligibilityResult` if Redemption Service needs to act on it.

### If producer renames a field
Eligibility Service bumps the path to `/v2/eligibility/check`. Update `EligibilityClient` to call the new path. The old path must remain available during the transition window. Do not hard-delete the v1 client until Eligibility Service retires v1.

### If producer adds a new ineligibility reason
Add the new value to `IneligibilityReason` enum. If your switch/case handling is exhaustive, add a default arm that logs and returns a generic ineligible result rather than throwing — this prevents a new reason code from crashing Redemption Service.

### If producer removes a field or changes an error code
Breaking change — coordinate with Team 2 before merging. Update `EligibilityCheckResponse`, `EligibilityResult`, and all tests before deploying.

## Blast Radius Checklist

When you receive a `skill-contract-eligibility-redemption` update — check these files in your repo:
- [ ] `infrastructure/acl/EligibilityClient.java` — request construction, response parsing, error mapping
- [ ] `infrastructure/acl/EligibilityCheckRequest.java` — request POJO fields
- [ ] `infrastructure/acl/EligibilityCheckResponse.java` — response POJO fields
- [ ] `domain/model/EligibilityResult.java` — domain value object that wraps the translated response
- [ ] `domain/model/IneligibilityReason.java` — enum; must include all reason values from the contract
- [ ] `domain/service/RedemptionService.java` — how it branches on `EligibilityResult`
- [ ] `test/acl/EligibilityClientTest.java` — stub response JSON for eligible, ineligible, and all error cases
- [ ] Any hardcoded strings like `"CAMPAIGN_INACTIVE"` or `"/eligibility/check"` scattered in tests

## Common Mistakes

**Mistake 1 — Pre-filtering `cartUPCs` before sending to Eligibility.**
The contract states: "Redemption Service must pass the exact `cartUPCs` from the POS transaction — no filtering before the call." Filtering excluded UPCs before the call bypasses Eligibility's threshold calculation, which uses the full cart.

**Mistake 2 — Treating a 503 from Eligibility as an ineligible result.**
If Eligibility Service is unreachable (`ResourceAccessException` / connection timeout), Redemption Service must return `503 ELIGIBILITY_UNAVAILABLE` to the caller — not `400 OFFER_INELIGIBLE`. These are different failure modes. A 503 means the check was not performed; a 400 means the check was performed and the customer failed.

**Mistake 3 — Reading `discountApplied` from the ineligible response.**
The `discountApplied` field is only present on `eligible: true` responses. Attempting to use it on an ineligible response will produce a null value and silently apply a zero discount if not guarded.

## Validation

| Scenario | Request | Expected Response |
|----------|---------|-----------------|
| 1 | `cust-002`, `camp-002`, `cartTotal: 22.50` | `eligible: true`, `discountApplied: 4.99` |
| 2 | `cust-002`, `camp-004` | `eligible: false`, `reason: SEGMENT_MISMATCH` |
| 3 | `cust-001`, `camp-004` | `eligible: true`, `discountApplied: 30% of wine UPC prices` |
| 4 | `cust-001`, `camp-001`, `cartUPCs` includes `upc-wine-cab` | `eligible: false`, `reason: UPC_EXCLUDED` |
| 5 | `cust-003`, `camp-003`, `cartTotal: 35.00` | `eligible: false`, `reason: THRESHOLD_NOT_MET` |
| 6 | `cust-003`, `camp-003`, `cartTotal: 67.50` | `eligible: true`, `discountApplied: 10.00` |
| 11 | any customer, `camp-006` (PAUSED) | `eligible: false`, `reason: CAMPAIGN_INACTIVE` |
| 14 | `cust-010`, `camp-001` (division-southwest) | `eligible: false`, `reason: GEO_MISMATCH` |
| — | Eligibility Service down | `EligibilityUnavailableException` → caller receives 503 |
| — | `campaignId` not found | 404 response → `EligibilityResourceNotFoundException` |
