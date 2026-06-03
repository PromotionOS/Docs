# PromotionOS — Railway Resource Creation Guide

> Use this guide to provision all infrastructure on Railway before skeleton build begins.
> Everything here must be running before any service skeleton is deployed.
> Execute in the exact order listed.

---

## Railway Project

**Project name:** `promotionos`
**Region:** US West (or closest to your location)

---

## Step 1 — Create PostgreSQL Databases

Create 6 separate PostgreSQL databases. One per bounded context. They must never share a database.

For each database — go to Railway → New Service → Database → PostgreSQL.

| Database Name | Used By | Notes |
|--------------|---------|-------|
| `campaign-db` | campaign-service | Stores campaigns, offers, funding, budgets |
| `eligibility-db` | eligibility-service | Stores eligibility rules, exclusions, thresholds |
| `redemption-db` | redemption-service | Stores redemptions, claims, idempotency keys |
| `catalog-db` | catalog-customer-service | Stores UPCs, categories, customers, purchase history |
| `analytics-db` | analytics-service | Stores campaign metrics, lift, burn, ROI |
| `eventstore-db` | event-store | Append-only domain event log |

### For each PostgreSQL database:

1. Railway → Project → **+ New** → **Database** → **PostgreSQL**
2. Name it exactly as listed above
3. Once provisioned — click the database → **Connect** tab
4. Copy the `DATABASE_URL` (format: `postgresql://user:pass@host:port/dbname`)
5. Store it — you will set this as `DB_URL` env var on each service

---

## Step 2 — Create Redis Instance

One Redis instance shared as the event bus across all services.

1. Railway → Project → **+ New** → **Database** → **Redis**
2. Name it: `promotionos-redis`
3. Once provisioned — click Redis → **Connect** tab
4. Copy the `REDIS_URL` (format: `redis://default:pass@host:port`)
5. Store it — you will set this as `REDIS_URL` env var on ALL services

### Redis Pub/Sub Channels

These channels are pre-defined. All services publish and consume on these exact channel names. No deviation.

| Channel | Publisher | Consumers |
|---------|-----------|----------|
| `promotionos.campaign.published` | campaign-service | eligibility-service, analytics-service, event-store |
| `promotionos.campaign.paused` | campaign-service | eligibility-service, frontend-dashboard (polling), event-store |
| `promotionos.catalog.item.excluded` | catalog-customer-service | eligibility-service, event-store |
| `promotionos.customer.segment.updated` | catalog-customer-service | eligibility-service, event-store |
| `promotionos.redemption.redeemed` | redemption-service | analytics-service, event-store |
| `promotionos.redemption.claim.submitted` | redemption-service | analytics-service, event-store |
| `promotionos.analytics.budget.exhausted` | analytics-service | campaign-service, event-store |
| `promotionos.campaign.budget.updated` | campaign-service | analytics-service, event-store |

---

## Step 3 — Create Backend Services

Create 6 backend services. Each connects to its own DB and to Redis.

For each service:
1. Railway → Project → **+ New** → **GitHub Repo**
2. Select the `promotionos` org and the relevant repo
3. Set the environment variables (see below)
4. Railway auto-deploys on every push to `main`

### campaign-service

```
DB_URL=<campaign-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
TENANT_ID=tenant-kroger-001
PORT=8081
SPRING_PROFILES_ACTIVE=railway
```

### eligibility-service

```
DB_URL=<eligibility-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
TENANT_ID=tenant-kroger-001
PORT=8082
SPRING_PROFILES_ACTIVE=railway
CAMPAIGN_SERVICE_URL=<campaign-service Railway URL>
CATALOG_SERVICE_URL=<catalog-customer-service Railway URL>
```

### redemption-service

```
DB_URL=<redemption-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
TENANT_ID=tenant-kroger-001
PORT=8083
SPRING_PROFILES_ACTIVE=railway
ELIGIBILITY_SERVICE_URL=<eligibility-service Railway URL>
CAMPAIGN_SERVICE_URL=<campaign-service Railway URL>
```

### catalog-customer-service

```
DB_URL=<catalog-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
TENANT_ID=tenant-kroger-001
PORT=8084
GIN_MODE=release
```

### analytics-service

```
DB_URL=<analytics-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
TENANT_ID=tenant-kroger-001
PORT=8085
GIN_MODE=release
CAMPAIGN_SERVICE_URL=<campaign-service Railway URL>
```

### event-store (Go/Gin)

```
DB_URL=<eventstore-db DATABASE_URL>
REDIS_URL=<promotionos-redis REDIS_URL>
PORT=8086
GIN_MODE=release
```

---

## Step 4 — DB Schema Migrations

Run migrations after services are deployed. Each service runs its own migrations on startup via Flyway (Java) or Goose (Go).

**Verify migrations ran:**
- Railway → service → **Logs** tab
- Look for: `Successfully applied N migration(s)` (Flyway) or `OK   migration_name` (Goose)

### campaign-db Schema

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    region VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    offer_type VARCHAR(50),
    offer_value DECIMAL(10,2),
    offer_threshold_amount DECIMAL(10,2),
    offer_discount_amount DECIMAL(10,2),
    upc_scope JSONB DEFAULT '[]',
    stack_permission BOOLEAN DEFAULT FALSE,
    stack_limit INTEGER DEFAULT 1,
    segment_restriction VARCHAR(50),
    geo_scope JSONB DEFAULT '["division-all"]',
    date_start DATE NOT NULL,
    date_end DATE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE fundings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    tenant_id UUID NOT NULL,
    vendor_id VARCHAR(255) NOT NULL,
    vendor_share DECIMAL(5,2) NOT NULL,
    kroger_share DECIMAL(5,2) NOT NULL,
    CONSTRAINT funding_shares_sum CHECK (vendor_share + kroger_share = 100)
);

CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    tenant_id UUID NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    burned_amount DECIMAL(12,2) DEFAULT 0,
    currency VARCHAR(10) DEFAULT 'USD',
    exhaustion_threshold DECIMAL(5,2) DEFAULT 95.0,
    budget_exhausted_emitted BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_campaigns_tenant_status ON campaigns(tenant_id, status);
CREATE INDEX idx_campaigns_tenant_dates ON campaigns(tenant_id, date_start, date_end);
```

### eligibility-db Schema

```sql
CREATE TABLE eligibility_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    stack_limit INTEGER DEFAULT 1,
    segment_restriction VARCHAR(50),
    geo_scope JSONB DEFAULT '["division-all"]',
    threshold_type VARCHAR(50),
    threshold_spend DECIMAL(10,2),
    threshold_quantity INTEGER,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE exclusions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES eligibility_rules(id),
    tenant_id UUID NOT NULL,
    category_id VARCHAR(255),
    upc_code VARCHAR(255),
    reason VARCHAR(255)
);

CREATE TABLE eligibility_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    campaign_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    eligible BOOLEAN NOT NULL,
    reason VARCHAR(100),
    cart_total DECIMAL(10,2),
    cart_upcs JSONB,
    evaluated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_eligibility_rules_tenant_campaign ON eligibility_rules(tenant_id, campaign_id);
CREATE INDEX idx_audit_log_tenant_campaign ON eligibility_audit_log(tenant_id, campaign_id);
```

### redemption-db Schema

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
    CONSTRAINT uq_idempotency UNIQUE (idempotency_key, tenant_id)
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

CREATE INDEX idx_redemptions_tenant_customer ON redemptions(tenant_id, customer_id);
CREATE INDEX idx_redemptions_tenant_campaign ON redemptions(tenant_id, campaign_id);
CREATE INDEX idx_claims_scheduled ON claims(scheduled_at) WHERE status = 'PENDING';
```

### catalog-db Schema

```sql
CREATE TABLE categories (
    id VARCHAR(255) NOT NULL,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    parent_id VARCHAR(255),
    excluded BOOLEAN DEFAULT FALSE,
    excluded_inherited_from VARCHAR(255),
    exclude_reason VARCHAR(255),
    PRIMARY KEY (id, tenant_id)
);

CREATE TABLE upcs (
    code VARCHAR(255) NOT NULL,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    category_id VARCHAR(255) NOT NULL,
    excluded BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (code, tenant_id)
);

CREATE TABLE upc_price_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    upc_code VARCHAR(255) NOT NULL,
    tenant_id UUID NOT NULL,
    previous_price DECIMAL(10,2),
    new_price DECIMAL(10,2) NOT NULL,
    effective_from TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(255),
    annual_spend DECIMAL(12,2) DEFAULT 0,
    segments JSONB DEFAULT '[]',
    division VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE purchase_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    upc_code VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL,
    spend DECIMAL(10,2) NOT NULL,
    purchased_at TIMESTAMPTZ NOT NULL,
    store_id VARCHAR(255)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_purchase_history_customer_date ON purchase_history(customer_id, purchased_at);
CREATE INDEX idx_categories_tenant_parent ON categories(tenant_id, parent_id);
```

### analytics-db Schema

```sql
CREATE TABLE campaign_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    baseline_sales_per_day DECIMAL(12,2) DEFAULT 0,
    actual_sales_per_day DECIMAL(12,2) DEFAULT 0,
    lift DECIMAL(12,2),
    lift_percentage DECIMAL(8,4),
    total_funding_cost DECIMAL(12,2) DEFAULT 0,
    incremental_margin DECIMAL(12,2),
    roi DECIMAL(8,4),
    budget_burn_percent DECIMAL(5,2) DEFAULT 0,
    burned_amount DECIMAL(12,2) DEFAULT 0,
    total_amount DECIMAL(12,2) DEFAULT 0,
    redemption_count INTEGER DEFAULT 0,
    budget_exhausted_emitted BOOLEAN DEFAULT FALSE,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT uq_campaign_tenant UNIQUE (campaign_id, tenant_id)
);

CREATE INDEX idx_metrics_tenant ON campaign_metrics(tenant_id);
```

### eventstore-db Schema

```sql
CREATE TABLE domain_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    context VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    schema_version INTEGER NOT NULL DEFAULT 1,
    payload JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    recorded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Append-only: no UPDATE, no DELETE ever allowed on this table
-- Enforce via: REVOKE UPDATE, DELETE ON domain_events FROM PUBLIC;
REVOKE UPDATE, DELETE ON domain_events FROM PUBLIC;

CREATE INDEX idx_events_tenant_context ON domain_events(tenant_id, context);
CREATE INDEX idx_events_aggregate ON domain_events(aggregate_id, aggregate_type);
CREATE INDEX idx_events_occurred ON domain_events(occurred_at);
```

---

## Step 5 — Seed Test Data

After migrations, seed the test data. Run these SQL inserts against each database.

The full test data is in the `test-data` repo (JSON format). Convert to SQL inserts before seeding, or use a seed script.

### Seed Order

Seed in this order — foreign key dependencies:

1. `catalog-db` — categories first, then UPCs, then customers
2. `campaign-db` — tenant first, then campaigns, then fundings, then budgets
3. `eligibility-db` — eligibility rules (after campaigns exist in campaign-db)
4. `analytics-db` — campaign metrics (after campaigns exist)
5. `redemption-db` — redemptions (after campaigns exist)
6. `eventstore-db` — no seed needed (append-only, populated by services)

### Special — redeem-008 Session Timestamp

When seeding `redemption-db` — set `redeem-008.redeemed_at` to the actual session start time:

```sql
INSERT INTO redemptions (id, tenant_id, idempotency_key, customer_id, campaign_id,
    discount_applied, cart_total, store_id, division, redeemed_at, status)
VALUES (
    'redeem-008-uuid',
    'tenant-kroger-001-uuid',
    'pos-store-chi-001-txn-SESSION',
    'cust-001-uuid',
    'camp-002-uuid',
    4.99,
    28.50,
    'store-chicago-001',
    'division-midwest',
    NOW(),  -- session start time — NOT a historical date
    'CONFIRMED'
);
```

This is critical for scenario 28 (claim at T+24hrs) to work during the session.

---

## Step 6 — Verify Everything

Run this verification checklist before session starts:

### Database Verification
```sql
-- Run on each DB — should return > 0
SELECT COUNT(*) FROM campaigns;          -- campaign-db
SELECT COUNT(*) FROM eligibility_rules;  -- eligibility-db
SELECT COUNT(*) FROM redemptions;        -- redemption-db
SELECT COUNT(*) FROM categories;         -- catalog-db
SELECT COUNT(*) FROM campaign_metrics;   -- analytics-db
```

### Redis Verification
```bash
# Connect to Redis and verify pub/sub works
redis-cli -u $REDIS_URL ping
# Should return: PONG
```

### Service Verification
```bash
# Each service should return 200 on health check
curl https://<campaign-service-url>/actuator/health
curl https://<eligibility-service-url>/health
curl https://<redemption-service-url>/health
curl https://<catalog-service-url>/health
curl https://<analytics-service-url>/health

# Each domain endpoint should return 501
curl https://<campaign-service-url>/campaigns?tenantId=tenant-kroger-001
# Expected: 501 Not Implemented
```

---

## Railway Environment Variables — Master Reference

Keep this table updated as Railway generates service URLs.

| Service | DB_URL | REDIS_URL | Service URL |
|---------|--------|-----------|-------------|
| campaign-service | campaign-db URL | shared redis | TBD after deploy |
| eligibility-service | eligibility-db URL | shared redis | TBD after deploy |
| redemption-service | redemption-db URL | shared redis | TBD after deploy |
| catalog-customer-service | catalog-db URL | shared redis | TBD after deploy |
| analytics-service | analytics-db URL | shared redis | TBD after deploy |
| event-store | eventstore-db URL | shared redis | TBD after deploy |

Fill in service URLs after first deploy — these are injected into dependent services as env vars.
