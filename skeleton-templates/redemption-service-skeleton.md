# Redemption Service — Skeleton Build Template

> Language: Java 17 / Spring Boot 3.x
> Build tool: Maven
> DB: PostgreSQL via Flyway + Spring Data JPA
> Events: Redis Pub/Sub (publishes OfferRedeemed, ClaimSubmitted)
> Deploy: Railway
> Status: Skeleton only — all business logic stubbed

---

## Repo Structure

```
redemption-service/
├── src/
│   ├── main/
│   │   ├── java/com/promotionos/redemption/
│   │   │   ├── RedemptionServiceApplication.java
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Redemption.java
│   │   │   │   │   ├── RedemptionStatus.java
│   │   │   │   │   ├── Claim.java
│   │   │   │   │   ├── ClaimStatus.java
│   │   │   │   │   ├── IdempotencyKey.java
│   │   │   │   │   ├── CustomerId.java
│   │   │   │   │   ├── Money.java
│   │   │   │   │   └── TenantId.java
│   │   │   │   ├── service/
│   │   │   │   │   ├── IdempotencyGuard.java
│   │   │   │   │   └── ClaimGenerator.java
│   │   │   │   └── repository/
│   │   │   │       └── RedemptionRepository.java
│   │   │   ├── application/
│   │   │   │   └── RedemptionApplicationService.java
│   │   │   ├── infrastructure/
│   │   │   │   ├── repository/
│   │   │   │   │   └── RedemptionRepositoryImpl.java
│   │   │   │   ├── client/
│   │   │   │   │   ├── EligibilityServiceClient.java
│   │   │   │   │   └── CampaignServiceClient.java
│   │   │   │   └── event/
│   │   │   │       └── EventPublisher.java
│   │   │   └── api/
│   │   │       └── RedemptionController.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           ├── V1__redemption_schema.sql
│   │           └── V2__seed_test_data.sql
│   └── test/
│       └── java/com/promotionos/redemption/
│           └── RedemptionContractTest.java
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
  port: ${PORT:8083}

promotionos:
  tenant-id: ${TENANT_ID:tenant-kroger-001}
  services:
    eligibility-url: ${ELIGIBILITY_SERVICE_URL:http://localhost:8082}
    campaign-url: ${CAMPAIGN_SERVICE_URL:http://localhost:8081}
  redis:
    channels:
      offer-redeemed: promotionos.redemption.redeemed
      claim-submitted: promotionos.redemption.claim.submitted
```

---

## Domain Model

### Redemption.java (Aggregate Root)

```java
@Data
@NoArgsConstructor
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
    private java.time.Instant redeemedAt;
    private RedemptionStatus status;
    private Claim claim;
    private List<DomainEvent> domainEvents = new ArrayList<>();

    // Immutable after confirm() — no public setters on core fields
    public static Redemption confirm(TenantId tenantId, IdempotencyKey key,
                                     CustomerId customerId, UUID campaignId,
                                     Money discount, Money cartTotal,
                                     String storeId, String division) {
        Redemption r = new Redemption();
        r.id = UUID.randomUUID();
        r.tenantId = tenantId;
        r.idempotencyKey = key;
        r.customerId = customerId;
        r.campaignId = campaignId;
        r.discountApplied = discount;
        r.cartTotal = cartTotal;
        r.storeId = storeId;
        r.division = division;
        r.redeemedAt = java.time.Instant.now(); // always server-side
        r.status = RedemptionStatus.CONFIRMED;
        r.domainEvents.add(new OfferRedeemed(r));
        return r;
    }

    public void attachClaim(Claim claim) {
        this.claim = claim;
        this.status = RedemptionStatus.CLAIMED;
        this.domainEvents.add(new ClaimSubmitted(claim));
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
// IdempotencyKey.java
@Value
public class IdempotencyKey {
    String key;
    // Format: pos-{storeId}-txn-{transactionId}
}

// CustomerId.java
@Value
public class CustomerId {
    UUID value;
}

// Money.java
@Value
public class Money {
    java.math.BigDecimal amount;
    String currency;

    public static Money of(double amount) {
        return new Money(java.math.BigDecimal.valueOf(amount), "USD");
    }

    public Money multiply(double factor) {
        return new Money(this.amount.multiply(java.math.BigDecimal.valueOf(factor)), this.currency);
    }
}

// Claim.java (Entity)
@Data
@NoArgsConstructor
public class Claim {
    private UUID id;
    private UUID redemptionId;
    private TenantId tenantId;
    private String vendorId;
    private Money amount;
    private Money deduction;
    private ClaimStatus status;
    private java.time.Instant scheduledAt;
    private java.time.Instant submittedAt;
}
```

---

## Domain Service Interfaces

```java
// IdempotencyGuard.java
public interface IdempotencyGuard {
    boolean isDuplicate(IdempotencyKey key, TenantId tenantId);
    void register(IdempotencyKey key, TenantId tenantId, UUID redemptionId);
    UUID getOriginalRedemptionId(IdempotencyKey key, TenantId tenantId);
}

// ClaimGenerator.java
public interface ClaimGenerator {
    Claim generate(Redemption redemption, double vendorShare, String vendorId);
    void scheduleT24(UUID redemptionId);
}

// RedemptionRepository.java
public interface RedemptionRepository {
    Redemption save(Redemption redemption);
    Optional<Redemption> findById(UUID id, TenantId tenantId);
    List<Redemption> findByCustomer(UUID customerId, TenantId tenantId, int page, int size);
    List<Redemption> findByCampaign(UUID campaignId, TenantId tenantId, int page, int size);
    List<Redemption> findPendingClaims(java.time.Instant before);
}
```

---

## Application Service (Stubs)

```java
@Service
@RequiredArgsConstructor
public class RedemptionApplicationService {

    private final RedemptionRepository redemptionRepository;
    private final IdempotencyGuard idempotencyGuard;
    private final ClaimGenerator claimGenerator;
    private final EligibilityServiceClient eligibilityClient;
    private final CampaignServiceClient campaignClient;
    private final EventPublisher eventPublisher;

    public Redemption redeem(RedeemCommand cmd) {
        // TODO Team 3 Sprint 1 — implement in this order:
        // 1. idempotencyGuard.isDuplicate() — if true return 409
        // 2. eligibilityClient.check() — if ineligible return 400
        // 3. Redemption.confirm()
        // 4. idempotencyGuard.register()
        // 5. redemptionRepository.save()
        // 6. eventPublisher.publishAll(redemption.pullDomainEvents())
        throw new NotImplementedException("Redemption not implemented");
    }

    public Optional<Redemption> findById(UUID id, TenantId tenantId) {
        // TODO Team 3 Sprint 2
        throw new NotImplementedException("Find by id not implemented");
    }

    public List<Redemption> findByCustomer(UUID customerId, TenantId tenantId, int page) {
        // TODO Team 3 Sprint 3
        throw new NotImplementedException("Find by customer not implemented");
    }

    public void processPendingClaims() {
        // TODO Team 3 Sprint 2 — scheduled job, runs every minute
        // finds redemptions where scheduledAt <= now and no claim exists
        // generates claim, saves, publishes ClaimSubmitted
        throw new NotImplementedException("Claim processing not implemented");
    }
}
```

---

## Infrastructure Clients

```java
// EligibilityServiceClient.java
@Component
@RequiredArgsConstructor
public class EligibilityServiceClient {

    private final RestTemplate restTemplate;

    @Value("${promotionos.services.eligibility-url}")
    private String eligibilityUrl;

    public EligibilityResult check(UUID campaignId, String tenantId,
                                    UUID customerId, double cartTotal,
                                    List<String> cartUPCs) {
        // POST /eligibility/check
        // Returns EligibilityResult or throws if service unavailable
        EligibilityCheckRequest request = new EligibilityCheckRequest(
            tenantId, customerId, campaignId, cartTotal, cartUPCs
        );
        return restTemplate.postForObject(
            eligibilityUrl + "/eligibility/check",
            request,
            EligibilityResult.class
        );
    }
}

// CampaignServiceClient.java
@Component
public class CampaignServiceClient {

    @Value("${promotionos.services.campaign-url}")
    private String campaignUrl;

    public CampaignSummary getSummary(UUID campaignId, String tenantId) {
        // GET /campaigns/:id/summary
        return restTemplate.getForObject(
            campaignUrl + "/campaigns/" + campaignId + "/summary?tenantId=" + tenantId,
            CampaignSummary.class
        );
    }
}
```

---

## REST Controller

```java
@RestController
@RequiredArgsConstructor
public class RedemptionController {

    private final RedemptionApplicationService redemptionService;

    @PostMapping("/redeem")
    public ResponseEntity<?> redeem(@RequestBody RedeemRequest request) {
        // TODO Team 3 Sprint 1
        throw new NotImplementedException("Redemption not implemented");
    }

    @GetMapping("/redemptions/{id}")
    public ResponseEntity<Redemption> findById(@PathVariable UUID id,
                                                @RequestParam String tenantId) {
        // TODO Team 3 Sprint 2
        throw new NotImplementedException("Find by id not implemented");
    }

    @GetMapping("/redemptions")
    public ResponseEntity<List<Redemption>> findByCustomer(
            @RequestParam UUID customerId,
            @RequestParam String tenantId,
            @RequestParam(defaultValue = "0") int page) {
        // TODO Team 3 Sprint 3
        throw new NotImplementedException("Find by customer not implemented");
    }

    @GetMapping("/redemptions/stats")
    public ResponseEntity<RedemptionStats> stats(@RequestParam UUID campaignId,
                                                  @RequestParam String tenantId) {
        // TODO Team 3 Sprint 3
        throw new NotImplementedException("Stats not implemented");
    }

    @GetMapping("/health")
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("OK");
    }
}
```

---

## Contract Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RedemptionContractTest {

    @Test
    void duplicateIdempotencyKey_returns409() {
        // Scenario 7
    }

    @Test
    void sameKeyDifferentTenant_returns200() {
        // Scenario 17
    }

    @Test
    void validRedemption_publishesOfferRedeemedEvent() {
        // Must match contract-redemption-analytics.md schema exactly
    }

    @Test
    void claimDeduction_calculatedFromVendorShare() {
        // Scenario 29 — deduction = discountApplied * vendorShare%
    }
}
```
