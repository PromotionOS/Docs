# PromotionOS — Ubiquitous Language

> This is the single source of truth for all domain terms used across PromotionOS.
> Every team uses these exact words — in specs, ADRs, code, variable names, API fields, event payloads, and conversations.
> If a term is not here, it does not exist in the domain.

---

## Core Principle

Domain Driven Design requires that everyone on the team — developers, facilitators, and Claude — speaks the same language. When Team 1 says "Campaign" and Team 2 says "Promotion", they are talking about different things even if they mean the same thing. Ambiguity in language becomes ambiguity in code.

**Rule: Use these terms exactly as written. No synonyms. No abbreviations.**

---

## Domain Terms

### Tenant
A retail organization using PromotionOS. All data is isolated per tenant. A Tenant never sees another Tenant's data.

- **Code:** `tenantId: TenantId`
- **In events:** always present as `tenantId`
- **Example:** Kroger is a Tenant

---

### Campaign
The business intent behind a promotion. A Campaign is created by the Merchandising team and defines what is being promoted, for how long, and at what cost. A Campaign is the Aggregate Root of the Campaign bounded context.

- **Lifecycle:** `DRAFT → ACTIVE → PAUSED | EXPIRED | SCHEDULED`
- **Code:** `Campaign` (aggregate root)
- **Not called:** Promotion, Deal, Sale, Offer (those are different things)

---

### Offer
The customer-facing expression of a Campaign. An Offer defines the specific discount a customer receives. An Offer belongs to exactly one Campaign.

- **Types:** `PCT_OFF`, `AMT_OFF`, `BOGO`, `THRESHOLD`
- **Code:** `Offer` (entity inside Campaign aggregate)
- **Not called:** Discount, Deal, Coupon

---

### Funding
The financial agreement between a Vendor and Kroger that backs a Campaign. Funding defines who pays what percentage of the discount cost. A Campaign cannot be published without a confirmed Funding source.

- **Fields:** `vendorId`, `vendorShare: Percentage`, `krogerShare: Percentage`
- **Rule:** `vendorShare + krogerShare = 100%` always
- **Code:** `Funding` (entity inside Campaign aggregate)

---

### Budget
The financial cap on a Campaign's total discount spend. A Budget tracks how much has been burned in real time. When burn reaches the exhaustion threshold (95%) the Campaign auto-pauses.

- **Fields:** `totalAmount: Money`, `burnedAmount: Money`, `exhaustionThreshold: Percentage`
- **Code:** `Budget` (value object inside Campaign aggregate)
- **Not called:** Cap, Limit, Ceiling

---

### Segment
A named group of customers sharing qualifying criteria. A Customer can belong to multiple Segments simultaneously. Segment membership is recalculated from 90-day rolling purchase history.

- **System segments:** `segment-high-value`, `segment-mid-value`, `segment-low-value`, `segment-new-customer`, `segment-wine-buyer`, `segment-vitamin-buyer`, `segment-midwest`, `segment-southeast`, `segment-west`, `segment-northeast`, `segment-southwest`
- **Code:** `String` (segment name), `List<String>` (customer segments)

---

### Loyalty Tier
A customer's classification based on annual spend. Tier determines eligibility for Tier-restricted Campaigns.

| Tier | Annual Spend |
|------|-------------|
| `BASIC` | < $500 |
| `SILVER` | $500 — $1,999 |
| `GOLD` | $2,000 — $4,999 |
| `PLATINUM` | $5,000+ |

- **Code:** `LoyaltyTier` (value object — enum)
- **Rule:** Tier is recalculated on every customer profile fetch — never trust stored tier
- **Not called:** Level, Grade, Status

---

### Eligibility
The determination of whether a specific Customer qualifies for a specific Campaign's Offer at a given point in time. Eligibility is evaluated by the Eligibility bounded context.

- **Result:** `eligible: boolean`, `reason` (if ineligible)
- **Ineligibility reasons:** `SEGMENT_MISMATCH`, `THRESHOLD_NOT_MET`, `UPC_EXCLUDED`, `GEO_MISMATCH`, `STACK_EXCEEDED`, `CAMPAIGN_INACTIVE`
- **Code:** `EligibilityResult` (value object)

---

### Exclusion
A product (UPC) or category explicitly outside the scope of a Campaign's Offer. Exclusions are inherited downward through the category hierarchy — excluding `cat-alcohol` excludes all wine, beer, and spirits UPCs.

- **Types:** category-level, UPC-level
- **Code:** `Exclusion` (value object inside EligibilityRule aggregate)
- **Rule:** Exclusion inheritance is resolved by the Catalog bounded context, not Eligibility

---

### Threshold
The minimum condition a customer's cart must meet before an Offer activates.

- **Types:** `SPEND` (cart total ≥ amount), `QUANTITY` (item count ≥ number)
- **Rule:** Cart total for SPEND threshold is calculated after Excluded UPCs are removed
- **Code:** `Threshold` (value object inside EligibilityRule aggregate)

---

### Stack
The application of two or more Offers to a single customer transaction. Stacking is controlled by `stackLimit` per Campaign and `stackPermission` flag.

- **Rule:** Stack limit is the minimum `stackLimit` across all Campaigns being applied
- **Default:** `stackLimit: 1` — no stacking unless explicitly permitted
- **Code:** validated by `RuleEngine.validateStack()`

---

### UPC
Universal Product Code. The unique identifier for a product in the Catalog. UPCs belong to exactly one Category.

- **Code:** `UPC` (value object — wraps a string code)
- **Not called:** SKU, Product Code, Item Code

---

### Category
A grouping of UPCs in the Catalog hierarchy. Categories form a tree — a Category can have a parent and multiple children. Exclusion status is inherited downward.

- **Code:** `Category` (entity inside Catalog bounded context)

---

### Redemption
A confirmed application of an Offer to a customer transaction. A Redemption is immutable once confirmed — it cannot be updated or deleted.

- **Fields:** `idempotencyKey`, `customerId`, `campaignId`, `discountApplied`, `cartTotal`, `redeemedAt`
- **Status:** `CONFIRMED → CLAIMED`
- **Code:** `Redemption` (aggregate root of Redemption bounded context)
- **Not called:** Transaction, Purchase, Application

---

### Idempotency Key
A unique key provided by the POS system with each redemption request. Prevents duplicate Redemptions from being recorded if the same request is sent more than once. Idempotency is enforced per Tenant — the same key from two different Tenants is not a duplicate.

- **Code:** `IdempotencyKey` (value object)
- **Format:** `pos-{storeId}-txn-{transactionId}`

---

### Claim
A vendor reimbursement request generated automatically 24 hours after a Redemption is confirmed. A Claim calculates how much the Vendor owes Kroger based on the Funding split.

- **Fields:** `amount: Money`, `deduction: Money`, `vendorId`, `status`
- **Status:** `PENDING → SUBMITTED`
- **Rule:** `deduction = discountApplied × vendorShare%`
- **Code:** `Claim` (entity inside Redemption aggregate)

---

### Deduction
The amount withheld from a Vendor's next payment to settle a Claim. Deduction = Claim amount × vendorShare%.

- **Code:** `Money` field on `Claim`

---

### Lift
The incremental sales attributed to a Campaign above the baseline forecast.

- **Formula:** `lift = actualSalesPerDay - baselineSalesPerDay`
- **Formula:** `liftPercentage = (lift / baselineSalesPerDay) × 100`
- **Rule:** If baseline is zero — liftPercentage is `null`, not a division error
- **Code:** `Lift` (value object inside Analytics bounded context)

---

### ROI
Return on Investment for a Campaign. Measures how much incremental margin the Campaign generated relative to its funding cost.

- **Formula:** `roi = incrementalMargin / totalFundingCost`
- **Formula:** `incrementalMargin = lift × averageMarginPercent` (default 30%)
- **Rule:** If `totalFundingCost` is zero (100% Kroger-funded) — ROI is `null`, not a division error
- **Code:** `Percentage` field on campaign metrics

---

### Budget Burn
The real-time percentage of a Campaign's Budget that has been consumed by Redemptions.

- **Formula:** `budgetBurnPercent = (burnedAmount / totalAmount) × 100`
- **Rule:** When `budgetBurnPercent >= 95.0` → publish `BudgetExhausted` event exactly once
- **Code:** tracked by `BudgetBurnTracker` domain service

---

### Division
Kroger's geographic grouping of stores. Used for geo-targeting Campaigns to specific regions.

- **Values:** `division-all`, `division-midwest`, `division-southeast`, `division-west`, `division-northeast`, `division-southwest`
- **Rule:** `division-all` matches any customer regardless of their division

---

### Anti-Corruption Layer (ACL)
A translation component inside a bounded context that converts domain events or API responses from another context's language into the local context's language. Prevents domain model pollution across context boundaries.

- **Where used:** Eligibility Service translates CampaignPublished, CatalogItemExcluded, SegmentUpdated into its own domain language
- **Code:** `ACL` class/struct in the infrastructure layer of the consuming service

---

### Domain Event
An immutable record of something that happened in the domain. Domain Events are the mechanism by which bounded contexts communicate without tight coupling. Every event carries `tenantId`, `occurredAt`, and `schemaVersion`.

- **Rule:** Events are named in past tense — CampaignPublished, OfferRedeemed, BudgetExhausted
- **Rule:** Schema version must increment when event payload changes
- **Code:** immutable classes/structs with `schemaVersion: int`

---

### Contract
The formal agreement between two bounded contexts defining the exact shape of events and APIs they share. Contracts are versioned. Breaking a contract requires updating the relevant contract skill and notifying the consuming team.

---

### Spec
A document written before implementation that defines what a feature does, the domain rules it enforces, the edge cases it handles, and the acceptance criteria it must pass. Specs are written using the `sdd-spec-writer` skill from requirements.

- **Rule:** No implementation without a spec
- **Location:** Recorded in the team's sprint gist comments or ADR

---

### ADR (Architectural Decision Record)
A document recording a significant domain or technical decision — what was decided, why, what alternatives were considered, and what the consequences are.

- **Rule:** Every significant domain decision must have an ADR before merging
- **Written using:** `adr-recorder` skill

---

### RCA (Root Cause Analysis)
A document written when a bug is found, explaining what the bug is, why it was introduced, what the correct behaviour should be, and how it was fixed.

- **Rule:** Bug fixes require an RCA before the fix spec is written
- **Written using:** `rca-writer` skill

---

## What Not to Say

| ❌ Do not say | ✅ Say instead |
|--------------|---------------|
| Promotion | Campaign |
| Deal | Campaign or Offer |
| Coupon | Offer |
| Discount | Offer value or discountApplied |
| SKU | UPC |
| Product | UPC or Category |
| User | Customer (CX) or Merchandiser (MX) |
| Cap | Budget |
| Level / Grade | Loyalty Tier |
| Transaction | Redemption |
| Dupe / Double | Duplicate Redemption |
| Reimbursement | Claim |
| Holdback | Deduction |
