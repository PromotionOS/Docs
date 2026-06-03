# PromotionOS — Skeleton Design

> What gets pre-built before the session.
> Rule: domain structure only. Zero business logic except Campaign Service and Analytics Service (which have bugs planted).

---

## Folder Structure — All Services

### Java Services (Campaign, Eligibility, Redemption)

```
{service-name}/
├── src/
│   ├── main/
│   │   ├── java/com/promotionos/{service}/
│   │   │   ├── domain/
│   │   │   │   ├── model/          ← entities + value objects (fields only)
│   │   │   │   ├── service/        ← domain service interfaces
│   │   │   │   ├── repository/     ← repository interfaces
│   │   │   │   └── event/          ← domain event classes
│   │   │   ├── application/        ← application service (stubs → throws NotImplemented)
│   │   │   ├── infrastructure/
│   │   │   │   ├── repository/     ← JPA implementations (CRUD wired)
│   │   │   │   └── event/          ← Redis publisher/consumer wired
│   │   │   └── api/                ← REST controllers (routes defined, calls app service)
│   │   └── resources/
│   │       └── application.yml     ← DB + Redis config
│   └── test/
│       └── java/com/promotionos/{service}/
│           └── contract/           ← contract tests (FAILING)
├── db/
│   └── migrations/                 ← Flyway SQL migrations + seed data
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── pom.xml
```

### Go Services (Catalog & Customer, Analytics)

```
{service-name}/
├── cmd/
│   └── main.go                     ← boots, connects DB + Redis
├── internal/
│   ├── domain/
│   │   ├── model/                  ← structs (fields only, no methods)
│   │   ├── service/                ← interfaces
│   │   └── repository/             ← interfaces
│   ├── application/                ← service stubs (return ErrNotImplemented)
│   ├── infrastructure/
│   │   ├── repository/             ← Gorm implementations wired
│   │   └── event/                  ← Redis publisher/consumer wired
│   └── api/
│       └── handler.go              ← Gin routes defined, calls application layer
├── db/
│   └── migrations/                 ← SQL migrations + seed data
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── go.mod
```

### Frontend (React.js)

```
frontend-dashboard/
├── src/
│   ├── pages/                      ← page scaffolds, no logic
│   ├── components/                 ← component scaffolds, no logic
│   ├── api/                        ← API client functions returning null
│   └── App.jsx                     ← routing wired
├── public/
├── .env                            ← API URLs pointing to Railway
├── package.json
└── scripts/
    ├── test.sh
    └── deploy.sh
```

---

## Domain Model — Per Service

### Campaign Service

**Entities + Value Objects (fields only, no methods)**

```java
// Campaign.java — Aggregate Root
public class Campaign {
    private UUID id;
    private TenantId tenantId;
    private String name;
    private CampaignStatus status; // DRAFT, ACTIVE, PAUSED, SCHEDULED, EXPIRED
    private Offer offer;
    private Funding funding;
    private Budget budget;
    private DateRange dateRange;
    private boolean stackPermission;
    private List<String> geoScope;
    private String segmentRestriction;
    // No methods — teams implement
}

// Offer.java — Entity
public class Offer {
    private UUID id;
    private OfferType type; // PCT_OFF, AMT_OFF, BOGO, THRESHOLD
    private Money value;
    private Money thresholdAmount;
    private List<UPC> upcScope;
}

// Funding.java — Entity
public class Funding {
    private UUID id;
    private String vendorId;
    private Percentage vendorShare;
    private Percentage krogerShare;
}

// Budget.java — Value Object
public class Budget {
    private Money totalAmount;
    private Money burnedAmount;
    private Percentage exhaustionThreshold; // default 95%
}

// Value Objects
public class Money { private BigDecimal amount; private String currency; }
public class Percentage { private BigDecimal value; }
public class DateRange { private LocalDate startDate; private LocalDate endDate; }
public class TenantId { private UUID value; }
public class UPC { private String code; }

// Domain Events
public class CampaignPublished { UUID campaignId; TenantId tenantId; Instant occurredAt; CampaignPublishedPayload payload; }
public class CampaignPaused { UUID campaignId; TenantId tenantId; String reason; Instant occurredAt; }
public class BudgetExhausted { UUID campaignId; TenantId tenantId; Money totalBurned; Instant occurredAt; }
```

**Repository Interface**
```java
public interface CampaignRepository {
    Campaign findById(UUID id, TenantId tenantId);
    List<Campaign> findActive(TenantId tenantId);
    List<Campaign> findByUPC(UPC upc, TenantId tenantId);
    Campaign save(Campaign campaign);
}
```

**Domain Service Interfaces**
```java
public interface FundingValidator {
    void validate(Funding funding); // throws DomainException if invalid
}

public interface ConflictResolver {
    void checkUPCOverlap(Campaign campaign, List<Campaign> active); // throws DomainException if conflict
}
```

**Application Service (stubs)**
```java
public class CampaignApplicationService {
    public Campaign createDraft(CreateCampaignCommand cmd) {
        throw new NotImplementedException("Team 1: implement campaign draft creation");
    }
    public Campaign publish(UUID campaignId, TenantId tenantId) {
        throw new NotImplementedException("Team 1: implement campaign publish");
    }
    public Campaign addFunding(UUID campaignId, Funding funding, TenantId tenantId) {
        throw new NotImplementedException("Team 1: implement funding addition");
    }
}
```

---

### Eligibility Service

```java
// EligibilityRule.java — Aggregate Root
public class EligibilityRule {
    private UUID id;
    private TenantId tenantId;
    private UUID campaignId;
    private Integer stackLimit;
    private List<Exclusion> exclusions;
    private Threshold threshold;
    private String segmentRestriction;
    private List<String> geoScope;
}

// Exclusion.java — Value Object
public class Exclusion {
    private String categoryId;
    private UPC upc;
    private String reason;
}

// Threshold.java — Value Object
public class Threshold {
    private ThresholdType type; // SPEND, QUANTITY
    private Money spendValue;
    private Integer quantityValue;
}

// Domain Service Interfaces
public interface RuleEngine {
    EligibilityResult evaluate(EligibilityRule rule, CustomerProfile customer, Cart cart);
}

public interface SegmentMatcher {
    boolean matches(CustomerProfile customer, String segmentRestriction);
}
```

---

### Redemption Service

```java
// Redemption.java — Aggregate Root
public class Redemption {
    private UUID id;
    private TenantId tenantId;
    private IdempotencyKey idempotencyKey;
    private CustomerId customerId;
    private UUID campaignId;
    private Money discountApplied;
    private Money cartTotal;
    private String storeId;
    private String division;
    private Instant redeemedAt;
    private RedemptionStatus status; // CONFIRMED, CLAIMED
    // Immutable after confirmation — no setters
}

// Claim.java — Entity
public class Claim {
    private UUID id;
    private UUID redemptionId;
    private String vendorId;
    private Money amount;
    private Money deduction;
    private ClaimStatus status; // PENDING, SUBMITTED
    private Instant scheduledAt;
}

// Value Objects
public class IdempotencyKey { private String key; }
public class CustomerId { private UUID value; }

// Domain Service Interfaces
public interface IdempotencyGuard {
    boolean isDuplicate(IdempotencyKey key, TenantId tenantId);
    void register(IdempotencyKey key, TenantId tenantId);
}

public interface ClaimGenerator {
    Claim generate(Redemption redemption, Funding funding);
}
```

---

### Catalog & Customer Service

```go
// model/upc.go
type UPC struct {
    Code     string
    Name     string
    Price    float64
    Category string
}

// model/category.go
type Category struct {
    ID           string
    Name         string
    ParentID     string
    Excluded     bool
    ExcludedFrom string // categoryId that caused inheritance
    ExcludeReason string
}

// model/customer.go
type Customer struct {
    ID           string
    TenantID     string
    LoyaltyTier  string // BASIC, SILVER, GOLD, PLATINUM
    Segments     []string
    Division     string
    AnnualSpend  float64
}

// service/hierarchy_resolver.go
type HierarchyResolver interface {
    IsExcluded(upc UPC, tenantID string) (bool, string)
    ResolveInheritance(categoryID string, tenantID string) []string
}

// service/segment_calculator.go
type SegmentCalculator interface {
    Calculate(customerID string, tenantID string) []string
}
```

---

### Analytics Service

```go
// model/analytics.go
type CampaignMetrics struct {
    CampaignID         string
    TenantID           string
    BaselineSalesPerDay float64
    ActualSalesPerDay   float64
    Lift                float64
    LiftPercentage      float64
    TotalFundingCost    float64
    ROI                 float64
    BudgetBurnPercent   float64
    RedemptionCount     int
}

// service/lift_calculator.go
type LiftCalculator interface {
    Calculate(campaignID string, tenantID string) (float64, float64) // lift, roi
}

// service/budget_burn_tracker.go
type BudgetBurnTracker interface {
    Update(campaignID string, tenantID string, amount float64)
    GetBurnPercent(campaignID string, tenantID string) float64
    IsExhausted(campaignID string, tenantID string) bool
}
```

---

## DB Schemas

### Campaign DB
```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    region VARCHAR(100)
);

CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    offer_type VARCHAR(50),
    offer_value DECIMAL(10,2),
    offer_threshold_amount DECIMAL(10,2),
    upc_scope JSONB,
    stack_permission BOOLEAN DEFAULT FALSE,
    segment_restriction VARCHAR(50),
    geo_scope JSONB,
    date_start DATE,
    date_end DATE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE fundings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    tenant_id UUID NOT NULL,
    vendor_id VARCHAR(255) NOT NULL,
    vendor_share DECIMAL(5,2) NOT NULL,
    kroger_share DECIMAL(5,2) NOT NULL
);

CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    tenant_id UUID NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    burned_amount DECIMAL(12,2) DEFAULT 0,
    currency VARCHAR(10) DEFAULT 'USD',
    exhaustion_threshold DECIMAL(5,2) DEFAULT 95.0
);
```

### Eligibility DB
```sql
CREATE TABLE eligibility_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    stack_limit INTEGER DEFAULT 1,
    segment_restriction VARCHAR(50),
    geo_scope JSONB,
    threshold_type VARCHAR(50),
    threshold_spend DECIMAL(10,2),
    threshold_quantity INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE exclusions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES eligibility_rules(id),
    tenant_id UUID NOT NULL,
    category_id VARCHAR(255),
    upc_code VARCHAR(255),
    reason VARCHAR(255)
);
```

### Redemption DB
```sql
CREATE TABLE redemptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    customer_id UUID NOT NULL,
    campaign_id UUID NOT NULL,
    discount_applied DECIMAL(10,2) NOT NULL,
    cart_total DECIMAL(10,2) NOT NULL,
    store_id VARCHAR(255),
    division VARCHAR(255),
    redeemed_at TIMESTAMPTZ NOT NULL,
    status VARCHAR(50) DEFAULT 'CONFIRMED',
    UNIQUE(idempotency_key, tenant_id)
);

CREATE TABLE claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    redemption_id UUID NOT NULL REFERENCES redemptions(id),
    tenant_id UUID NOT NULL,
    vendor_id VARCHAR(255) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    deduction DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    scheduled_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ
);
```

### Catalog DB
```sql
CREATE TABLE categories (
    id VARCHAR(255) PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    parent_id VARCHAR(255),
    excluded BOOLEAN DEFAULT FALSE,
    excluded_from VARCHAR(255),
    exclude_reason VARCHAR(255)
);

CREATE TABLE upcs (
    code VARCHAR(255) PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    category_id VARCHAR(255) REFERENCES categories(id)
);

CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(255),
    loyalty_tier VARCHAR(50) DEFAULT 'BASIC',
    segments JSONB DEFAULT '[]',
    division VARCHAR(255),
    annual_spend DECIMAL(12,2) DEFAULT 0
);
```

### Analytics DB
```sql
CREATE TABLE campaign_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    baseline_sales_per_day DECIMAL(12,2),
    actual_sales_per_day DECIMAL(12,2),
    lift DECIMAL(12,2),
    lift_percentage DECIMAL(5,2),
    total_funding_cost DECIMAL(12,2) DEFAULT 0,
    roi DECIMAL(8,2),
    budget_burn_percent DECIMAL(5,2) DEFAULT 0,
    redemption_count INTEGER DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Pre-planted Bugs

### Bug 1 — Campaign Service (Java)
**Location:** `BudgetTracker` — budget exhaustion check
**Bug:** Uses `>=` instead of `>` for 95% threshold check causing campaigns to pause one redemption too early
**Symptom:** Scenario 8 fails — campaign pauses at 94.9% not 95%
**RCA hint:** Off-by-one in floating point comparison

### Bug 2 — Analytics Service (Go)
**Location:** `LiftCalculator` — ROI calculation
**Bug:** Divides by `totalFundingCost` before checking if it is zero — causes divide-by-zero panic on Kroger-funded campaigns where vendor share is 0
**Symptom:** Scenario for camp-003 (100% Kroger funded) returns 500 error
**RCA hint:** Missing zero-guard on denominator

---

## What Teams Build

| Team | Builds |
|------|--------|
| Team 1 — Campaign | Bug hunt + RCA + fix the budget threshold bug via SDD |
| Team 2 — Eligibility | Full RuleEngine + SegmentMatcher + application service |
| Team 3 — Redemption | Full IdempotencyGuard + Redemption.confirm() + ClaimGenerator |
| Team 4 — Catalog | Full HierarchyResolver + SegmentCalculator + application service |
| Team 5 — Analytics | Bug hunt + RCA + fix the ROI divide-by-zero via SDD |
| Team 6 — Frontend | Campaign form + offer list + burn chart wired to real APIs |
