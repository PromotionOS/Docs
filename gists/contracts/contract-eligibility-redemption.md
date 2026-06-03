# Contract — Eligibility → Redemption

> Producer: Eligibility Service (Team 2)
> Consumer: Redemption Service (Team 3)
> Type: REST API
> Status: LOCKED — changes require contract skill update + Team 3 notification

---

## REST API Exposed by Eligibility Service

Redemption Service calls this API before confirming any Redemption to verify the customer is still eligible at the moment of redemption.

### POST /eligibility/check

**Request:**
```json
{
  "tenantId": "string",
  "customerId": "uuid",
  "campaignId": "uuid",
  "cartTotal": "number",
  "cartUPCs": ["string"]
}
```

**Field rules:**
- `tenantId` — required, must match campaign's tenantId
- `customerId` — required, must exist in Catalog & Customer Service
- `campaignId` — required, must be an ACTIVE campaign
- `cartTotal` — required, must be >= 0
- `cartUPCs` — required, list of UPC codes in the customer's cart

**Response — Eligible:**
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

**Response — Ineligible:**
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

**Error responses:**
```json
{ "error": "CAMPAIGN_NOT_FOUND", "campaignId": "uuid" }     // 404
{ "error": "CUSTOMER_NOT_FOUND", "customerId": "uuid" }     // 404
{ "error": "TENANT_MISMATCH" }                               // 403
{ "error": "INVALID_REQUEST", "details": "string" }         // 400
```

---

## GET /offers

Called by Offer Delivery layer (Frontend) to get all eligible offers for a customer. Not used by Redemption Service directly but part of this context's API surface.

**Request:**
```
GET /offers?customerId={customerId}&tenantId={tenantId}&cartTotal={cartTotal}
```

**Response:**
```json
{
  "tenantId": "string",
  "customerId": "uuid",
  "eligibleOffers": [
    {
      "campaignId": "uuid",
      "campaignName": "string",
      "offerType": "PCT_OFF | AMT_OFF | BOGO | THRESHOLD",
      "discountApplied": "number",
      "expiresAt": "YYYY-MM-DD"
    }
  ],
  "ineligibleOffers": [
    {
      "campaignId": "uuid",
      "campaignName": "string",
      "reason": "string"
    }
  ],
  "evaluatedAt": "ISO-8601 timestamp"
}
```

---

## Redemption Service Usage Rules

1. Redemption Service MUST call `POST /eligibility/check` before confirming any Redemption
2. If `eligible: false` → Redemption Service returns `400 OFFER_INELIGIBLE` to the caller
3. If Eligibility Service is unavailable → Redemption Service returns `503 ELIGIBILITY_UNAVAILABLE`
4. Redemption Service must pass the exact `cartUPCs` from the POS transaction — no filtering before the call

---

## Breaking Change Protocol

If Team 2 changes any field in the request or response:

1. Update `skill-contract-eligibility-redemption.md` with what changed
2. Notify Team 3 — they must update their Eligibility API client
3. Bump API version if breaking (`/v2/eligibility/check`)

**Safe to add:** New optional response fields
**Breaking:** Rename fields, change types, remove fields, change error codes

---

## Validation

| Scenario | Request | Expected Response |
|----------|---------|-----------------|
| 1 | cust-002, camp-002, cartTotal:22.50 | eligible:true, discountApplied:4.99 |
| 2 | cust-002, camp-004 | eligible:false, SEGMENT_MISMATCH |
| 3 | cust-001, camp-004 | eligible:true, discountApplied:30% of wine UPC prices |
| 4 | cust-001, camp-001, cartUPCs includes upc-wine-cab | eligible:false, UPC_EXCLUDED |
| 5 | cust-003, camp-003, cartTotal:35.00 | eligible:false, THRESHOLD_NOT_MET |
| 6 | cust-003, camp-003, cartTotal:67.50 | eligible:true, discountApplied:10.00 |
| 11 | any customer, camp-006 (PAUSED) | eligible:false, CAMPAIGN_INACTIVE |
| 14 | cust-010, camp-001 (division-southwest, not in scope) | eligible:false, GEO_MISMATCH |
