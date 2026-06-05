# Vendor Service — Skeleton Build Template

> Language: Java 17 / Spring Boot 3.x
> Build tool: Maven
> DB: PostgreSQL via Flyway + Spring Data JPA
> Events: Redis Pub/Sub via Spring Data Redis
> Deploy: Railway (auto-deploy on push to main)
> Status: Pure skeleton — all application service methods throw NotImplementedException

---

## Repo Structure

```
vendor-service/
├── src/
│   ├── main/
│   │   ├── java/com/promotionos/vendor/
│   │   │   ├── VendorServiceApplication.java
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Vendor.java
│   │   │   │   │   ├── VendorStatus.java
│   │   │   │   │   ├── FundingProposal.java
│   │   │   │   │   ├── ProposalStatus.java
│   │   │   │   │   ├── Approval.java
│   │   │   │   │   ├── ApprovalDecision.java
│   │   │   │   │   ├── Forecast.java
│   │   │   │   │   └── ApprovalThreshold.java
│   │   │   │   ├── service/
│   │   │   │   │   ├── FundingProposalValidator.java
│   │   │   │   │   ├── ForecastCalculator.java
│   │   │   │   │   └── ApprovalGateway.java
│   │   │   │   ├── repository/
│   │   │   │   │   ├── VendorRepository.java
│   │   │   │   │   ├── FundingProposalRepository.java
│   │   │   │   │   ├── ApprovalRepository.java
│   │   │   │   │   └── ForecastRepository.java
│   │   │   │   └── event/
│   │   │   │       ├── FundingProposalSubmitted.java
│   │   │   │       ├── FundingApproved.java
│   │   │   │       └── FundingRejected.java
│   │   │   ├── application/
│   │   │   │   └── VendorApplicationService.java
│   │   │   ├── infrastructure/
│   │   │   │   ├── repository/
│   │   │   │   │   ├── VendorRepositoryImpl.java
│   │   │   │   │   ├── FundingProposalRepositoryImpl.java
│   │   │   │   │   ├── ApprovalRepositoryImpl.java
│   │   │   │   │   └── ForecastRepositoryImpl.java
│   │   │   │   └── event/
│   │   │   │       ├── EventPublisher.java
│   │   │   │       ├── CampaignPublishedConsumer.java
│   │   │   │       ├── ClaimSubmittedConsumer.java
│   │   │   │       └── BudgetExhaustedConsumer.java
│   │   │   └── api/
│   │   │       └── VendorController.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           ├── V1__vendor_schema.sql
│   │           └── V2__seed_test_data.sql
│   └── test/
│       └── java/com/promotionos/vendor/
│           └── VendorContractTest.java
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── pom.xml
```

---

## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    <groupId>com.promotionos</groupId>
    <artifactId>vendor-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## application.yml

```yaml
spring:
  datasource:
    url: ${DB_URL}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration
  data:
    redis:
      url: ${REDIS_URL}

server:
  port: ${PORT:8086}

management:
  endpoints:
    web:
      exposure:
        include: health

promotionos:
  tenant-id: ${TENANT_ID:tenant-kroger-001}
  vendor:
    approval-threshold-usd: 50000
  redis:
    channels:
      # Published by this service
      funding-proposal-submitted: promotionos.vendor.proposal.submitted
      funding-approved:           promotionos.vendor.funding.approved
      funding-rejected:           promotionos.vendor.funding.rejected
      # Consumed by this service
      campaign-published-in:      promotionos.campaign.published
      claim-submitted-in:         promotionos.claim.submitted
      budget-exhausted-in:        promotionos.analytics.budget.exhausted
```

---

## Domain Model

### model/Vendor.java (Aggregate Root)

```java
package com.promotionos.vendor.domain.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.Instant;
import java.util.UUID;

@Data
@NoArgsConstructor
public class Vendor {
    private UUID vendorId;
    private String tenantId;
    private String name;
    private String contactEmail;
    private VendorStatus status;
    private Instant createdAt;

    public static Vendor register(String tenantId, String name, String contactEmail) {
        Vendor v = new Vendor();
        v.vendorId = UUID.randomUUID();
        v.tenantId = tenantId;
        v.name = name;
        v.contactEmail = contactEmail;
        v.status = VendorStatus.ACTIVE;
        v.createdAt = Instant.now();
        return v;
    }

    public void suspend() {
        this.status = VendorStatus.SUSPENDED;
    }

    public void deactivate() {
        this.status = VendorStatus.INACTIVE;
    }
}
```

### model/VendorStatus.java

```java
package com.promotionos.vendor.domain.model;

public enum VendorStatus {
    ACTIVE,
    INACTIVE,
    SUSPENDED
}
```

### model/FundingProposal.java

```java
package com.promotionos.vendor.domain.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

// FundingProposal represents the vendor's offer to co-fund a campaign.
// It is created when an MX team member attempts to publish a campaign
// with totalAmount >= ApprovalThreshold ($50,000).
@Data
@NoArgsConstructor
public class FundingProposal {
    private UUID id;
    private UUID campaignId;
    private UUID vendorId;
    private String tenantId;
    private BigDecimal vendorShare;   // percentage, e.g. 60.0
    private BigDecimal krogerShare;   // percentage, e.g. 40.0
    private ProposalStatus status;
    private String rejectionReason;
    private Instant submittedAt;
    private Instant decidedAt;

    public static FundingProposal draft(UUID campaignId, UUID vendorId, String tenantId,
                                        BigDecimal vendorShare, BigDecimal krogerShare) {
        FundingProposal p = new FundingProposal();
        p.id = UUID.randomUUID();
        p.campaignId = campaignId;
        p.vendorId = vendorId;
        p.tenantId = tenantId;
        p.vendorShare = vendorShare;
        p.krogerShare = krogerShare;
        p.status = ProposalStatus.DRAFT;
        return p;
    }

    public void submit() {
        this.status = ProposalStatus.SUBMITTED;
        this.submittedAt = Instant.now();
    }

    public void approve() {
        this.status = ProposalStatus.APPROVED;
        this.decidedAt = Instant.now();
    }

    public void reject(String reason) {
        this.status = ProposalStatus.REJECTED;
        this.rejectionReason = reason;
        this.decidedAt = Instant.now();
    }
}
```

### model/ProposalStatus.java

```java
package com.promotionos.vendor.domain.model;

public enum ProposalStatus {
    DRAFT,
    SUBMITTED,
    APPROVED,
    REJECTED
}
```

### model/Approval.java

```java
package com.promotionos.vendor.domain.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.Instant;
import java.util.UUID;

// Approval records a single approver's decision on a FundingProposal.
@Data
@NoArgsConstructor
public class Approval {
    private UUID id;
    private UUID proposalId;
    private String tenantId;
    private String approverRole;    // e.g. "FINANCE_DIRECTOR"
    private ApprovalDecision decision;
    private String reason;
    private Instant decidedAt;
}
```

### model/ApprovalDecision.java

```java
package com.promotionos.vendor.domain.model;

public enum ApprovalDecision {
    APPROVED,
    REJECTED
}
```

### model/Forecast.java (Value Object)

```java
package com.promotionos.vendor.domain.model;

import lombok.Value;
import java.time.LocalDate;
import java.util.UUID;

// Forecast is a value object produced by ForecastCalculator.
// It is immutable — recalculate rather than mutate.
@Value
public class Forecast {
    UUID campaignId;
    double predictedLift;           // absolute $ lift over baseline
    double predictedLiftPct;        // percentage lift over baseline
    double burnVelocityPerDay;      // $ burned per day at current redemption rate
    LocalDate estimatedExhaustionDate;
    double confidence;              // 0.0–1.0
}
```

### model/ApprovalThreshold.java (Value Object)

```java
package com.promotionos.vendor.domain.model;

import lombok.Value;
import java.math.BigDecimal;

// ApprovalThreshold defines the minimum totalAmount that requires vendor approval
// before a Campaign can be published. Default: $50,000 USD.
@Value
public class ApprovalThreshold {
    Money amount;

    public static ApprovalThreshold defaultThreshold() {
        return new ApprovalThreshold(Money.of(new BigDecimal("50000.00")));
    }

    public boolean requires(Money campaignTotal) {
        return campaignTotal.getAmount().compareTo(this.amount.getAmount()) >= 0;
    }
}
```

### Shared Value Objects (reuse Campaign Service pattern)

```java
// Money.java — same as Campaign Service
@Value
public class Money {
    BigDecimal amount;
    String currency;

    public static Money of(BigDecimal amount) {
        return new Money(amount, "USD");
    }
}
```

---

## Domain Service Interfaces

```java
// FundingProposalValidator.java
package com.promotionos.vendor.domain.service;

public interface FundingProposalValidator {
    // Validates a proposal before submission.
    // Throws DomainException if vendorShare + krogerShare != 100,
    // or if vendor is INACTIVE/SUSPENDED.
    void validate(FundingProposal proposal, Vendor vendor);
}

// ForecastCalculator.java
public interface ForecastCalculator {
    // Calculates predicted lift from campaign benchmarks and historical data.
    // Returns a Forecast value object.
    Forecast calculate(UUID campaignId, String tenantId);
}

// ApprovalGateway.java
public interface ApprovalGateway {
    // Returns true if the campaign totalAmount meets or exceeds ApprovalThreshold.
    // Campaign Service calls this implicitly via REST before transitioning to PENDING_APPROVAL.
    boolean requiresApproval(Money campaignTotal);
}
```

---

## Repository Interfaces

```java
// VendorRepository.java
package com.promotionos.vendor.domain.repository;

public interface VendorRepository {
    Vendor findById(UUID vendorId);
    List<Vendor> findByTenantId(String tenantId);
    Vendor save(Vendor vendor);
}

// FundingProposalRepository.java
public interface FundingProposalRepository {
    FundingProposal findByCampaignId(UUID campaignId);
    List<FundingProposal> findByVendorId(UUID vendorId);
    FundingProposal save(FundingProposal proposal);
}

// ApprovalRepository.java
public interface ApprovalRepository {
    List<Approval> findByProposalId(UUID proposalId);
    Approval save(Approval approval);
}

// ForecastRepository.java
public interface ForecastRepository {
    Optional<Forecast> findByCampaignId(UUID campaignId);
    void save(UUID campaignId, Forecast forecast);
}
```

---

## Domain Events

```java
// FundingProposalSubmitted.java
public record FundingProposalSubmitted(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(
        String proposalId,
        String vendorId,
        double vendorShare,
        double krogerShare,
        String submittedAt
    ) {}
}

// FundingApproved.java
public record FundingApproved(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(
        String proposalId,
        String vendorId,
        double vendorShare,
        double krogerShare,
        String approvedAt
    ) {}
}

// FundingRejected.java
public record FundingRejected(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(
        String proposalId,
        String vendorId,
        String rejectionReason,
        String rejectedAt
    ) {}
}
```

---

## Application Service

```java
// VendorApplicationService.java
package com.promotionos.vendor.application;

import org.apache.commons.lang3.NotImplementedException;

@Service
@RequiredArgsConstructor
public class VendorApplicationService {

    private final VendorRepository vendorRepository;
    private final FundingProposalRepository proposalRepository;
    private final ApprovalRepository approvalRepository;
    private final ForecastRepository forecastRepository;
    private final FundingProposalValidator proposalValidator;
    private final ForecastCalculator forecastCalculator;
    private final ApprovalGateway approvalGateway;
    private final EventPublisher eventPublisher;

    // Sprint 1 — register a new vendor for a tenant
    public Vendor registerVendor(RegisterVendorCommand cmd) {
        // TODO Team 7 Sprint 1
        throw new NotImplementedException("registerVendor not implemented");
    }

    // Sprint 1 — vendor submits a funding proposal for a campaign
    // Triggered when Campaign Service calls POST /vendors/:id/proposals
    public FundingProposal submitFundingProposal(SubmitProposalCommand cmd) {
        // TODO Team 7 Sprint 1
        // 1. Load vendor, validate ACTIVE status
        // 2. Create FundingProposal.draft(...)
        // 3. Run proposalValidator.validate(...)
        // 4. Call proposal.submit()
        // 5. Save proposal
        // 6. Publish FundingProposalSubmitted event
        throw new NotImplementedException("submitFundingProposal not implemented");
    }

    // Sprint 2 — MX finance director approves a pending proposal
    public FundingProposal approveProposal(UUID proposalId, String tenantId) {
        // TODO Team 7 Sprint 2
        // 1. Load proposal, assert SUBMITTED
        // 2. Call proposal.approve()
        // 3. Create Approval record
        // 4. Save both
        // 5. Publish FundingApproved event → Campaign Service unblocks
        throw new NotImplementedException("approveProposal not implemented");
    }

    // Sprint 2 — MX finance director rejects a pending proposal
    public FundingProposal rejectProposal(UUID proposalId, String reason, String tenantId) {
        // TODO Team 7 Sprint 2
        // 1. Load proposal, assert SUBMITTED
        // 2. Call proposal.reject(reason)
        // 3. Create Approval record with REJECTED decision
        // 4. Save both
        // 5. Publish FundingRejected event → Campaign Service keeps DRAFT
        throw new NotImplementedException("rejectProposal not implemented");
    }

    // Sprint 3 — return forecast for a campaign (reads cached or recalculates)
    public Forecast getForecast(UUID campaignId, String tenantId) {
        // TODO Team 7 Sprint 3
        throw new NotImplementedException("getForecast not implemented");
    }

    // Sprint 3 — recalculate burn velocity from current redemption data
    public double calculateBurnVelocity(UUID campaignId, String tenantId) {
        // TODO Team 7 Sprint 3
        throw new NotImplementedException("calculateBurnVelocity not implemented");
    }

    // ---- Event handlers (consumed from Redis) ----

    // Consumed: CampaignPublished → check if approval needed, initiate proposal flow
    public void handleCampaignPublished(CampaignPublishedEvent event) {
        // TODO Team 7 Sprint 1
        // 1. Check approvalGateway.requiresApproval(totalAmount)
        // 2. If true: transition campaign status to PENDING_APPROVAL via REST callback
        //    (or accept that Campaign Service already did this — confirm with Team 1)
        // 3. Initiate proposal flow: create DRAFT proposal, notify vendor
        throw new NotImplementedException("handleCampaignPublished not implemented");
    }

    // Consumed: ClaimSubmitted → update vendor claim records
    public void handleClaimSubmitted(ClaimSubmittedEvent event) {
        // TODO Team 7 Sprint 3
        throw new NotImplementedException("handleClaimSubmitted not implemented");
    }

    // Consumed: BudgetExhausted → update vendor campaign status
    public void handleBudgetExhausted(BudgetExhaustedEvent event) {
        // TODO Team 7 Sprint 3
        throw new NotImplementedException("handleBudgetExhausted not implemented");
    }
}
```

---

## REST Controller

```java
// VendorController.java
@RestController
@RequiredArgsConstructor
public class VendorController {

    private final VendorApplicationService vendorService;

    // POST /vendors — register a new vendor
    // TODO Team 7 Sprint 1
    @PostMapping("/vendors")
    public ResponseEntity<Vendor> registerVendor(@RequestBody RegisterVendorRequest request) {
        throw new NotImplementedException("registerVendor not implemented");
    }

    // GET /vendors/:id — vendor profile + active campaigns
    // TODO Team 7 Sprint 1
    @GetMapping("/vendors/{id}")
    public ResponseEntity<VendorProfile> getVendor(@PathVariable UUID id,
                                                    @RequestParam String tenantId) {
        throw new NotImplementedException("getVendor not implemented");
    }

    // POST /vendors/:id/proposals — submit a funding proposal
    // Called by Campaign Service when a campaign totalAmount >= $50,000
    // and MX team initiates publish. Campaign Service transitions to PENDING_APPROVAL
    // before calling this endpoint.
    // TODO Team 7 Sprint 1
    @PostMapping("/vendors/{id}/proposals")
    public ResponseEntity<FundingProposal> submitProposal(@PathVariable UUID id,
                                                           @RequestBody SubmitProposalRequest request) {
        throw new NotImplementedException("submitProposal not implemented");
    }

    // PUT /proposals/:id/approve — approve a funding proposal
    // TODO Team 7 Sprint 2
    @PutMapping("/proposals/{id}/approve")
    public ResponseEntity<FundingProposal> approveProposal(@PathVariable UUID id,
                                                            @RequestParam String tenantId) {
        throw new NotImplementedException("approveProposal not implemented");
    }

    // PUT /proposals/:id/reject — reject a funding proposal
    // TODO Team 7 Sprint 2
    @PutMapping("/proposals/{id}/reject")
    public ResponseEntity<FundingProposal> rejectProposal(@PathVariable UUID id,
                                                           @RequestBody RejectProposalRequest request,
                                                           @RequestParam String tenantId) {
        throw new NotImplementedException("rejectProposal not implemented");
    }

    // GET /forecasts?campaignId=X — get forecast for a campaign
    // TODO Team 7 Sprint 3
    @GetMapping("/forecasts")
    public ResponseEntity<Forecast> getForecast(@RequestParam UUID campaignId,
                                                @RequestParam String tenantId) {
        throw new NotImplementedException("getForecast not implemented");
    }

    // GET /health
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of("status", "ok", "service", "vendor-service"));
    }
}
```

---

## Infrastructure — Event Publisher

```java
@Component
@RequiredArgsConstructor
public class EventPublisher {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    @Value("${promotionos.redis.channels.funding-proposal-submitted}")
    private String proposalSubmittedChannel;

    @Value("${promotionos.redis.channels.funding-approved}")
    private String fundingApprovedChannel;

    @Value("${promotionos.redis.channels.funding-rejected}")
    private String fundingRejectedChannel;

    public void publish(DomainEvent event) {
        try {
            String channel = resolveChannel(event);
            String payload = objectMapper.writeValueAsString(event);
            redisTemplate.convertAndSend(channel, payload);
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event: " + event.getClass().getSimpleName(), e);
        }
    }

    private String resolveChannel(DomainEvent event) {
        if (event instanceof FundingProposalSubmitted) return proposalSubmittedChannel;
        if (event instanceof FundingApproved)          return fundingApprovedChannel;
        if (event instanceof FundingRejected)          return fundingRejectedChannel;
        throw new IllegalArgumentException("Unknown event type: " + event.getClass());
    }
}
```

---

## Infrastructure — Event Consumers

```java
// CampaignPublishedConsumer.java
@Component
@RequiredArgsConstructor
public class CampaignPublishedConsumer implements MessageListener {
    private final VendorApplicationService vendorService;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            CampaignPublishedEvent event = objectMapper.readValue(
                message.getBody(), CampaignPublishedEvent.class
            );
            vendorService.handleCampaignPublished(event);
        } catch (Exception e) {
            // log and continue — consumer must not crash
        }
    }
}

// ClaimSubmittedConsumer.java — same pattern, calls handleClaimSubmitted
// BudgetExhaustedConsumer.java — same pattern, calls handleBudgetExhausted
```

---

## DB Migrations

### V1__vendor_schema.sql

```sql
CREATE SCHEMA IF NOT EXISTS vendor;

CREATE TABLE vendor.vendors (
    vendor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       TEXT NOT NULL,
    name            TEXT NOT NULL,
    contact_email   TEXT NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('ACTIVE','INACTIVE','SUSPENDED')) DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_vendors_tenant ON vendor.vendors (tenant_id);

CREATE TABLE vendor.funding_proposals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL,
    vendor_id           UUID NOT NULL REFERENCES vendor.vendors(vendor_id),
    tenant_id           TEXT NOT NULL,
    vendor_share        NUMERIC(5,2) NOT NULL,      -- percentage e.g. 60.00
    kroger_share        NUMERIC(5,2) NOT NULL,      -- percentage e.g. 40.00
    status              TEXT NOT NULL CHECK (status IN ('DRAFT','SUBMITTED','APPROVED','REJECTED')) DEFAULT 'DRAFT',
    rejection_reason    TEXT,
    submitted_at        TIMESTAMPTZ,
    decided_at          TIMESTAMPTZ
);

CREATE INDEX idx_proposals_campaign ON vendor.funding_proposals (campaign_id);
CREATE INDEX idx_proposals_vendor   ON vendor.funding_proposals (vendor_id);

CREATE TABLE vendor.approvals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proposal_id     UUID NOT NULL REFERENCES vendor.funding_proposals(id),
    tenant_id       TEXT NOT NULL,
    approver_role   TEXT NOT NULL,
    decision        TEXT NOT NULL CHECK (decision IN ('APPROVED','REJECTED')),
    reason          TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_approvals_proposal ON vendor.approvals (proposal_id);

CREATE TABLE vendor.forecasts (
    campaign_id                 UUID PRIMARY KEY,
    predicted_lift              NUMERIC(12,2),
    predicted_lift_pct          NUMERIC(6,2),
    burn_velocity_per_day       NUMERIC(12,2),
    estimated_exhaustion_date   DATE,
    confidence                  NUMERIC(4,3),       -- 0.000 to 1.000
    calculated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### V2__seed_test_data.sql

```sql
-- Seed vendor for tenant-kroger-001 integration tests
INSERT INTO vendor.vendors (vendor_id, tenant_id, name, contact_email, status) VALUES
    ('a1b2c3d4-0000-0000-0000-000000000001', 'tenant-kroger-001', 'Frito-Lay', 'vendor@fritolay.com', 'ACTIVE'),
    ('a1b2c3d4-0000-0000-0000-000000000002', 'tenant-kroger-001', 'Coca-Cola', 'vendor@cocacola.com', 'ACTIVE');
```

---

## Contract Note — Campaign Service Integration

> The Vendor Service introduces a new Campaign status: `PENDING_APPROVAL`.
> Campaign Service (Team 1) must add this value to `CampaignStatus` before Sprint 2.

**Flow when MX team publishes a high-value campaign:**

1. Campaign Service receives `PUT /campaigns/:id/publish`
2. Campaign Service calls `ApprovalGateway.requiresApproval(totalAmount)`
3. If true → Campaign Service transitions status to `PENDING_APPROVAL` (does NOT emit `CampaignPublished`)
4. Campaign Service calls `POST /vendors/:vendorId/proposals` on Vendor Service
5. Vendor Service creates `FundingProposal`, publishes `FundingProposalSubmitted`
6. Finance director calls `PUT /proposals/:id/approve`
7. Vendor Service publishes `FundingApproved`
8. Campaign Service consumes `FundingApproved` → transitions `PENDING_APPROVAL → ACTIVE` → emits `CampaignPublished`

See `contract-vendor-campaign.md` for full event schemas.

---

## scripts/test.sh

```bash
#!/bin/bash
set -e
mvn test -Dspring.profiles.active=test
```

## scripts/deploy.sh

```bash
#!/bin/bash
set -e
mvn clean package -DskipTests
java -jar target/vendor-service-*.jar
```

---

## .github/workflows/test.yml

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: vendor-db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}

      - name: Run tests
        env:
          DB_URL: jdbc:postgresql://localhost:5432/vendor-db
          REDIS_URL: redis://localhost:6379
        run: ./scripts/test.sh
```

---

## README.md

```markdown
# Vendor Service

Bounded context: Vendor
Language: Java 17 / Spring Boot 3.x
Status: Pure skeleton — all application service methods throw NotImplementedException

## Documentation
- SDL: [link]
- Ubiquitous Language: [link]
- Architecture: [link]
- Contract (Vendor → Campaign): contract-vendor-campaign.md

## Sprint Requirements
Handed out by facilitator at sprint start.

## Local Development
\`\`\`bash
export DB_URL=jdbc:postgresql://localhost:5432/vendor-db
export REDIS_URL=redis://localhost:6379
mvn spring-boot:run
\`\`\`
```
