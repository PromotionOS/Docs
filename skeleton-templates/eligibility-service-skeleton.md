# Eligibility Service — Skeleton Build Template

> Language: Java 17 / Spring Boot 3.x
> Build tool: Maven
> DB: PostgreSQL via Flyway + Spring Data JPA
> Events: Redis Pub/Sub (consumer of CampaignPublished, CampaignPaused, CatalogItemExcluded, SegmentUpdated)
> Deploy: Railway
> Status: Skeleton only — all business logic stubbed with NotImplementedException

---

## Repo Structure to Create

```
eligibility-service/
├── src/
│   ├── main/
│   │   ├── java/com/promotionos/eligibility/
│   │   │   ├── EligibilityServiceApplication.java
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── EligibilityRule.java
│   │   │   │   │   ├── Exclusion.java
│   │   │   │   │   ├── Threshold.java
│   │   │   │   │   ├── ThresholdType.java
│   │   │   │   │   ├── EligibilityResult.java
│   │   │   │   │   ├── CustomerProfile.java
│   │   │   │   │   ├── Cart.java
│   │   │   │   │   ├── TenantId.java
│   │   │   │   │   └── UPC.java
│   │   │   │   ├── service/
│   │   │   │   │   ├── RuleEngine.java
│   │   │   │   │   └── SegmentMatcher.java
│   │   │   │   └── repository/
│   │   │   │       └── EligibilityRuleRepository.java
│   │   │   ├── application/
│   │   │   │   └── EligibilityApplicationService.java
│   │   │   ├── infrastructure/
│   │   │   │   ├── repository/
│   │   │   │   │   └── EligibilityRuleRepositoryImpl.java
│   │   │   │   ├── client/
│   │   │   │   │   ├── CatalogServiceClient.java
│   │   │   │   │   └── CampaignServiceClient.java
│   │   │   │   └── event/
│   │   │   │       ├── CampaignPublishedConsumer.java
│   │   │   │       ├── CampaignPausedConsumer.java
│   │   │   │       ├── CatalogItemExcludedConsumer.java
│   │   │   │       ├── SegmentUpdatedConsumer.java
│   │   │   │       └── ACL.java
│   │   │   └── api/
│   │   │       └── EligibilityController.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           ├── V1__eligibility_schema.sql
│   │           └── V2__seed_test_data.sql
│   └── test/
│       └── java/com/promotionos/eligibility/
│           └── EligibilityContractTest.java
├── scripts/
│   ├── test.sh
│   └── deploy.sh
└── pom.xml
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
  flyway:
    enabled: true
  data:
    redis:
      url: ${REDIS_URL}

server:
  port: ${PORT:8082}

management:
  endpoints:
    web:
      exposure:
        include: health

promotionos:
  tenant-id: ${TENANT_ID:tenant-kroger-001}
  services:
    catalog-url: ${CATALOG_SERVICE_URL:http://localhost:8084}
    campaign-url: ${CAMPAIGN_SERVICE_URL:http://localhost:8081}
  redis:
    channels:
      campaign-published: promotionos.campaign.published
      campaign-paused: promotionos.campaign.paused
      catalog-item-excluded: promotionos.catalog.item.excluded
      segment-updated: promotionos.customer.segment.updated
```

---

## Domain Model

### EligibilityRule.java (Aggregate Root)

```java
package com.promotionos.eligibility.domain.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Data
@NoArgsConstructor
public class EligibilityRule {
    private UUID id;
    private UUID campaignId;
    private TenantId tenantId;
    private int stackLimit;
    private String segmentRestriction;
    private List<String> geoScope = new ArrayList<>();
    private List<Exclusion> exclusions = new ArrayList<>();
    private Threshold threshold;
    private boolean active;
    private java.time.LocalDate dateStart;
    private java.time.LocalDate dateEnd;

    public static EligibilityRule create(UUID campaignId, TenantId tenantId) {
        EligibilityRule rule = new EligibilityRule();
        rule.id = UUID.randomUUID();
        rule.campaignId = campaignId;
        rule.tenantId = tenantId;
        rule.stackLimit = 1;
        rule.active = true;
        return rule;
    }

    public void deactivate() {
        this.active = false;
    }

    public boolean isActiveOn(java.time.LocalDate date) {
        return this.active
            && !date.isBefore(this.dateStart)
            && !date.isAfter(this.dateEnd);
    }
}
```

### Value Objects

```java
// Exclusion.java
@Value
public class Exclusion {
    String categoryId;
    String upcCode;
    String reason;
}

// Threshold.java
@Value
public class Threshold {
    ThresholdType type;
    java.math.BigDecimal spendValue;
    Integer quantityValue;
}

// EligibilityResult.java
@Value
public class EligibilityResult {
    boolean eligible;
    UUID campaignId;
    String offerType;
    java.math.BigDecimal discountApplied;
    String reason; // null if eligible

    public static EligibilityResult eligible(UUID campaignId, String offerType,
                                              java.math.BigDecimal discount) {
        return new EligibilityResult(true, campaignId, offerType, discount, null);
    }

    public static EligibilityResult ineligible(UUID campaignId, String reason) {
        return new EligibilityResult(false, campaignId, null, null, reason);
    }
}

// CustomerProfile.java
@Value
public class CustomerProfile {
    UUID customerId;
    String loyaltyTier;
    List<String> segments;
    String division;
    double annualSpend;
}

// Cart.java
@Value
public class Cart {
    double total;
    List<String> upcCodes;
}
```

---

## Domain Service Interfaces

```java
// RuleEngine.java
package com.promotionos.eligibility.domain.service;

public interface RuleEngine {
    EligibilityResult evaluate(EligibilityRule rule, CustomerProfile customer, Cart cart);
}

// SegmentMatcher.java
public interface SegmentMatcher {
    boolean matches(CustomerProfile customer, String segmentRestriction);
}

// EligibilityRuleRepository.java
public interface EligibilityRuleRepository {
    List<EligibilityRule> findActive(TenantId tenantId);
    Optional<EligibilityRule> findByCampaignId(UUID campaignId, TenantId tenantId);
    EligibilityRule save(EligibilityRule rule);
    void deactivate(UUID campaignId, TenantId tenantId);
}
```

---

## Application Service (Stubs)

```java
@Service
@RequiredArgsConstructor
public class EligibilityApplicationService {

    private final EligibilityRuleRepository ruleRepository;
    private final RuleEngine ruleEngine;
    private final SegmentMatcher segmentMatcher;
    private final CatalogServiceClient catalogClient;

    public EligibilityResult check(UUID campaignId, TenantId tenantId,
                                    UUID customerId, Cart cart) {
        // TODO Team 2 Sprint 1: implement segment matching + status filter
        // TODO Team 2 Sprint 2: add exclusions, threshold, geo, stack
        throw new NotImplementedException("Eligibility check not implemented");
    }

    public List<EligibilityResult> getOffers(TenantId tenantId,
                                              UUID customerId, Cart cart) {
        // TODO Team 2 Sprint 2: implement GET /offers
        throw new NotImplementedException("Get offers not implemented");
    }

    public void loadRules(CampaignPublishedEvent event) {
        // TODO Team 2 Sprint 1: implement ACL translation + rule loading
        throw new NotImplementedException("Load rules not implemented");
    }

    public void removeRules(UUID campaignId, TenantId tenantId) {
        // TODO Team 2 Sprint 1: implement rule removal on CampaignPaused
        throw new NotImplementedException("Remove rules not implemented");
    }

    public void updateExclusions(CatalogItemExcludedEvent event) {
        // TODO Team 2 Sprint 2: implement exclusion update
        throw new NotImplementedException("Update exclusions not implemented");
    }
}
```

---

## ACL Stub

```java
// ACL.java — Anti-Corruption Layer
// Translates Campaign context language into Eligibility context language
@Component
public class ACL {

    public EligibilityRule translate(CampaignPublishedEvent event) {
        // TODO Team 2 Sprint 1: translate CampaignPublished fields into EligibilityRule
        // Field mapping defined in contract-campaign-eligibility.md
        // payload.stackLimit → rule.stackLimit
        // payload.segmentRestriction → rule.segmentRestriction
        // payload.exclusions → rule.exclusions (list of Exclusion value objects)
        // payload.geoScope → rule.geoScope
        // payload.dateStart/dateEnd → rule.dateStart/dateEnd
        throw new NotImplementedException("ACL translation not implemented");
    }
}
```

---

## REST Controller

```java
@RestController
@RequiredArgsConstructor
public class EligibilityController {

    private final EligibilityApplicationService eligibilityService;

    @PostMapping("/eligibility/check")
    public ResponseEntity<EligibilityResult> check(@RequestBody EligibilityCheckRequest request) {
        // TODO Team 2 Sprint 1
        throw new NotImplementedException("Eligibility check not implemented");
    }

    @GetMapping("/offers")
    public ResponseEntity<OffersResponse> getOffers(
            @RequestParam String tenantId,
            @RequestParam UUID customerId,
            @RequestParam(defaultValue = "0") double cartTotal) {
        // TODO Team 2 Sprint 2
        throw new NotImplementedException("Get offers not implemented");
    }

    @GetMapping("/eligibility/audit")
    public ResponseEntity<List<AuditLogEntry>> getAudit(
            @RequestParam String campaignId,
            @RequestParam String tenantId) {
        // TODO Team 2 Sprint 3
        throw new NotImplementedException("Audit log not implemented");
    }

    @GetMapping("/health")
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("OK");
    }
}
```

---

## Event Consumers (Stubs)

```java
// CampaignPublishedConsumer.java
@Component
@RequiredArgsConstructor
public class CampaignPublishedConsumer implements MessageListener {

    private final EligibilityApplicationService eligibilityService;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            CampaignPublishedEvent event = objectMapper.readValue(
                message.getBody(), CampaignPublishedEvent.class
            );
            eligibilityService.loadRules(event);
        } catch (NotImplementedException e) {
            // expected during skeleton phase — log and continue
        } catch (Exception e) {
            // log error and continue
        }
    }
}

// CampaignPausedConsumer.java — same pattern, calls eligibilityService.removeRules()
// CatalogItemExcludedConsumer.java — same pattern, calls eligibilityService.updateExclusions()
// SegmentUpdatedConsumer.java — same pattern, invalidates customer cache
```

---

## Contract Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class EligibilityContractTest {

    @Test
    void goldCustomerRejectedFromPlatinumCampaign() {
        // Scenario 2 — must return SEGMENT_MISMATCH
    }

    @Test
    void platinumCustomerQualifiesForPlatinumCampaign() {
        // Scenario 3 — must return eligible: true
    }

    @Test
    void pausedCampaignNotServed() {
        // Scenario 11 — must not appear in offers response
    }

    @Test
    void scheduledCampaignNotServed() {
        // Scenario 12 — must not appear in offers response
    }
}
```
