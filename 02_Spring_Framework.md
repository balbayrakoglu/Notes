# Spring Framework — Senior Developer Edition

## Overview
This section covers **Spring Framework (Core, Context, AOP, Tx, MVC)** fundamentals and advanced usage aligned with **Spring 6 / Java 17**. Focus is on **IoC/DI**, **Bean lifecycle**, **AOP proxies**, **transaction management**, **events**, and **configuration binding**, with production-grade patterns and pitfalls.

---

## Inversion of Control (IoC) & Dependency Injection (DI)

Spring manages object creation and wiring via the **IoC Container** (typically `ApplicationContext`).

### DI Styles
- **Constructor Injection (preferred)** → immutability, testability, required deps.
- **Setter Injection** → optional deps.
- **Field Injection** → *avoid* (harder to test/mock, breaks immutability).

```java
@Service
public class PaymentService {
    private final PaymentGateway gateway;
    private final FraudChecker fraudChecker; // optional

    public PaymentService(PaymentGateway gateway, @Autowired(required = false) FraudChecker fraudChecker) {
        this.gateway = gateway;
        this.fraudChecker = fraudChecker;
    }
}
```

### Bean Discovery
- **Component scanning** with stereotypes: `@Component`, `@Service`, `@Repository`, `@Controller`.
- **Java Config** with `@Configuration` + `@Bean` methods.
- `@Configuration(proxyBeanMethods = false)` disables CGLIB method interception (better startup/perf when inter-bean calls don’t rely on proxying).

```java
@Configuration(proxyBeanMethods = false)
@ComponentScan(basePackages = "com.example")
public class AppConfig {
    @Bean
    public Clock systemClock() { return Clock.systemUTC(); }
}
```

### Qualifying Beans
- Use `@Primary` for default beans, `@Qualifier("name")` for explicit wiring.
- `@Lazy` defers instantiation until first use (mitigate heavy deps, break cycles carefully).

```java
@Service
public class ReportService {
    public ReportService(@Qualifier("fastClient") ExternalClient client) { /*...*/ }
}
```

---

## Bean Scopes & Lifecycle

### Common Scopes
- `singleton` *(default)*, `prototype`, web scopes: `request`, `session`, `application`.

### Lifecycle Phases
1. Instantiation
2. Dependency Injection
3. Initialization (`@PostConstruct`, `InitializingBean#afterPropertiesSet()`)
4. Ready for use
5. Destruction (`@PreDestroy`, `DisposableBean#destroy()`)

```java
@Component
public class WarmupCache {
    @PostConstruct
    void warmup() { /* preload cache */ }

    @PreDestroy
    void cleanup() { /* flush/close resources */ }
}
```

**Guideline**: Avoid heavy logic in constructors; prefer `@PostConstruct`.

### Circular Dependencies
- Caused by mutual constructor injection (A→B, B→A). Strategies:
    - Refactor to break cycles (extract ports/interfaces).
    - Use setter injection for *one side* (last resort).
    - Introduce orchestrator/service that coordinates both.
    - `@Lazy` can break cycles but may hide design issues.

---

## Configuration Properties & Externalization

Prefer strongly-typed binding via `@ConfigurationProperties` over multiple `@Value` injections.

```java
@ConfigurationProperties(prefix = "payment")
public record PaymentProps(Duration timeout, BigDecimal maxAmount, URI endpoint) {}

@Configuration
@EnableConfigurationProperties(PaymentProps.class)
class PaymentConfig {

    @Bean
    PaymentClient paymentClient(PaymentProps props) {
        return new PaymentClient(props.endpoint(), props.timeout());
    }
}
```

- Supports conversion to Java types (`Duration`, `URI`, `List`, nested records).
- Validate config using JSR-380:

```java
@Validated
@ConfigurationProperties("service")
public record ServiceProps(
    @NotBlank String apiKey,
    @Min(1) int poolSize
) { }
```

---

## Aspect-Oriented Programming (AOP)

AOP modularizes cross-cutting concerns (logging, security, transactions). Spring uses **runtime proxies**:
- **JDK proxies** for interfaces.
- **CGLIB** for concrete classes (or when proxy-target-class=true).

### Core Concepts
- **Aspect**: a class with advices.
- **JoinPoint**: a point during execution (method call).
- **Pointcut**: expression selecting join points.
- **Advice**: code at a join point (`@Before`, `@AfterReturning`, `@AfterThrowing`, `@Around`).

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example..service..*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long t0 = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long elapsedMs = (System.nanoTime() - t0) / 1_000_000;
            log.info("{}.{} took {} ms", pjp.getSignature().getDeclaringTypeName(),
                                  pjp.getSignature().getName(), elapsedMs);
        }
    }
}
```

**Pitfalls**:
- **Self-invocation** bypasses proxies → advice won’t run for internal method calls.
- Private/final methods aren’t advised by proxies.
- Keep aspects lean; avoid heavy logic inside advices.

---

## Transaction Management (Spring Tx)

Supports **declarative** (`@Transactional`) and **programmatic** (`TransactionTemplate`) transactions.

### Declarative Transactions
```java
@Service
public class OrderService {
    @Transactional // default: Propagation.REQUIRED
    public void placeOrder(OrderRequest req) { /* write ops */ }
}
```

**Defaults & Rules**
- Rollback **only for unchecked exceptions** (`RuntimeException`, `Error`).  
  Use `rollbackFor = Exception.class` for checked exceptions.
- Apply `@Transactional` on **public** methods; proxies target public methods.
- Do not annotate **private** methods; no effect via proxies.
- Keep transaction boundaries **short**; avoid blocking/remote calls inside.

### Propagation
- `REQUIRED` (default) → join existing or create new.
- `REQUIRES_NEW` → suspend existing and start a new transaction.
- `NESTED` → savepoints (only if DB/driver supports it).
- `MANDATORY` → Must run existing Tx.
- `NEVER` → Must not run with existing Tx.
- `SUPPORT` →  Run with or without Tx.
- `SUPPORT_NEW` → Executes method without supporting current one.


### Isolation
- `READ_COMMITTED` (default in many RDBMS), `REPEATABLE_READ`, `SERIALIZABLE`, `READ_UNCOMMITTED`.
- Choose based on consistency vs contention requirements.
- Isolation is one of the common ACID properties: Atomicity, Consistency, Isolation, and Durability. Isolation describes how changes applied by concurrent transactions are visible to each other.

- Each isolation level prevents zero or more concurrency side effects on a transaction:
  ```  
    Dirty read: read the uncommitted change of a concurrent transaction
    Nonrepeatable read: get different value on re-read of a row if a concurrent transaction updates the same row and commits
    Phantom read: get different rows after re-execution of a range query if another transaction adds or removes some rows in the range and commits
  ```
- We can set the isolation level of a transaction by @Transactional::isolation. It has these five enumerations in Spring: DEFAULT, READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE.

### Programmatic Transactions
```java
@Service
public class BillingService {
    private final TransactionTemplate tx;

    public BillingService(PlatformTransactionManager ptm) {
        this.tx = new TransactionTemplate(ptm);
    }

    public Receipt bill(Order order) {
        return tx.execute(status -> {
            // do work; status.setRollbackOnly() if needed
            return createReceipt(order);
        });
    }
}
```

**Common Pitfalls**
- **Self-invocation**: `this.method()` won’t trigger `@Transactional`.
- Mixing read-write in same Tx can cause lock contention.
- Long transactions → deadlocks, lock timeouts; keep scope minimal.

---

## Events & ApplicationContext

Spring’s event mechanism enables decoupled communication.

```java
public record OrderPlacedEvent(UUID orderId) { }

@Service
public class OrderAppService {
    private final ApplicationEventPublisher publisher;
    public OrderAppService(ApplicationEventPublisher publisher) { this.publisher = publisher; }
    public void place(Order order) {
        // ... domain logic
        publisher.publishEvent(new OrderPlacedEvent(order.id()));
    }
}

@Component
class Notifications {
    @EventListener
    @Async // optional (requires @EnableAsync)
    public void onOrderPlaced(OrderPlacedEvent e) { /* send email */ }
}
```

- Use events for side-effects (notifications, audit) not core invariants.
- `@TransactionalEventListener(phase = AFTER_COMMIT)` to fire **after successful commit**.

---

## Validation

Use **Jakarta Bean Validation** (JSR 380/381) with Spring MVC/Service layers.

```java
public record CreateUserReq(
    @NotBlank String email,
    @Size(min = 8) String password
) { }

@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public ResponseEntity<Void> create(@Valid @RequestBody CreateUserReq req) {
        // ...
        return ResponseEntity.ok().build();
    }
}
```

- For service-layer validation: `@Validated` on classes + constraint annotations on method params.

---

## Error Handling (MVC)

Centralize REST error handling with `@ControllerAdvice` and `@ExceptionHandler`.

```java
@RestControllerAdvice
public class GlobalErrors {

    @ExceptionHandler(BusinessException.class)
    ResponseEntity<ApiError> onBusiness(BusinessException ex) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
                             .body(new ApiError("BUSINESS_ERROR", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ApiError> onValidation(MethodArgumentNotValidException ex) {
        var field = ex.getBindingResult().getFieldError();
        return ResponseEntity.badRequest()
            .body(new ApiError("VALIDATION_ERROR", field != null ? field.getDefaultMessage() : "Invalid request"));
    }
}
```

---

## Profiles & Environment Awareness

Use **profiles** to switch beans/configuration per environment.

```java
@Profile("prod")
@Configuration
class ProdConfig {
    @Bean DataSource ds() { /* prod datasource */ }
}

@Profile("dev")
@Configuration
class DevConfig {
    @Bean DataSource ds() { /* h2 or local */ }
}
```

- Prefer environment variables / config server for secrets; avoid hardcoding.

---

## Resource Management

- Use `@PreDestroy` to close pools/resources.
- Prefer **constructor injection** and let container manage lifecycle.
- For heavy beans (HTTP clients, mappers), define as singletons, reuse across services.

---

## Testing Spring Components

- **Slice tests**: `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`.
- **Context tests**: `@SpringBootTest` (use profiles, test config).
- **MockMvc** for MVC controller tests.
- Avoid hitting real external systems: mock with WireMock / Mockito.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;

    @Test
    void createsUser() throws Exception {
        mvc.perform(post("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{"email":"a@b.com","password":"12345678"}"))
            .andExpect(status().isOk());
    }
}
```

---

## Quick Checklist (Production)

- Constructor injection only; avoid field injection.
- Keep beans stateless; use prototype only for stateful, short-lived objects.
- Break circular dependencies; avoid `@Lazy` unless necessary.
- Use `@ConfigurationProperties` for typed config; validate with `@Validated`.
- AOP: beware of self-invocation and final methods.
- Transactions: public methods, short-lived, correct propagation/isolation, rollback rules.
- Events: use for side-effects; `@TransactionalEventListener` for after-commit.
- Centralize error handling; never leak stack traces to clients.
- Profiles for env parity; secrets externalized.

---