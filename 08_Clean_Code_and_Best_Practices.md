# Clean Code Practices — Senior Developer Edition

## Overview
Opinionated, production-proven guidelines for **Java 17 + Spring Boot 3** projects. Covers: SOLID, packaging strategies, hexagonal/DDD, exception policy, logging strategy, validation, testing pyramid, CI/CD quality gates, code review checklist, and secure coding notes.

---

## Principles (SOLID + Pragmatism)

- **S - Single Responsibility**: A class has one reason to change. Split orchestration from domain logic.
- **O - Open/Closed**: Extend behavior via composition/strategy rather than `if/else` trees.
- **L - Liskov**: Subtypes shouldn’t weaken contracts; prefer **final classes** + composition.
- **I - Interface Segregation**: Fine-grained ports; avoid “god” interfaces.
- **D - Dependency Inversion**: Depend on **ports (interfaces)**; implement adapters for infra.

**Pragmatic rules**
- Prefer **immutability** for DTOs/configs.
- Fail **fast** at boundaries (validate inputs).
- Small functions (≤ 20–30 lines), expressive names, no magic numbers.

---

## Packaging Strategy

Prefer **package-by-feature** + **hexagonal** boundaries over package-by-layer.

```
com.example.orders
 ├─ app        (use-cases/orchestrators)
 ├─ domain     (aggregates, entities, value objects, domain services)
 ├─ port       (interfaces: outbox, payment, repo)
 ├─ adapter
 │   ├─ web    (controllers, request/response mappers)
 │   ├─ db     (JpaRepositories, entities, mappers)
 │   └─ msg    (Kafka/RabbitMQ producers/consumers)
 └─ config     (DI, properties)
```

**Benefits**: Cohesion, focused tests, easy modularity.

---

## Hexagonal Architecture (Ports & Adapters)

- **Domain** is framework-agnostic.
- **Ports** (interfaces) expose required behaviors.
- **Adapters** implement ports using HTTP, DB, MQ, etc.

```java
// port
public interface PaymentPort { PaymentResult charge(PaymentCommand cmd); }

// app service (use case)
@Service
public class CheckoutUseCase {
    private final PaymentPort payment;
    public CheckoutUseCase(PaymentPort payment) { this.payment = payment; }
    public Receipt handle(CheckoutCommand cmd) { return payment.charge(cmd.toPayment()); }
}

// adapter
@Component
public class StripePaymentAdapter implements PaymentPort { /* calls Stripe SDK */ }
```

---

## DTOs, Mappers & Validation

- Use **DTOs** for API boundaries; keep entities internal.
- Map with **MapStruct** (fast, compile-time).

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    Order toEntity(CreateOrderRequest req);
    OrderResponse toResponse(Order order);
}
```

**Validation**
- Controller layer: `@Valid` for request DTOs + constraint annotations.
- Service layer: `@Validated` on classes; annotate method parameters.
- Domain invariants: enforce in constructors/factories.

```java
public record CreateUserRequest(
    @Email @NotBlank String email,
    @Size(min = 8) String password
) {}
```

---

## Exception Handling Policy

### Rules
- **Checked exceptions** for recoverable I/O (rare in services).
- **Unchecked** for programming and business rule violations.
- Do **not** expose stack traces to clients; map to problem details.

```java
// domain
public class BusinessException extends RuntimeException {
    public BusinessException(String message) { super(message); }
}

// presentation
@RestControllerAdvice
class Errors {
    @ExceptionHandler(BusinessException.class)
    ResponseEntity<ApiError> onBusiness(BusinessException ex) {
        return ResponseEntity.unprocessableEntity()
            .body(new ApiError("BUSINESS_ERROR", ex.getMessage()));
    }
}
```

**Guidelines**
- Attach **context** (ids, inputs) to exceptions; avoid logging secrets/PII.
- Avoid catching `Exception` broadly unless to translate and rethrow.

---

## Logging Strategy

- **Structured JSON** logs; include **correlation IDs** (MDC).
- Log levels: `ERROR` (actionable failure), `WARN` (unexpected but tolerated), `INFO` (high-level flow), `DEBUG` (dev only), `TRACE` (deep).

```java
try (MDC.MDCCloseable c1 = MDC.putCloseable("requestId", reqId)) {
    log.info("Create order start orderId={}", orderId);
}
```

**Don’ts**
- Don’t log passwords, tokens, or card data (PCI/GDPR).
- Don’t log huge payloads; truncate or hash.
- Don’t use `e.printStackTrace()` — always use logger.

---

## Configuration & Secrets

- Strongly typed config with `@ConfigurationProperties + @Validated`.
- Secrets in Vault/KMS/Kubernetes Secrets; **never** in repo.
- Fail **fast** when required config missing.

```java
@ConfigurationProperties(prefix = "payment")
@Validated
public record PaymentProps(@NotNull URI endpoint, @Min(1) int timeoutSec) {}
```

---

## Testing Strategy (Pyramid)

- **Unit** (fast, isolated): domain, mappers, pure functions.
- **Slice**: `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`.
- **Integration**: Testcontainers for DB/MQ/HTTP deps.
- **E2E/Contract**: consumer-driven contracts (e.g., Pact).
- **Non-functional**: load tests (Gatling/JMeter), chaos in non-prod.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @Test void createsOrder() throws Exception {
        mvc.perform(post("/orders").contentType(APPLICATION_JSON).content("{...}"))
           .andExpect(status().isCreated());
    }
}
```

**Rules**
- Deterministic tests; no time or random without control.
- Name tests as behavior specs (`should_create_order_when_payload_valid`).
- Keep test data realistic; use builders/factories.

---

## CI/CD Quality Gates

- Static analysis: **SpotBugs**, **Checkstyle**, **Error Prone**.
- Security: **OWASP Dependency-Check** / **Snyk**.
- Coverage ≥ **80%** (mutation testing for critical logic with **PIT**) — focus on **meaningful** coverage.
- Build reproducibility; pinned versions.
- **Fail the build** on vulnerabilities, style errors, flaky tests.

```xml
<!-- example: maven-surefire + failsafe for unit/integration separation -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.2.5</version>
</plugin>
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.2.5</version>
</plugin>
```

---

## Code Review Checklist (Senior)

**Design**
- Is module **cohesive** and following hexagonal boundaries?
- Appropriate visibility (`public` only where necessary)?
- Clear separation between orchestration and domain logic?

**Correctness**
- Are edge-cases and failure modes covered (nulls, timezones, overflow)?
- Transactions & isolation levels correct? Idempotency where needed?

**Performance**
- Any N+1 risk? Proper pagination? Appropriate caches/TTLs?
- Proper thread pools/timeouts for I/O?

**Security**
- No secrets in logs; input validation present; output encoding?
- Authorization checks enforced; least-privilege for tokens/keys.

**Quality**
- Tests readable and deterministic? Code duplication minimized?
- Naming clear; comments only where non-obvious; docs updated.

---

## Naming & Readability

- Express **intent**: `calculateTax()` vs `doWork()`.
- Avoid abbreviations; prefer domain terms.
- Small, focused methods; return early instead of deep nesting.
- Keep parameter lists short; use records/builders for many params.

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
}
```

---

## Concurrency & Time

- Avoid shared mutable state; prefer immutable DTOs and local variables.
- Use `java.time` (`Instant`, `OffsetDateTime`, `ZoneId`) — **no** legacy `Date`/`Calendar`.
- Inject `Clock` to make time testable.

```java
@Service
public class TokenService {
    private final Clock clock;
    public TokenService(Clock clock) { this.clock = clock; }
    public Instant expiresAt(Duration ttl) { return Instant.now(clock).plus(ttl); }
}
```

---

## Documentation

- Keep concise **README** with run, test, and deploy instructions.
- ADRs (Architecture Decision Records) for significant decisions.
- Generate OpenAPI and link to usage examples.

---

## Performance Hygiene

- Avoid premature optimization; measure with **JMH/JFR**.
- Watch GC allocations; prefer primitives; reuse codecs/formatters.
- Avoid reflection on hot paths; cache mappers/clients.

---

## Security Notes (Brief)

- Validate all inputs; encode outputs (XSS).
- Use CSRF protection where relevant.
- Use short-lived JWTs; rotate refresh tokens; validate audience/issuer.
- Enforce HTTPS; HSTS; secure cookies.
- Encrypt PII at rest; minimize data retention.

---

## Quick Templates

**Controller Skeleton**
```java
@RestController
@RequestMapping("/orders")
class OrderController {
    private final CheckoutUseCase useCase;
    private final OrderMapper mapper;
    OrderController(CheckoutUseCase useCase, OrderMapper mapper) { this.useCase = useCase; this.mapper = mapper; }

    @PostMapping
    ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        var res = useCase.handle(mapper.toEntity(req));
        return ResponseEntity.status(HttpStatus.CREATED).body(mapper.toResponse(res));
    }
}
```

**Service with Transaction Boundary**
```java
@Service
public class OrderService {
    @Transactional
    public Order create(CreateOrderCommand cmd) {
        // validate, persist, emit outbox event
    }
}
```

**Global Error Contract**
```java
public record ApiError(String code, String message) {}
```

---

## Final Checklist

- Package-by-feature; hexagonal boundaries.
- DTOs at edges; MapStruct mappers; domain invariants enforced.
- Centralized exception handling; structured logging with correlation IDs.
- Validation at controller/service; no PII in logs.
- Tests across pyramid; integration via Testcontainers.
- Quality gates in CI; security scanning; reproducible builds.

---

