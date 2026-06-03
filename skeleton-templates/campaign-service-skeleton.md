# Campaign Service — Skeleton Build Template

> Language: Java 17 / Spring Boot 3.x
> Build tool: Maven
> DB: PostgreSQL via Flyway + Spring Data JPA
> Events: Redis Pub/Sub via Spring Data Redis
> Deploy: Railway (auto-deploy on push to main)
> Status: Pre-built with business logic + bug planted in BudgetTracker

---

## Repo Structure to Create

```
campaign-service/
├── src/
│   ├── main/
│   │   ├── java/com/promotionos/campaign/
│   │   │   ├── CampaignServiceApplication.java
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Campaign.java
│   │   │   │   │   ├── CampaignStatus.java
│   │   │   │   │   ├── Offer.java
│   │   │   │   │   ├── OfferType.java
│   │   │   │   │   ├── Funding.java
│   │   │   │   │   ├── Budget.java
│   │   │   │   │   ├── Money.java
│   │   │   │   │   ├── Percentage.java
│   │   │   │   │   ├── DateRange.java
│   │   │   │   │   ├── TenantId.java
│   │   │   │   │   └── UPC.java
│   │   │   │   ├── service/
│   │   │   │   │   ├── FundingValidator.java
│   │   │   │   │   ├── ConflictResolver.java
│   │   │   │   │   └── BudgetTracker.java
│   │   │   │   ├── repository/
│   │   │   │   │   └── CampaignRepository.java
│   │   │   │   └── event/
│   │   │   │       ├── CampaignPublished.java
│   │   │   │       ├── CampaignPaused.java
│   │   │   │       └── BudgetExhausted.java
│   │   │   ├── application/
│   │   │   │   └── CampaignApplicationService.java
│   │   │   ├── infrastructure/
│   │   │   │   ├── repository/
│   │   │   │   │   └── CampaignRepositoryImpl.java
│   │   │   │   └── event/
│   │   │   │       ├── EventPublisher.java
│   │   │   │       └── BudgetExhaustedConsumer.java
│   │   │   └── api/
│   │   │       └── CampaignController.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           ├── V1__campaign_schema.sql
│   │           └── V2__seed_test_data.sql
│   └── test/
│       └── java/com/promotionos/campaign/
│           └── CampaignContractTest.java
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
    <artifactId>campaign-service</artifactId>
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
  port: ${PORT:8081}

management:
  endpoints:
    web:
      exposure:
        include: health

promotionos:
  tenant-id: ${TENANT_ID:tenant-kroger-001}
  redis:
    channels:
      campaign-published: promotionos.campaign.published
      campaign-paused: promotionos.campaign.paused
      budget-exhausted-in: promotionos.analytics.budget.exhausted
      budget-updated: promotionos.campaign.budget.updated
```

---

## Domain Model

### Campaign.java (Aggregate Root)

```java
package com.promotionos.campaign.domain.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Data
@NoArgsConstructor
public class Campaign {
    private UUID id;
    private TenantId tenantId;
    private String name;
    private CampaignStatus status;
    private Offer offer;
    private Funding funding;
    private Budget budget;
    private DateRange dateRange;
    private boolean stackPermission;
    private int stackLimit;
    private String segmentRestriction;
    private List<String> geoScope = new ArrayList<>();
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public static Campaign draft(TenantId tenantId, String name, Offer offer,
                                  DateRange dateRange, boolean stackPermission,
                                  int stackLimit, String segmentRestriction,
                                  List<String> geoScope) {
        Campaign c = new Campaign();
        c.id = UUID.randomUUID();
        c.tenantId = tenantId;
        c.name = name;
        c.offer = offer;
        c.dateRange = dateRange;
        c.stackPermission = stackPermission;
        c.stackLimit = stackLimit;
        c.segmentRestriction = segmentRestriction;
        c.geoScope = geoScope;
        c.status = CampaignStatus.DRAFT;
        return c;
    }

    public void addFunding(Funding funding) {
        this.funding = funding;
    }

    public void setBudget(Budget budget) {
        this.budget = budget;
    }

    public void publish(FundingValidator fundingValidator, ConflictResolver conflictResolver,
                        List<Campaign> activeCampaigns) {
        fundingValidator.validate(this.funding);
        conflictResolver.checkUPCOverlap(this, activeCampaigns);
        this.status = CampaignStatus.ACTIVE;
        this.domainEvents.add(new CampaignPublished(this));
    }

    public void pause(String reason) {
        this.status = CampaignStatus.PAUSED;
        this.domainEvents.add(new CampaignPaused(this.id, this.tenantId, reason));
    }

    public void checkBudgetExhaustion() {
        if (this.budget == null) return;
        // BUG PLANTED HERE: uses > 94 instead of >= 95
        // This causes campaigns to pause at 94.9% instead of 95.0%
        // Teams find this in Sprint 1 via RCA
        double burnPercent = (this.budget.getBurnedAmount().getAmount().doubleValue()
                / this.budget.getTotalAmount().getAmount().doubleValue()) * 100;
        if (burnPercent > 94 && !this.budget.isBudgetExhaustedEmitted()) {
            this.budget.markExhausted();
            this.domainEvents.add(new BudgetExhausted(this.id, this.tenantId, this.budget));
        }
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }
}
```

### Value Objects

```java
// Money.java
package com.promotionos.campaign.domain.model;

import lombok.Value;
import java.math.BigDecimal;

@Value
public class Money {
    BigDecimal amount;
    String currency;

    public static Money of(BigDecimal amount) {
        return new Money(amount, "USD");
    }
}

// Percentage.java
@Value
public class Percentage {
    BigDecimal value;

    public static Percentage of(double value) {
        return new Percentage(BigDecimal.valueOf(value));
    }
}

// DateRange.java
@Value
public class DateRange {
    java.time.LocalDate startDate;
    java.time.LocalDate endDate;

    public boolean overlaps(DateRange other) {
        return !this.endDate.isBefore(other.startDate)
            && !other.endDate.isBefore(this.startDate);
    }
}

// TenantId.java
@Value
public class TenantId {
    String value;
}

// UPC.java
@Value
public class UPC {
    String code;
}
```

### Offer.java

```java
@Data
@NoArgsConstructor
public class Offer {
    private UUID id;
    private OfferType type;
    private Money value;
    private Money thresholdAmount;
    private Money discountAmount;
    private List<UPC> upcScope = new ArrayList<>();
}
```

### Funding.java

```java
@Data
@NoArgsConstructor
public class Funding {
    private UUID id;
    private String vendorId;
    private Percentage vendorShare;
    private Percentage krogerShare;
}
```

### Budget.java

```java
@Data
@NoArgsConstructor
public class Budget {
    private UUID id;
    private Money totalAmount;
    private Money burnedAmount;
    private Percentage exhaustionThreshold;
    private boolean budgetExhaustedEmitted;

    public void addBurn(Money amount) {
        this.burnedAmount = Money.of(
            this.burnedAmount.getAmount().add(amount.getAmount())
        );
    }

    public void markExhausted() {
        this.budgetExhaustedEmitted = true;
    }

    public double getBurnPercent() {
        return (burnedAmount.getAmount().doubleValue()
            / totalAmount.getAmount().doubleValue()) * 100;
    }
}
```

---

## Domain Service Interfaces

```java
// FundingValidator.java
package com.promotionos.campaign.domain.service;

public interface FundingValidator {
    void validate(Funding funding); // throws DomainException if invalid
}

// ConflictResolver.java
public interface ConflictResolver {
    void checkUPCOverlap(Campaign campaign, List<Campaign> activeCampaigns);
}

// CampaignRepository.java
public interface CampaignRepository {
    Campaign findById(UUID id, TenantId tenantId);
    List<Campaign> findByStatus(CampaignStatus status, TenantId tenantId);
    List<Campaign> findByUPC(UPC upc, TenantId tenantId);
    Campaign save(Campaign campaign);
}
```

---

## Domain Events

```java
// CampaignPublished.java
public record CampaignPublished(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(
        String offerId,
        String offerType,
        double offerValue,
        List<String> upcScope,
        String dateStart,
        String dateEnd,
        boolean stackPermission,
        int stackLimit,
        String segmentRestriction,
        List<String> geoScope,
        List<String> exclusions
    ) {}
}

// CampaignPaused.java
public record CampaignPaused(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(String reason, String pausedAt) {}
}

// BudgetExhausted.java
public record BudgetExhausted(
    String eventId,
    String tenantId,
    String campaignId,
    String occurredAt,
    int schemaVersion,
    Payload payload
) implements DomainEvent {
    public record Payload(
        double totalAmount,
        double burnedAmount,
        double budgetBurnPercent,
        int redemptionCount,
        String exhaustedAt
    ) {}
}
```

---

## Application Service

```java
// CampaignApplicationService.java
@Service
@RequiredArgsConstructor
public class CampaignApplicationService {

    private final CampaignRepository campaignRepository;
    private final FundingValidator fundingValidator;
    private final ConflictResolver conflictResolver;
    private final EventPublisher eventPublisher;

    public Campaign createDraft(CreateCampaignCommand cmd) {
        Campaign campaign = Campaign.draft(
            new TenantId(cmd.tenantId()),
            cmd.name(),
            cmd.offer(),
            cmd.dateRange(),
            cmd.stackPermission(),
            cmd.stackLimit(),
            cmd.segmentRestriction(),
            cmd.geoScope()
        );
        Campaign saved = campaignRepository.save(campaign);
        return saved;
    }

    public Campaign addFunding(UUID campaignId, TenantId tenantId, Funding funding) {
        Campaign campaign = campaignRepository.findById(campaignId, tenantId);
        campaign.addFunding(funding);
        return campaignRepository.save(campaign);
    }

    public Campaign publish(UUID campaignId, TenantId tenantId) {
        Campaign campaign = campaignRepository.findById(campaignId, tenantId);
        List<Campaign> active = campaignRepository.findByStatus(CampaignStatus.ACTIVE, tenantId);
        campaign.publish(fundingValidator, conflictResolver, active);
        Campaign saved = campaignRepository.save(campaign);
        eventPublisher.publishAll(saved.pullDomainEvents());
        return saved;
    }

    public void handleBudgetExhausted(BudgetExhaustedEvent event) {
        Campaign campaign = campaignRepository.findById(
            UUID.fromString(event.campaignId()),
            new TenantId(event.tenantId())
        );
        campaign.pause("BUDGET_EXHAUSTED");
        Campaign saved = campaignRepository.save(campaign);
        eventPublisher.publishAll(saved.pullDomainEvents());
    }

    public Campaign getSummary(UUID campaignId, TenantId tenantId) {
        return campaignRepository.findById(campaignId, tenantId);
    }
}
```

---

## REST Controller

```java
// CampaignController.java
@RestController
@RequestMapping("/campaigns")
@RequiredArgsConstructor
public class CampaignController {

    private final CampaignApplicationService campaignService;

    @PostMapping
    public ResponseEntity<Campaign> create(@RequestBody CreateCampaignRequest request) {
        // TODO Team 1 Sprint 1: call campaignService.createDraft()
        throw new NotImplementedException("Campaign creation not implemented");
    }

    @PutMapping("/{id}/publish")
    public ResponseEntity<Campaign> publish(@PathVariable UUID id,
                                             @RequestParam String tenantId) {
        // TODO Team 1 Sprint 1: call campaignService.publish()
        throw new NotImplementedException("Campaign publish not implemented");
    }

    @PostMapping("/{id}/funding")
    public ResponseEntity<Campaign> addFunding(@PathVariable UUID id,
                                                @RequestBody FundingRequest request) {
        // TODO Team 1 Sprint 1: call campaignService.addFunding()
        throw new NotImplementedException("Funding addition not implemented");
    }

    @GetMapping
    public ResponseEntity<List<Campaign>> list(@RequestParam String tenantId,
                                                @RequestParam(required = false) String status) {
        // TODO Team 1 Sprint 4: implement campaign list with status filter
        throw new NotImplementedException("Campaign list not implemented");
    }

    @GetMapping("/{id}/summary")
    public ResponseEntity<CampaignSummary> summary(@PathVariable UUID id,
                                                    @RequestParam String tenantId) {
        // Exposed in Sprint 1 — needed by Redemption Service for claim deduction
        Campaign campaign = campaignService.getSummary(id, new TenantId(tenantId));
        return ResponseEntity.ok(CampaignSummary.from(campaign));
    }

    @GetMapping("/{id}/validate-funding")
    public ResponseEntity<FundingValidationResult> validateFunding(@PathVariable UUID id,
                                                                    @RequestParam String tenantId) {
        // TODO Team 1 Sprint 3
        throw new NotImplementedException("Funding validation not implemented");
    }

    @PutMapping("/{id}/budget")
    public ResponseEntity<Campaign> updateBudget(@PathVariable UUID id,
                                                  @RequestBody BudgetUpdateRequest request) {
        // TODO Team 1 Sprint 3
        throw new NotImplementedException("Budget update not implemented");
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

    @Value("${promotionos.redis.channels.campaign-published}")
    private String campaignPublishedChannel;

    @Value("${promotionos.redis.channels.campaign-paused}")
    private String campaignPausedChannel;

    public void publishAll(List<DomainEvent> events) {
        events.forEach(this::publish);
    }

    private void publish(DomainEvent event) {
        try {
            String channel = resolveChannel(event);
            String payload = objectMapper.writeValueAsString(event);
            redisTemplate.convertAndSend(channel, payload);
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event: " + event.getClass().getSimpleName(), e);
        }
    }

    private String resolveChannel(DomainEvent event) {
        if (event instanceof CampaignPublished) return campaignPublishedChannel;
        if (event instanceof CampaignPaused) return campaignPausedChannel;
        throw new IllegalArgumentException("Unknown event type: " + event.getClass());
    }
}
```

---

## Infrastructure — BudgetExhausted Consumer

```java
@Component
@RequiredArgsConstructor
public class BudgetExhaustedConsumer implements MessageListener {

    private final CampaignApplicationService campaignService;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            BudgetExhaustedEvent event = objectMapper.readValue(
                message.getBody(), BudgetExhaustedEvent.class
            );
            campaignService.handleBudgetExhausted(event);
        } catch (Exception e) {
            // log and continue — do not crash the consumer
        }
    }
}
```

---

## Contract Test (Failing Until Teams Implement)

```java
// CampaignContractTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CampaignContractTest {

    @Test
    void publishWithNoFunding_returns400() {
        // Scenario 9 — must return 400 NO_FUNDING_SOURCE
        // TODO: implement this test after Team 1 Sprint 1
    }

    @Test
    void publishWithUPCOverlap_returns409() {
        // Scenario 10 — must return 409 UPC_OVERLAP
        // TODO: implement after Team 1 Sprint 2
    }

    @Test
    void budgetAt95Percent_emitsCampaignPaused() {
        // Scenario 8/26 — must emit CampaignPaused at exactly 95.0%
        // TODO: implement after Team 1 Sprint 1 bug fix
    }

    @Test
    void getSummary_returnsFundingVendorShare() {
        // Unblocks Team 3 Sprint 2 claim deduction
        // Must return vendorShare in response
    }
}
```

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
java -jar target/campaign-service-*.jar
```

---

## README.md

```markdown
# Campaign Service

Bounded context: Campaign
Language: Java 17 / Spring Boot 3.x
Status: Pre-built with bug planted in BudgetTracker.checkBudgetExhaustion()

## Documentation
- SDL: [link]
- Ubiquitous Language: [link]
- Architecture: [link]
- Contract (Campaign → Eligibility): [link]
- Contract (Analytics → Campaign): [link]
- Codebase Skill: [link]

## Sprint Requirements
Handed out by facilitator at sprint start.

## Local Development
\`\`\`bash
export DB_URL=postgresql://localhost:5432/campaign-db
export REDIS_URL=redis://localhost:6379
mvn spring-boot:run
\`\`\`
```
