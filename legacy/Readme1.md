# 🧠 Java & Spring Master Notes — Senior Developer Edition

## Core Java
(şu anda elimizde hazır — JVM, Concurrency, Streams, Optional, Records, vs.)

## Spring Framework
(DI, Bean lifecycle, AOP, Transactions, Events, ConfigurationProperties, Circular dependency)

## Spring Boot & Microservices
(Auto-configuration, Profiles, Config Server, Circuit Breakers, CQRS, Saga, Outbox Pattern)

## Messaging (Kafka & RabbitMQ)
(Architecture, retries, DLQ, idempotency, exactly-once, schema evolution)



### Outbox — Multi-Node Concurrency (Race-Free)

- **Batch seçiminde satır kilitleme:**
  ```sql
  SELECT id, aggregate_id, payload_json
  FROM outbox
  WHERE status = 'PENDING'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 50;
  ```
  Birden fazla node aynı anda çalışsa bile **`SKIP LOCKED`** sayesinde aynı satır iki kez seçilmez.

- **Idempotent publish:** `event_id` için **unique index** ve tüketici tarafında **Inbox (processed_events)** tablosu.
- **Durum geçişi:** `PENDING → SENT/FAILED`; `retry_count` artışı; `FAILED` için ayrı **DLT/quarantine** akışı.



### Kafka — `transactional.id` ve `transactionIdPrefix` (Spring)

- **EOS (Exactly-Once Semantics)** için producer `transactional.id` benzersiz olmalı.
- Spring Kafka'da:
  ```java
  kafkaTemplate.setTransactionIdPrefix("orders-svc-");
  // her producer instance, prefix + rastgele ek ile benzersiz transactional.id üretir
  ```
- Aynı `transactional.id` ile birden fazla aktif producer **KULLANMAYIN** ( fencing ).

## Caching & Redis
(@Cacheable, TTL, eviction strategies, multi-level cache, cache stampede, serialization)



### Transaction-Aware Cache & L1 Senkronizasyonu

- **`TransactionAwareCacheManagerProxy`** ile cache yazmaları **commit sonrası** yapılır.
- **L1 (Caffeine) → L2 (Redis)**: L2 güncellendiğinde **Pub/Sub invalidation** mesajı yayınlayın:
  ```java
  redisTemplate.convertAndSend("cache:invalidate", "prices:" + sku);
  ```
  Her node mesajı alıp kendi L1 girişini **evict** eder; böylece drift olmaz.

## Data Access & Performance
(JPA, fetch strategies, batch operations, N+1, locks, projections, connection pooling)



### JPA İlişki Sahipliği — Hızlı Özet

- **Owning side**: `@JoinColumn` bulunan taraf; DB foreign key'i burada tutulur.
- **Inverse side**: `mappedBy` bulunan taraf; değişiklikler owning side üzerinden yapılır.
- **Kural**: `@ManyToMany` yerine mümkünse **join-entity** kullanın (audit/ek kolonlar için esnek).

## System Design Essentials
(CAP, scalability, load balancing, distributed tracing, fault tolerance, caching layers)



### Load Balancer Algoritmaları — Karşılaştırma

| Algoritma              | Avantaj                               | Dezavantaj                          | Kullanım Durumu                     |
|------------------------|----------------------------------------|-------------------------------------|-------------------------------------|
| Round Robin            | Basit, eşit dağılım                    | Yük farklılıklarını gözetmez        | Homojen, kısa istekler              |
| Weighted Round Robin   | Güçlü node’a daha çok trafik           | Ağırlıkların bakımı gerekli         | Heterojen node kapasiteleri         |
| Least Connections      | Yoğun node’ları atlar                  | Uzun-süren bağlantılarda sapma      | Long-lived conn (WebSocket/gRPC)    |
| Least Response Time    | Latency odaklı                         | Ölçüm gürültüsü etkilenebilir       | Dengesiz gecikmelerde               |
| Consistent Hashing     | Sticky/anahtar-bağlı dağıtım           | Hot-key riski                        | Cache sharding, session stickiness  |

## Clean Code & Best Practices
(SOLID, logging, testing, CI/CD, versioning, documentation, security, performance)

## Advanced Topics
(Reactive, Security/OAuth2, Saga orchestration, Testcontainers, Terraform, Observability)

# 🧠 Java & Spring Master Notes — Senior Developer Edition

# Core Java — Senior Developer Edition

## Overview
This section covers the fundamentals and advanced concepts of Java 17, focusing on performance, concurrency, memory management, and clean design principles expected from a senior developer.

---

## JVM Architecture
The Java Virtual Machine (JVM) provides an abstraction between compiled Java code and the operating system. It consists of the following major components:

- **Class Loader Subsystem**: Loads `.class` files into memory and performs linking (verification, preparation, resolution).
- **Runtime Data Areas**
    - **Heap**: Stores objects and class instances (Young + Old Generation).
    - **Stack**: Holds local variables and method call frames per thread.
    - **Metaspace**: Stores class metadata (replaced PermGen).
    - **PC Register**: Holds the current instruction address per thread.
    - **Native Method Stack**: Used for JNI calls.
- **Execution Engine**: Includes the interpreter and JIT compiler for optimized native code.
- **Garbage Collector (GC)**: Automatically manages object lifecycle. Modern collectors include **G1GC**, **ZGC**, and **Shenandoah**.

```bash
java -Xms512m -Xmx1024m -XX:+UseG1GC -jar app.jar
```

---

## Java Memory Model (JMM)
Defines how threads interact through memory and ensures visibility and ordering of operations using the **happens-before** relationship.

```java
volatile boolean flag = false;

public void writer() {
    flag = true; // write
}

public void reader() {
    if (flag) { // guaranteed to see updated value
        System.out.println("Visible to reader thread");
    }
}
```

- `volatile` ensures visibility but **not atomicity**.
- Use `AtomicInteger`, `synchronized`, or `Lock` for atomic operations.
- Synchronization creates a happens-before relationship between threads.

---

## Threading and Locks
- **synchronized**: Ensures mutual exclusion and visibility.
- **ReentrantLock**: Advanced locking (`tryLock`, fairness, interruptible, timed).
- **ReadWriteLock**: Improves concurrency for read-heavy workloads.
- **ThreadLocal**: Provides thread-confined variables (use with care to avoid leaks).

```java
Lock lock = new ReentrantLock(true);

public void process() {
    if (lock.tryLock()) {
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Virtual Threads (Java 21+)
Virtual threads are lightweight and reduce blocking cost in high-concurrency environments. They enable large numbers of concurrent tasks without exhausting platform threads.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println(Thread.currentThread()));
}
```

---

## CompletableFuture & Asynchronous Programming
Used for non-blocking asynchronous operations and composition of tasks.

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(this::transform)
    .thenAccept(System.out::println)
    .exceptionally(ex -> {
        log.error("Error occurred", ex);
        return null;
    });
```

- Use `thenCombine()` for merging tasks, `allOf()` for aggregation.
- Prefer custom executors to control thread pool size.

---

## Immutability
Immutable objects improve safety and thread-safety.

Rules:
- Fields are `private final`.
- No setters.
- Defensive copies for mutable fields.
- Class declared `final`.

```java
public final class User {
    private final String name;
    private final LocalDate createdAt;

    public User(String name, LocalDate createdAt) {
        this.name = name;
        this.createdAt = createdAt;
    }

    public String name() { return name; }
}
```

Using Java 16+ Records:

```java
public record Product(String id, String name, BigDecimal price) {}
```

---

## Exception Handling

### Hierarchy
```
Throwable
 ├─ Exception (Checked)
 │   └─ IOException, SQLException
 └─ RuntimeException (Unchecked)
     └─ NullPointerException, IllegalArgumentException
```

### Best Practices
- Catch only when recovery is possible.
- Wrap low-level exceptions into domain-specific ones.
- Never swallow exceptions silently.
- Log meaningful context.

```java
try {
    process();
} catch (IOException ex) {
    throw new BusinessException("Failed to process", ex);
}
```

---

## Functional Programming & Streams
Streams provide declarative data processing.

```java
List<Integer> list = List.of(1, 2, 3, 4, 5);
int sum = list.stream()
              .filter(n -> n % 2 == 0)
              .mapToInt(n -> n)
              .sum();
```

- Avoid modifying shared state in stream operations.
- Use `parallelStream()` only for CPU-bound tasks.

Example – Word frequency map:

```java
Map<String, Long> frequency = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

---

## Optional API
Used to avoid `NullPointerException` and make absence explicit.

```java
Optional.ofNullable(user)
    .map(User::getEmail)
    .filter(email -> email.endsWith("@company.com"))
    .ifPresent(System.out::println);
```

- Do not use `Optional` as a class field.
- Avoid nested `Optional<Optional<T>>`.
- Use for method return types, not parameters.

---

## Wrapper Classes & Autoboxing
Autoboxing converts primitives to wrapper types automatically. Excessive boxing can impact performance.

```java
Integer counter = 0;
for (int i = 0; i < 1_000_000; i++) {
    counter += 1; // creates new Integer each iteration
}
```

Prefer primitives in performance-critical sections.

---

## Common Pitfalls & Best Practices
- Always use `try-with-resources` for automatic resource management:

```java
try (var reader = Files.newBufferedReader(Path.of("data.txt"))) {
    return reader.readLine();
}
```

- Use `StringBuilder` for concatenation in loops.
- Use `equals()` instead of `==` for string comparison.
- Use `enum` instead of magic constants.
- Ensure `equals()` and `hashCode()` are consistent.
- Protect shared mutable state with synchronization.
- Benchmark critical code with JMH.

---

## Performance & Profiling Tools
- **JMH**: Micro-benchmarking.
- **VisualVM / Java Flight Recorder (JFR)**: Profiling memory and CPU usage.
- **jstat -gc <pid> 1000**: Monitor GC behavior.
- **Thread dumps**: Analyze deadlocks and contention.
  deadlocks

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

### Isolation
- `READ_COMMITTED` (default in many RDBMS), `REPEATABLE_READ`, `SERIALIZABLE`, `READ_UNCOMMITTED`.
- Choose based on consistency vs contention requirements.

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

# Spring Boot & Microservices — Senior Developer Edition

## Overview
This section covers **Spring Boot 3** foundations and production-grade **microservice architecture**: auto-configuration, profiles, configuration management, service discovery, API gateway, resilience (Resilience4j), asynchronous messaging, **CQRS**, **Event Sourcing**, **Outbox Pattern**, **Saga**, **idempotency**, and **observability** with OpenTelemetry & Micrometer. All examples target **Java 17**.

---

## Spring Boot Essentials

### Auto-Configuration & Application Entry
Spring Boot reduces boilerplate with opinionated **auto-configuration** and an embedded application server.

```java
@SpringBootApplication // @Configuration + @EnableAutoConfiguration + @ComponentScan
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

**Notes**
- Favor **constructor injection**, immutable configuration, and fail-fast startup.
- Disable unwanted auto-config via `spring.autoconfigure.exclude` when needed.

### Configuration Precedence (high → low)
1. Command-line args
2. OS env vars
3. `application-{profile}.yml`
4. `application.yml`
5. Defaults in code

---

## Profiles & Externalized Config

```yaml
# application.yml
spring:
  application:
    name: payments

---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 8080
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

**Guidelines**
- Use **@ConfigurationProperties** for typed config and validate with `@Validated`.
- Secrets via Vault/AWS Secrets Manager/K8s Secrets; **never** commit to Git.

---

## Centralized Configuration

Use **Spring Cloud Config** or Git-backed config repos for consistency across services.

- Dynamic refresh with `spring-boot-starter-actuator` and `/actuator/refresh` (or Spring Cloud Bus).
- Keep config versioned, reviewed, and rolled out through CI/CD.

---

## Service Discovery & Client-side Load Balancing

- Kubernetes Service/Endpoints, Consul, or Eureka for discovery.
- Use **Spring Cloud LoadBalancer** for client-side load balancing.

```java
@Configuration
public class LbConfig {
    @Bean
    ReactorLoadBalancer<ServiceInstance> lb(Environment env,
                                            LoadBalancerClientFactory f) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(f.getLazyProvider(name,
                ServiceInstanceListSupplier.class), name);
    }
}
```

---

## API Gateway

A gateway centralizes cross-cutting concerns: routing, rate limiting, auth, and observability.

```yaml
# Spring Cloud Gateway
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: http://orders:8080
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
        - id: inventory
          uri: http://inventory:8080
          predicates:
            - Path=/api/inventory/**
```

**Alternative**: Kong, NGINX, Traefik, or an Ingress Controller in Kubernetes.

---

## Communication Styles

- **Synchronous**: REST/gRPC — simple request/response, tighter coupling.
- **Asynchronous**: Kafka/RabbitMQ — event-driven, decoupled, resilient to spikes.
- Choose per use case; avoid chaining long synchronous hops (latency & fragility).

---

## Resilience (Resilience4j)

Add runtime protections around remote calls.

```java
@Retry(name = "payments", fallbackMethod = "fallback")
@CircuitBreaker(name = "payments", fallbackMethod = "fallback")
@RateLimiter(name = "payments")
@TimeLimiter(name = "payments")
public CompletableFuture<OrderDto> createOrder(OrderDto dto) {
    return CompletableFuture.supplyAsync(() -> client.create(dto));
}

private CompletableFuture<OrderDto> fallback(OrderDto dto, Throwable ex) {
    // map to a safe response or enqueue for later processing
    return CompletableFuture.completedFuture(dto.withStatus("PENDING"));
}
```

**Patterns**
- **CircuitBreaker** to stop hammering failing deps.
- **Retry** with exponential backoff & jitter.
- **RateLimiter/Bulkhead** to isolate resources and protect threads.
- Timeouts everywhere — no remote call without a timeout.

---

## CQRS (Command Query Responsibility Segregation)

Separate **write model** (commands → domain) and **read model** (queries → projections). Benefits:
- Scales reads & writes independently.
- Allows different data models/technologies per side.

```text
API -> CommandHandler -> Domain -> Outbox(Event)  ||  API -> QueryHandler -> Read DB/Cache
```

Use cases: analytics dashboards, search views, heavy reporting.

---

## Event Sourcing

Persist **events** as the source of truth; rebuild state by replaying events.

- Strong audit & temporal queries (what/when/how changed).
- Usually combined with CQRS (projectors build read models).

Considerations: event schema evolution, snapshotting, replay performance.

---

## Outbox Pattern (Dual-write Guard)

**Goal**: publish domain events **atomically** with database writes to avoid lost messages.

### Transactional Outbox (Scheduler)
1) Persist domain state + outbox event **in the same local transaction**.
2) A scheduler/worker reads `PENDING` events and publishes to Kafka/Rabbit; marks `SENT`.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("order.created", json(order)));
}

@Scheduled(fixedDelay = "2s")
public void publish() {
    var batch = outboxRepo.findTop100ByStatus("PENDING");
    for (var e : batch) {
        kafkaTemplate.send("order-events", e.getAggregateId(), e.getPayload());
        e.setStatus("SENT");
        outboxRepo.save(e);
    }
}
```

### CDC-based Outbox (Debezium)
Use **Debezium** to capture DB changes and publish to Kafka — eliminates scheduler race conditions and scales better.

**Status Columns**: `status` (PENDING/SENT/FAILED), `retryCount`, `lastError`.

**Idempotency**: set a stable `eventId` and deduplicate on the consumer side.

---

## Saga Pattern (Distributed Transactions)

### Orchestration
A central **Orchestrator** coordinates steps and **compensations**.

```java
public void createOrderSaga(CreateOrder cmd) {
    reserveInventory(cmd);
    chargePayment(cmd);
    confirmOrder(cmd);
}
// On failure -> run compensations: refundPayment(), releaseInventory()
```

### Choreography
Services emit/subscribe to domain events; **no central coordinator**.  
Keep steps small; avoid cyclic event storms. Use correlation IDs.

**Choose**
- Orchestration for complex, multi-step flows.
- Choreography for simple, loosely-coupled flows.

---

## Idempotency

Guarantee that retried operations **do not create duplicates**.

- Generate an **idempotency key** (requestId) from client or server.
- Store a **dedup record** keyed by (operation, key).
- For Kafka, use **idempotent producer** + **transactional writes** and dedup on consumer.

```java
public PaymentResponse charge(PaymentRequest req) {
    return dedupStore.computeIfAbsent(req.idempotencyKey(),
        k -> gateway.charge(req));
}
```

---

## Observability

### Metrics (Micrometer → Prometheus/Grafana)
- JVM: memory, GC, threads, classes.
- App: HTTP latencies, DB pool, message lag.
- Custom: business counters (orders_created_total).

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
```

### Tracing (OpenTelemetry)
- Propagate **traceId/spanId** across services.
- Export to Jaeger/Tempo/Zipkin.
- Include correlation ID in logs (MDC).

### Logging
- Structured JSON logs; **never log PII**.
- Include request IDs and user context.

---

## Health, Liveness, Readiness

Expose Kubernetes-friendly health groups.

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: db, kafka, redis
        liveness:
          include: ping
```

- **Liveness**: restart stuck containers.
- **Readiness**: stop receiving traffic until dependencies are ready.

---

## 12-Factor & Release Engineering

- One codebase, multiple deploys; config in env; logs as event streams.
- Immutable images; Git-versioned artifacts; fast startup & graceful shutdown.
- **Feature flags** for safe, gradual rollout.

---

## Containerization & Deployment

### Dockerfile (Multi-stage)
```dockerfile
# build stage
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn -q -DskipTests package

# runtime stage
FROM eclipse-temurin:17-jre
ENV JAVA_OPTS="-XX:+UseG1GC -XX:MaxRAMPercentage=75.0"
WORKDIR /opt/app
COPY --from=build /app/target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Kubernetes Probes
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
```

### Deployment Patterns
- **Rolling update** for zero-downtime default.
- **Blue/Green** for instant rollback.
- **Canary** for gradual traffic shifting.

---

## Security Notes (Brief)

- OAuth2/OIDC for auth; stateless JWTs with short TTLs and rotation.
- Validate inputs, enforce RBAC/ABAC, encrypt at rest & in transit (TLS).
- Store secrets in Vault/KMS; never in repo.

---

## Quick Production Checklist

- Timeouts + retries + circuit breakers for all remote calls.
- Idempotency for external payments/commands.
- Outbox/CDC for reliable events; DLT for poison messages.
- Metrics, tracing, logs with correlation IDs.
- Health groups for K8s; readiness gates for dependencies.
- Immutable Docker images; version everything; automate via CI/CD.

---

# Messaging — Kafka & RabbitMQ (Senior Developer Edition)

## Overview
This section covers **asynchronous messaging** with **RabbitMQ** and **Apache Kafka** for microservice architectures. It focuses on delivery semantics, idempotency, retries & DLQs, publisher confirms, transactional outbox, consumer rebalances, schema evolution, and operational monitoring. All examples align with **Java 17 + Spring Boot 3**.

---

## RabbitMQ

### Core Concepts
- **Producer** → sends messages to an **Exchange**.
- **Exchange** routes messages to **Queues** using **Bindings**.
- **Consumer** reads from queues and acknowledges (`ACK/NACK/REQUEUE`).
- Exchange types: **direct**, **topic**, **fanout**, **headers**.

### Durable Topology & Message Persistence
- Declare **durable exchanges/queues** + publish messages with `deliveryMode=2` for persistence.
- Use **quorum queues** (recommended) for HA instead of classic mirrored queues.

```java
@Configuration
class RabbitTopologyConfig {

    public static final String EXCHANGE = "orders.ex";
    public static final String QUEUE = "orders.q";
    public static final String DLX = "orders.dlx";
    public static final String DLQ = "orders.dlq";

    @Bean
    DirectExchange ordersExchange() { return ExchangeBuilder.directExchange(EXCHANGE).durable(true).build(); }

    @Bean
    DirectExchange deadLetterExchange() { return ExchangeBuilder.directExchange(DLX).durable(true).build(); }

    @Bean
    Queue ordersQueue() {
        return QueueBuilder.durable(QUEUE)
            .withArgument("x-dead-letter-exchange", DLX)
            .withArgument("x-dead-letter-routing-key", "orders.dead")
            .build();
    }

    @Bean
    Queue ordersDlq() { return QueueBuilder.durable(DLQ).build(); }

    @Bean
    Binding bindOrders() { return BindingBuilder.bind(ordersQueue()).to(ordersExchange()).with("orders.created"); }

    @Bean
    Binding bindDlq() { return BindingBuilder.bind(ordersDlq()).to(deadLetterExchange()).with("orders.dead"); }
}
```

### TTL, Delays, Retry & DLQ
- Use **per-message TTL** or **per-queue TTL** to implement delays.
- Retry strategy:
    1) Consumer NACK → requeue to **retry queue** with TTL.
    2) After TTL, message returns to main queue.
    3) After **max attempts**, route to **DLQ** (poison messages).

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: MANUAL
        prefetch: 50
        retry:
          enabled: false # prefer explicit retry handling
```

### Publisher Confirms & Returns
- Enable **publisher confirms** to ensure the broker persisted the message.
- Use **mandatory** + return callbacks for unroutable messages.

```java
@Bean
RabbitTemplate rabbitTemplate(ConnectionFactory cf) {
    var tpl = new RabbitTemplate(cf);
    tpl.setMandatory(true);
    tpl.setConfirmCallback((correlation, ack, cause) -> {
        if (!ack) log.error("Publish not confirmed: {}", cause);
    });
    tpl.setReturnsCallback(ret -> log.error("Unroutable message: {}", ret));
    return tpl;
}
```

### Consumer (Manual ACK + Idempotency)
```java
@RabbitListener(queues = RabbitTopologyConfig.QUEUE, concurrency = "3-10")
public void onMessage(Message msg, Channel channel) throws IOException {
    var deliveryTag = msg.getMessageProperties().getDeliveryTag();
    try {
        var eventId = msg.getMessageProperties().getHeader("eventId");
        if (dedupStore.isProcessed(eventId)) {
            channel.basicAck(deliveryTag, false); // idempotent ack
            return;
        }
        handle(msg);
        dedupStore.markProcessed(eventId);
        channel.basicAck(deliveryTag, false);
    } catch (TransientException e) {
        channel.basicNack(deliveryTag, false, true); // requeue
    } catch (Exception e) {
        channel.basicReject(deliveryTag, false); // to DLQ via DLX
    }
}
```

**Notes**
- Set **prefetch** to tune consumer throughput and memory.
- Always include a stable **eventId** for idempotency.

---

## Apache Kafka

### Core Concepts
- **Topic** (append-only log) split into **Partitions** for scalability and per-partition ordering.
- **Broker** stores partitions; **Replication** provides fault tolerance.
- **Producer** writes records; **Consumer** reads via **Consumer Groups** for parallelism.
- Each record has an **offset** within a partition.

### Producer: Idempotence & Transactions
- Enable **idempotent producer** (`enable.idempotence=true`) to avoid duplicates on retries.
- Use **transactions** for atomic writes across partitions or topics (EOS).

```yaml
spring:
  kafka:
    producer:
      acks: all
      retries: 5
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        transactional.id: app-payments-tx-1
```

```java
@Service
public class PaymentPublisher {
    private final KafkaTemplate<String, PaymentEvent> kafka;

    public PaymentPublisher(KafkaTemplate<String, PaymentEvent> kafka) {
        this.kafka = kafka;
        this.kafka.setTransactionIdPrefix("payments-");
    }

    @Transactional("kafkaTransactionManager") // Spring for Kafka TM
    public void publish(PaymentEvent evt) {
        kafka.executeInTransaction(kt -> {
            kt.send("payments", evt.orderId(), evt);
            return true;
        });
    }
}
```

### Consumer Groups & Rebalances
- A **group** coordinates partitions among consumers (max one consumer per partition).
- On **rebalance** (join/leave/assignments), avoid processing until partitions are assigned; commit offsets **after** successful processing.

```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL
      concurrency: 3
    consumer:
      enable-auto-commit: false
      group-id: payments-svc
```

```java
@KafkaListener(topics = "payments", concurrency = "3")
public void onMessage(ConsumerRecord<String, PaymentEvent> r, Acknowledgment ack) {
    try {
        // process
        ack.acknowledge(); // commit offset after success
    } catch (TransientException e) {
        // retry/park
    } catch (Exception e) {
        // send to DLT
    }
}
```

### Retry Topics & Dead Letter Topic (DLT)
- Avoid blocking consumer threads; use **retry topics** with backoff delays.
- On final failure, send to **DLT** for inspection.

```yaml
spring:
  kafka:
    topics:
      - name: payments
      - name: payments-retry-5s
      - name: payments-retry-30s
      - name: payments-dlt
```

```java
@Component
@RequiredArgsConstructor
class RetryHandler {
    private final KafkaTemplate<String, PaymentEvent> kafka;

    void retry(PaymentEvent evt, String topic) {
        kafka.send(topic, evt.orderId(), evt);
    }

    void deadLetter(PaymentEvent evt) {
        kafka.send("payments-dlt", evt.orderId(), evt);
    }
}
```

### Exactly-Once Processing (EOS)
- Producer: **idempotence + transactions**.
- Consumer: deduplicate by a **stable key** (eventId) or use an **Inbox** table with unique constraint.
- Combine with **Outbox** on producer side for end-to-end guarantees.

### Schema Evolution
- Prefer **Avro/Protobuf** with a **Schema Registry** for compatibility (backward/forward).
- Strategy: bump schema versions, use defaults for new fields, never break consumers.

### Kafka Streams (Brief)
- High-level DSL for processing with **state stores** and **windowing**.

```java
StreamsBuilder b = new StreamsBuilder();
KStream<String, PaymentEvent> stream = b.stream("payments");
stream
  .groupByKey()
  .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
  .count(Materialized.as("payments-counts"))
  .toStream()
  .to("payments-aggregates");
```

**Notes**
- Monitor **changelogs**, state store size, and rebalancing pauses.
- Use **RocksDB** (default) carefully; watch disk I/O and compaction.

### Partitions, Keys & Ordering
- Ordering is **per partition**; choose a stable **key** (e.g., `orderId`) to keep related records ordered.
- Balance partitions to avoid **hot keys**; increase partitions carefully (affects ordering).

---

## Delivery Semantics

- **At most once**: possible loss, no duplicates.
- **At least once**: no loss (with retries), duplicates possible → require **idempotency**.
- **Exactly once**: no loss, no duplicates; requires **EOS** setup and careful consumer design.

---

## Reliable Messaging Patterns

### Outbox (recap)
- Write domain state + outbox in same DB transaction.
- Publisher process (scheduler/CDC) emits events; mark as SENT.
- Include `eventId`, `aggregateId`, `occurredAt`, `type`, `payload`, `retryCount`.

### Inbox (Idempotent Consumer)
- Consumers insert `eventId` into **Inbox** with unique constraint before processing.
- If insert fails (duplicate), **skip** processing; otherwise, proceed and mark **PROCESSED**.

### Exactly-Once with Kafka + DB
- Use Kafka transactions to write to a topic **and** a DB in one logical unit:
    - **Consume** → process → **produce** (or write DB) → **commit offsets** transactionally.

---

## Monitoring & Operations

### Key Metrics
- **RabbitMQ**: queue depth, unacked messages, consumer utilization, connection count.
- **Kafka**: consumer lag, request rate, byte in/out, under-replicated partitions, ISR size.
- Track **publish acks**, **DLT volume**, **retry counts**, and **processing latency**.

### Tools
- RabbitMQ Management UI, Prometheus + Grafana.
- Kafka JMX + Prometheus JMX Exporter, Kafka UI/AKHQ/Conduktor, Confluent Control Center.

### Capacity & Tuning
- RabbitMQ: set **prefetch**, tune **quorum queue** params, size connections & channels.
- Kafka: size **partitions**, **replication factor ≥ 3**, tune broker I/O and network.

---

## Quick Checklists

**RabbitMQ**
- Durable exchanges/queues, quorum queues.
- Mandatory flag + confirms/returns.
- TTL-based retry + DLQ routing.
- Manual ACK and idempotency keys.

**Kafka**
- Keys for ordering; enough partitions for throughput.
- Idempotent producer + transactional writes if needed.
- Manual acks; commit after success.
- Retry topics with backoff; DLT for poison messages.
- Schema registry for evolution.

---

# Caching & Redis — Senior Developer Edition

## Overview
This section covers **Spring Cache Abstraction** with **Redis** and **Caffeine**, transaction-aware caching, multi‑level cache design, serialization, eviction/TTL strategies, cache stampede/penetration/avalanche prevention, **distributed locking (Redisson)**, and **observability**. Target stack: **Java 17 + Spring Boot 3**.

---

## Spring Cache Abstraction

### Core Annotations
- `@EnableCaching` — activates caching.
- `@Cacheable` — caches method results (skips execution when cache hit).
- `@CachePut` — always executes method and **updates** cache.
- `@CacheEvict` — removes entries (single or all).
- `@Caching` — compose multiple cache operations.
- `@CacheConfig` — common cache config at class level (cache names, key generator, cacheManager).

```java
@EnableCaching
@SpringBootApplication
public class App { public static void main(String[] args) { SpringApplication.run(App.class, args); } }

@CacheConfig(cacheNames = "product-cache", cacheManager = "redisCacheManager")
@Service
public class ProductService {

    @Cacheable(key = "#id") // miss → load; hit → return cached
    public ProductDto getProduct(UUID id) { return repository.findDtoById(id); }

    @CachePut(key = "#dto.id()")
    public ProductDto update(ProductDto dto) { return repository.save(dto); }

    @CacheEvict(key = "#id")
    public void delete(UUID id) { repository.deleteById(id); }

    @CacheEvict(allEntries = true) // careful!
    public void clearAll() { /* maintenance */ }
}
```

### SpEL Keys & Custom Key Generator
```java
@Bean
public KeyGenerator tenantAwareKeyGen() {
    return (target, method, params) -> "%s:%s:%s".formatted(
        TenantContext.getTenantId(), method.getName(), Arrays.deepToString(params));
}
```

```java
@CacheConfig(keyGenerator = "tenantAwareKeyGen")
class PriceService {
    @Cacheable(cacheNames = "prices", key = "#sku + ':' + #currency")
    BigDecimal getPrice(String sku, String currency) { /* ... */ }
}
```

---

## Cache Managers

### Caffeine (Local, High-Perf)
```java
@Bean
public CacheManager caffeineCacheManager() {
    Caffeine<Object, Object> spec = Caffeine.newBuilder()
        .maximumSize(50_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .recordStats();
    return new CaffeineCacheManagerBuilder().fromCaffeine(spec).build();
}
```

### Redis (Distributed)
Spring Boot 3 default serializer for Redis cache: **GenericJackson2JsonRedisSerializer**.

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 1h
      cache-null-values: false
  data:
    redis:
      host: localhost
      port: 6379
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
```

```java
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory cf) {
    RedisCacheConfiguration conf = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofHours(1))
        .disableCachingNullValues()
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    return RedisCacheManager.builder(cf).cacheDefaults(conf).build();
}
```

**Tip**: Use **separate cache managers** (local + Redis) if you design **multi‑level cache**.

---

## Transaction-Aware Caching

Avoid caching uncommitted data in a transactional context.

```java
@Bean
public CacheManager txAwareRedisCacheManager(RedisConnectionFactory cf) {
    RedisCacheManager rcm = redisCacheManager(cf);
    return new TransactionAwareCacheManagerProxy(rcm);
}
```

`TransactionAwareCacheManagerProxy` postpones cache writes until **after commit**.

---

## Multi‑Level Cache (Caffeine + Redis)

- **L1 (Caffeine)**: ultra‑fast local hits, instance‑scoped.
- **L2 (Redis)**: cross‑instance consistency, TTL control.
- **Invalidation**: publish an **invalidation event** (Redis Pub/Sub) so all nodes evict L1 when L2 changes.

```java
public interface TwoLevelCache {
    <T> T get(String cache, Object key, Class<T> type, Supplier<T> loader, Duration ttl);
}
```

**Invalidation Flow**: write → DB → update L2 (Redis) → publish `invalidate:L1:cache:key` → all nodes evict L1 entry.

---

## Eviction & TTL Strategies

- **TTL (Time‑to‑Live)**: ensure stale data eventually expires.
- **LRU / LFU**: Redis supports LFU/LRU with `maxmemory-policy`.
- Different object classes often require **different TTLs** (pricing 5m, product 1h, catalog 24h).

```yaml
# redis.conf excerpt
maxmemory 4gb
maxmemory-policy allkeys-lfu
```

---

## Serialization

- Prefer JSON for readability and cross‑lang clients (`GenericJackson2JsonRedisSerializer`).
- Include **type information** when polymorphism is required.
- For hot paths, consider **Kryo/Smile** (custom) — weigh readability vs perf.
- Keep payloads small; cache identifiers rather than huge graphs.

**Pitfall**: Changing class names/packages breaks deserialization of existing keys. Use **stable DTOs**.

---

## Cache Stampede / Avalanche / Penetration

### Problems
- **Stampede (Dogpile)**: Many requests rebuild the same key after expiry.
- **Avalanche**: Many keys expire at once → thundering herd.
- **Penetration**: Repeatedly querying **non‑existent** keys (DB miss each time).

### Mitigations
- **Mutex/Single‑Flight**: Only one thread recomputes; others wait.
- **Probabilistic Early Refresh**: Refresh slightly **before TTL** (add jitter).
- **Randomized TTLs**: Avoid synchronized expirations (+/− jitter).
- **Negative Caching**: Cache “miss” (e.g., `NULL`) with **short TTL** (e.g., 30s).
- **Bloom Filter**: Reject obviously invalid keys before cache/DB.

```java
// Single-flight (naive)
ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

public <T> T cached(String key, Supplier<T> loader) {
    String k = "p:" + key;
    T v = redis.get(k);
    if (v != null) return v;

    var lock = locks.computeIfAbsent(k, kk -> new ReentrantLock());
    lock.lock();
    try {
        v = redis.get(k);
        if (v != null) return v;
        v = loader.get();
        if (v == null) redis.setex(k, 30, NULL); // negative cache
        else redis.setex(k, 3600 + randJitter(), v);
        return v;
    } finally {
        lock.unlock();
    }
}
```

---

## Distributed Locks (Redisson)

Use **Redisson** for reliable distributed locks (supports watchdog extensions).

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.redisson</groupId>
  <artifactId>redisson-spring-boot-starter</artifactId>
  <version>3.27.2</version>
</dependency>
```

```java
@Service
public class BillingJob {
    private final RedissonClient redisson;

    public BillingJob(RedissonClient redisson) { this.redisson = redisson; }

    public void runMonthlyBilling() {
        RLock lock = redisson.getLock("locks:billing-monthly");
        boolean acquired = lock.tryLock(5, 120, TimeUnit.SECONDS); // wait 5s, lease 120s
        if (!acquired) return;
        try {
            // critical section (idempotent operations)
        } finally {
            lock.unlock();
        }
    }
}
```

**Guidelines**
- Always set **lease time** or rely on **watchdog** (auto‑extends while thread is alive).
- Make critical sections **idempotent** (in case of retries or failover).
- Avoid long locks; prefer small, composable operations.

---

## Atomic Operations (Lua)

Use **Lua scripts** for atomic read‑modify‑write.

```java
DefaultRedisScript<Long> script = new DefaultRedisScript<>(
    "if redis.call('EXISTS', KEYS[1]) == 1 then " +
    "  return redis.call('INCRBY', KEYS[1], ARGV[1]); " +
    "else return nil end", Long.class);

Long res = stringRedisTemplate.execute(script, List.of("counter:sku:123"), "1");
```

Use cases: rate limiting, counters, guarding cache rebuilds.

---

## Redis Topologies

- **Standalone**: single node (dev/testing).
- **Sentinel**: HA failover management for master‑replica.
- **Cluster**: sharding & HA (production).

```yaml
spring:
  data:
    redis:
      cluster:
        nodes: host1:6379,host2:6379,host3:6379
      # OR
      sentinel:
        master: mymaster
        nodes: host1:26379,host2:26379,host3:26379
```

---

## Redis Data Structures (Quick Use Cases)

- **Strings**: cache values, counters, locks, tokens.
- **Hashes**: per‑object attributes (`HGETALL product:123`).
- **Lists**: queues, recent items.
- **Sets**: unique tags, membership checks.
- **Sorted Sets (ZSET)**: leaderboards, score‑based ranking.
- **Pub/Sub**: cache invalidation broadcasts, notifications.
- **Streams**: durable event logs with consumer groups (lightweight Kafka alternative).

---

## Observability & Troubleshooting

### Metrics
- Cache hit/miss ratio, evictions, average load time.
- Redis: memory usage, connected clients, ops/sec, keyspace hits/misses, latency spikes.

### Tools
- **RedisInsight**, `INFO` command.
- Prometheus + Grafana dashboards (Redis exporter).
- Spring Boot Actuator: `/actuator/metrics/cache.*`

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
```

### Common Pitfalls
- Huge values → memory pressure + slow serialization.
- Key explosions (unbounded cardinality).
- Missing TTLs → unbounded growth.
- Storing PII without encryption.
- Not invalidating L1 after L2 updates (multi‑level drift).

---

## Quick Checklists

**Design**
- Choose correct TTL per domain (short for volatile, long for static).
- Negative caching for known‑missing records.
- Multi‑level cache with L1 invalidation via Pub/Sub.

**Reliability**
- Transaction‑aware cache writes.
- Use Lua or Redisson locks for atomic sections.
- Jitter TTLs to prevent avalanche.

**Security**
- Never cache sensitive data unencrypted.
- Separate keyspaces per environment/tenant (`app:env:module:key`).

---


# Data Access & Performance — Senior Developer Edition

## Overview
End-to-end guidance for **Spring Data JPA + Hibernate** on **Java 17 / Spring Boot 3**: entity design, fetch strategies, N+1 avoidance, projections, batch & bulk operations, pagination (offset vs keyset), transaction boundaries, locking (optimistic/pessimistic), connection pool tuning (**HikariCP**), migrations (Flyway/Liquibase), and query monitoring. Production-first, interview-ready.

---

## Entity Design

### Rules
- Keep entities **lean**; avoid dumping business logic that belongs to services/aggregates.
- Use **immutable identifiers** (UUID) and **surrogate keys** when natural key is unstable.
- Prefer **unidirectional** associations unless you truly need bidirectional navigation.
- Consider DTO/record for API models; do not expose entities directly over REST.

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Customer customer;

    @Version
    private long version; // optimistic locking

    @Column(nullable = false)
    private BigDecimal total;

    // getters/setters
}
```

**Tip**: Use `@Version` for optimistic locking on *mutable* aggregates.

---

## Relationship Mapping

- `@ManyToOne` defaults to **EAGER** in JPA spec → **override to LAZY** explicitly.
- `@OneToMany` should be **LAZY** (default). Use `Set` when order doesn't matter to avoid duplicates.
- `@ManyToMany` creates a join table; avoid for hot paths. Prefer two **OneToMany** with an explicit join entity.

```java
@Entity
class Customer {
    @OneToMany(mappedBy = "customer", cascade = CascadeType.PERSIST, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}
```

---

## Fetch Strategies & N+1

### Detect
- Enable Hibernate `org.hibernate.SQL` and `org.hibernate.type.descriptor.sql` **only in dev/test**.
- Use integration tests + Testcontainers and assert query counts if needed.

### Avoid
- **Fetch Join** (JPQL)
```java
@Query("select c from Customer c join fetch c.orders where c.id = :id")
Customer findWithOrders(UUID id);
```

- **Entity Graph**
```java
@EntityGraph(attributePaths = {"orders", "orders.items"})
Optional<Customer> findById(UUID id);
```

- **Batch Fetching** (for LAZY collections/associations)
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=64
```

- **DTO Projections** to load only needed columns
```java
public record OrderSummary(UUID id, BigDecimal total, Instant createdAt) {}

@Query("select new com.example.OrderSummary(o.id, o.total, o.createdAt) from Order o where o.customer.id = :cid")
List<OrderSummary> findSummaries(UUID cid);
```

**Note**: Avoid `EAGER` as global fix — can cause Cartesian explosion.

---

## Projections (Interface / Class)

- **Interface-based** (close to column aliases):
```java
public interface CustomerView {
    UUID getId();
    String getEmail();
    BigDecimal getLifetimeValue();
}
List<CustomerView> findByEmailContaining(String s);
```

- **Class-based** (constructor):
```java
public record CustomerDto(UUID id, String email, BigDecimal lifetimeValue) {}
@Query("select new com.example.CustomerDto(c.id, c.email, c.lifetimeValue) from Customer c")
List<CustomerDto> fetchAll();
```

- **Native projections** with `@SqlResultSetMapping` when necessary for complex queries.

---

## Pagination

### Offset Pagination (simple, common)
```java
Page<Order> page = repo.findByCustomerId(cid, PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "createdAt")));
```

**Cons**: Large offsets degrade performance (DB scans).

### Keyset Pagination (seek method, scalable)
```java
@Query("select o from Order o where o.customer.id = :cid and o.createdAt < :before order by o.createdAt desc")
List<Order> findNextPage(UUID cid, Instant before, Pageable pageable);
```

**Tip**: Use an indexed, monotonic column (e.g., `createdAt`, `id`) for the seek condition.

---

## Batch & Bulk Operations

### JDBC Batch (Hibernate)
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

```java
for (int i = 0; i < items.size(); i++) {
    em.persist(items.get(i));
    if (i % 50 == 0) {
        em.flush(); // push to JDBC
        em.clear(); // detach to avoid memory bloat
    }
}
```

### Bulk JPQL (no entity lifecycle events)
```java
@Modifying
@Query("update Order o set o.status = 'CANCELLED' where o.expireAt < :now")
int cancelExpired(Instant now);
```

**Note**: Bulk JPQL bypasses first-level cache → refresh entities or clear persistence context after execution.

---

## Transactions & Boundaries

- Put `@Transactional` on **public service methods** (not private).
- Default rollback only for **unchecked** exceptions; use `rollbackFor` for checked.
- Keep transactions **short**; avoid remote calls inside Tx.
- Separate **read** and **write** transactions when possible.

```java
@Service
public class OrderService {
    @Transactional
    public void place(OrderCommand cmd) { /* write operations */ }

    @Transactional(readOnly = true)
    public Optional<OrderView> detail(UUID id) { /* read operations */ }
}
```

**Propagations**: `REQUIRES_NEW` for outbox logging or audit writes on failure paths.

---

## Locking

### Optimistic (default choice)
- Add `@Version` to detect lost updates. On conflict, retry command.

```java
try {
   repo.save(entity);
} catch (ObjectOptimisticLockingFailureException e) {
   // retry or surface conflict to client
}
```

### Pessimistic (when contention is real)
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select o from Order o where o.id = :id")
Optional<Order> lockForUpdate(UUID id);
```

**Caveats**: DB-level locks can cause deadlocks; always set timeouts.

```properties
spring.jpa.properties.hibernate.jpa.compliance.query=true
spring.jpa.properties.hibernate.jdbc.timeout=5
```

---

## Connection Pool (HikariCP)

Tune based on workload & DB limits.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 600000     # 10m
      connection-timeout: 30000
      max-lifetime: 1800000    # 30m
```

**Guidelines**
- `maximum-pool-size` ≤ DB max connections per app node.
- Watch **wait time**, **active connections**, and **timeouts** in metrics.
- Validate connection liveness (`connectionTestQuery` or JDBC4 isValid).

---

## Migrations (Flyway/Liquibase)

- Versioned, repeatable, and **idempotent** migrations as part of CI/CD.
- One-way migrations; never edit applied scripts — create new ones.
- Keep **DDL** in V scripts, **reference data** in R scripts (if needed).

**Flyway Example**
```
V1__init.sql
V2__add_index_on_order_created_at.sql
R__baseline_reference_data.sql
```

**Indexing**
```sql
-- Narrow composite index for common filter
CREATE INDEX ix_order_customer_created_at ON orders (customer_id, created_at DESC);

-- Partial index (PostgreSQL)
CREATE INDEX ix_order_status_open ON orders (created_at) WHERE status = 'OPEN';
```

---

## Native Queries & Query Plans

- Use native SQL for complex reporting queries or vendor-specific features.
- Always check **EXPLAIN / EXPLAIN ANALYZE** to verify index usage and cost.

```java
@Query(value = "select * from orders where customer_id = :cid order by created_at desc limit :n",
       nativeQuery = true)
List<Order> lastOrders(@Param("cid") UUID cid, @Param("n") int n);
```

---

## Specifications & Criteria API

### Spring Data Specifications (dynamic filters)
```java
public class OrderSpecs {
    public static Specification<Order> byCustomer(UUID cid) {
        return (root, q, cb) -> cb.equal(root.get("customer").get("id"), cid);
    }
    public static Specification<Order> createdAfter(Instant t) {
        return (root, q, cb) -> cb.greaterThan(root.get("createdAt"), t);
    }
}
```
```java
List<Order> res = repo.findAll(byCustomer(cid).and(createdAfter(t)));
```

### Criteria API (type-safe dynamic queries)
Useful when building complex, dynamic predicates programmatically.

---

## Testing Strategy (Data Layer)

- **@DataJpaTest** for fast slice tests (auto-configures H2/embedded DB by default).
- Use **Testcontainers** with PostgreSQL/MySQL for realistic integration tests.
- Clear DB between tests; use transactions/rollback in tests.

```java
@DataJpaTest
@Testcontainers
class OrderRepoIT {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
}
```

---

## Monitoring & Diagnostics

- Enable Hibernate statistics only in non-prod:
```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```
- Use **Micrometer** DB pool metrics and custom timers for repository methods.
- Log slow queries (p6spy in dev; disable SQL logs in prod).

---

## Quick Checklists

**Design**
- LAZY by default; avoid EAGER except on read-only aggregates.
- DTO projections for read-heavy endpoints.
- Keyset pagination for infinite-scroll feeds.

**Performance**
- Batch inserts/updates; flush & clear periodically.
- Index most-used filter columns; use partial indexes.
- Avoid N+1 via fetch joins/entity graphs/batch size.

**Reliability**
- Use @Version optimistic locking; retry on conflict.
- Keep transactions short; place @Transactional on public service methods.
- Set timeouts; avoid long-held locks.

---

# System Design — Senior Developer Edition

## Overview
A practical guide to designing **scalable, resilient, observable** systems. Covers **CAP/BASE**, consistency models, idempotency, load balancing, backpressure, rate limiting, sharding/partitioning, caching layers, queues/streams, leader election, and release strategies (blue/green, canary). Focus: **Java 17 + Spring Boot 3** microservices in Kubernetes.

---

## CAP, Consistency & Availability

### CAP Theorem
- **Consistency (C)**: every read gets the latest write or an error.
- **Availability (A)**: every request receives a non-error response (not necessarily latest).
- **Partition Tolerance (P)**: system continues despite network partitions.

In the presence of partitions, you **choose** between C or A per subsystem. Many real systems are **AP with eventual consistency** for non-critical paths; **CP** for critical invariants (e.g., balances).

### BASE vs ACID
- **ACID**: strong guarantees within a single DB transaction.
- **BASE**: **B**asically **A**vailable, **S**oft state, **E**ventual consistency — suitable for distributed systems via async replication and compensation.

---

## Consistency Models (Quick Map)
- **Strong**: linearizable reads (CP systems, e.g., etcd/consensus).
- **Read-your-writes**: a client sees own writes after commit.
- **Monotonic reads**: reads don’t go back in time.
- **Causal**: preserves cause-effect order.
- **Eventual**: converges over time (with CRDTs/anti-entropy).

**Guideline**: apply strong consistency on *money movement & invariants*; eventual elsewhere (notifications, analytics).

---

## Idempotency

Ensure retries don’t create duplicates or side effects.

- **Keys**: `Idempotency-Key` header or server-side generated `requestId`.
- **Store**: dedup table/cache (`(operation, key) -> result/status`).
- **Scope**: define per resource (`PUT /orders/{id}`) or per operation type.

```java
public OrderResponse create(CreateOrder req, String key) {
    return dedupRepo.find(key).orElseGet(() -> {
        OrderResponse res = doCreate(req);
        dedupRepo.save(key, res);
        return res;
    });
}
```

For messaging, use **Outbox (producer)** + **Inbox (consumer)**.

---

## Load Balancing Strategies

- **Round Robin / Weighted RR**: simple distribution.
- **Least Connections / Least Response Time**: better under uneven load.
- **Client-side LB**: Spring Cloud LoadBalancer with service discovery.
- **CDN / Anycast**: for edge distribution of static assets/APIs.

**Sticky Sessions**: avoid for stateless services; if required, use consistent hashing.

---

## Backpressure & Flow Control

Prevent overload by **signaling producers** to slow down:

- **Queues/Streams** between services (Kafka/RabbitMQ) buffer spikes.
- **Rate limiting** at ingress (token bucket/leaky bucket).
- **Bulkheads**: isolate thread pools and queues per dependency.
- **Timeouts** + **circuit breakers** to fail fast.

```java
@Bulkhead(name = "inventory", type = Bulkhead.Type.THREADPOOL)
@RateLimiter(name = "inventory")
@TimeLimiter(name = "inventory")
public CompletableFuture<InventoryDto> reserve(...) { /* ... */ }
```

---

## Rate Limiting

- **Token Bucket** (burst-friendly) or **Leaky Bucket** (smoothed).
- Enforce at **API Gateway** (Kong/NGINX/SCG) + per-user/tenant keys.
- Persist counters in **Redis** (atomic Lua) or **distributed ratelimiters**.

```lua
-- token bucket (simplified)
local tokens = redis.call('GET', KEYS[1])
if (tokens and tonumber(tokens) > 0) then
  redis.call('DECR', KEYS[1]); return 1
else return 0 end
```

---

## Sharding & Partitioning

Distribute data/workload across nodes/partitions.

- **Range** (by time/id range) → good for scans; risk hot shards.
- **Hash** (by key, e.g., `orderId % N`) → balances writes; breaks range scans.
- **Geo/Directory** (by region/tenant) → data locality; cross-shard queries.

**Design**
- Choose **stable shard key** (not frequently updated).
- Keep **secondary indexes** per shard.
- Plan **re-sharding** (double-writing, dual reads, or online migration tools).

---

## Caching Layers

- **Client cache** (browser/app)
- **Edge cache** (CDN)
- **Service cache** (in-memory L1, Redis L2)
- **DB cache** (materialized views, read replicas)

**Patterns**
- Cache-aside, write-through, write-behind.
- Prevent stampede (single-flight, jitter, negative cache).

---

## Queues vs Streams

- **Queues (RabbitMQ/SQS)**: task distribution, per-message ACK, competing consumers.
- **Streams (Kafka/Pulsar)**: ordered logs, replayable, multiple consumer groups, stateful processing.

**Guideline**: use **queues** for work dispatch; **streams** for event-driven architectures & analytics.

---

## Release Strategies

### Blue/Green
Two identical environments; switch traffic instantly. Instant rollback. Requires DB schema **forward/backward** compatibility.

### Canary
Roll out to small % of traffic; watch metrics/SLOs; then ramp up.

### Rolling Update
Replace pods gradually. Default in Kubernetes; watch readiness & max surge/unavailable.

---

## Leader Election & Coordination

Use **consensus systems** (Raft/Paxos):
- **etcd**, **Consul**, or **Zookeeper** for distributed locks, service registry, and config.
- For simple single-winner tasks, prefer **K8s CronJobs** or **lease API** (coordination.k8s.io).

**Note**: Implementing consensus in-app is error-prone—delegate to platform.

---

## Observability Essentials

- **Metrics**: RED/USE methods; Micrometer → Prometheus/Grafana.
- **Tracing**: OpenTelemetry SDK + auto-instrumentation; propagate `traceId`.
- **Logs**: JSON, correlation IDs (MDC), PII-safe.

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
logging:
  pattern:
    level: "%5p traceId=%X{traceId} spanId=%X{spanId} - %m%n"
```

**SLOs**
- Availability (e.g., 99.9%), latency (p95/p99), error rate.
- Alert on **user-facing** symptoms, not just system internals.

---

## Storage Patterns

- **Read replicas** for heavy reads (eventual consistency trade-offs).
- **CQRS + Projections** for complex read models.
- **Event sourcing** for auditability & temporal queries.
- **Blob/object storage** (S3/GCS) for large media; serve via presigned URLs.

---

## Security & Multi-Tenancy (Brief)

- OAuth2/OIDC; short-lived JWTs; rotate refresh tokens.
- Tenant isolation: **schema-per-tenant** or **row-level** with tenantId + policies.
- Encrypt in transit (TLS) and at rest (KMS).
- Apply **least privilege**; rotate secrets automatically.

---

## Failure Modes & Chaos

- **Timeouts** everywhere; **retries** with backoff & jitter.
- **Circuit breakers** to stop cascading failures.
- **Bulkheads** to limit blast radius.
- **Chaos experiments** (latency, kill) in non-prod to validate resilience.

---

## Quick Blueprints

**High-throughput write path (payments)**
- API (idempotency key) → Command svc → DB (Tx) + Outbox → CDC → Kafka → Downstream.

**Read-heavy analytics**
- Event stream → Projections (Elasticsearch/ClickHouse) → API read svc → Cache → Client.

**File ingest pipeline**
- Upload → Virus scan → Extract metadata → Store S3 → Emit event → Index → Notify.

---

## Checklists

**Scalability**
- Stateless services; horizontal autoscaling.
- Partition-friendly data models; hot-key mitigation.
- Async processing and backpressure.

**Resilience**
- Timeouts, retries, circuit breakers.
- Graceful shutdown; SIGTERM handling; idempotent consumers.
- Health groups; readiness gates per dependency.

**Operability**
- Metrics/tracing/logs with correlation.
- Runbooks + SLOs; dashboards & alerts.
- Feature flags and safe rollouts.

---


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



# Advanced Topics — Senior Developer Edition

## Overview
Deep-dive notes for **Java 17 + Spring Boot 3** projects on production-grade concerns: **Resilience4j** advanced, **MapStruct** advanced mappings, **Spring Cache/Redis** advanced techniques, **Kafka retry/DLT** architectures, **Virtual Threads** adoption, **Workflow/Camunda** patterns, **Saga monitoring**, **OpenTelemetry** advanced tracing, and **Security hardening** checklist.

---

## Resilience4j — Advanced

### Policy Design
- **Timeout** on *every* remote call (HTTP/DB/MQ). Choose `TimeLimiter` for async, client timeout for sync (e.g., WebClient).
- **Retry** only for transient errors (5xx, timeouts, connection reset). **Never** retry POST without idempotency key.
- **CircuitBreaker** short-circuits failing dependencies; **Half-Open** probes recovery.
- **Bulkhead** isolates pools per dependency; use **THREADPOOL** for blocking I/O, **SEMAPHORE** for async.
- Centralize configs per dependency; avoid per-call ad‑hoc annotations.

```yaml
resilience4j:
  timelimiter:
    instances:
      payments:
        timeout-duration: 2s
  retry:
    instances:
      payments:
        max-attempts: 3
        wait-duration: 200ms
        retry-exceptions: org.springframework.web.client.ResourceAccessException, java.net.SocketTimeoutException
  circuitbreaker:
    instances:
      payments:
        sliding-window-size: 50
        minimum-number-of-calls: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
  bulkhead:
    instances:
      payments:
        max-concurrent-calls: 50
        max-wait-duration: 100ms
```

```java
@TimeLimiter(name = "payments")
@Retry(name = "payments")
@CircuitBreaker(name = "payments")
@Bulkhead(name = "payments", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<OrderDto> createOrder(OrderDto dto) {
    return CompletableFuture.supplyAsync(() -> client.create(dto));
}
```

**Observability**: expose metrics to Prometheus; alert on open breakers and high retry rates.

---

## MapStruct — Advanced Mappings

### Null Handling & Defaults
```java
@Mapper(componentModel = "spring", nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS,
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface UserMapper {
    @Mapping(target = "name", source = "fullName", defaultValue = "Anonymous")
    @Mapping(target = "createdAt", expression = "java(java.time.Instant.now())")
    User toEntity(UserDto dto);

    @BeanMapping(ignoreByDefault = true)
    @Mapping(target = "name", source = "name")
    @Mapping(target = "email", source = "email")
    UserDto toDto(User user);
}
```

### Nested & Custom Conversions
```java
@Mapper(componentModel = "spring", uses = { MoneyMapper.class, AddressMapper.class })
public interface OrderMapper {
    @Mappings({
        @Mapping(target = "total", source = "totalMinor", qualifiedByName = "minorToMoney"),
        @Mapping(target = "address.line", source = "addressLine")
    })
    Order toEntity(OrderDto dto);
}
```

```java
@Mapper(componentModel = "spring")
public interface MoneyMapper {
    @Named("minorToMoney")
    default Money fromMinor(long minor, String currency) {
        return new Money(BigDecimal.valueOf(minor, 2), Currency.getInstance(currency));
    }
}
```

### Update Mapping (PATCH semantics)
```java
@Mapper(componentModel = "spring")
public interface PatchMapper {
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void update(@MappingTarget User target, UserPatch patch);
}
```

**Tips**
- Use **records** for DTOs when possible.
- Generate **test mappers** to assert tricky field conversions.

---

## Spring Cache & Redis — Advanced

### Multi-Region & Tenant Isolation
- Prefix keys: `app:env:tenant:cache:key`.
- For multi-region, avoid cache cross-talk; layer per region with eventual consistency via async invalidation events.

### Read-Through vs Write-Behind
- **Write-through**: update cache and DB synchronously.
- **Write-behind**: queue updates to DB; higher throughput, risk of loss without durable queue.

### Hot Key Mitigation
- Shard hot keys with **consistent hashing suffix**: `key:{hash(n)}`; combine aggregates on read.
- Use **local L1** for hot keys with short TTL and **Pub/Sub invalidations**.

### Rate Limiting with Redis
```java
DefaultRedisScript<Long> script = new DefaultRedisScript<>(
  "local tokens=redis.call('get',KEYS[1]) or 0; " +
  "if tonumber(tokens) > 0 then return redis.call('decr',KEYS[1]) else return -1 end", Long.class);
Long ok = stringRedisTemplate.execute(script, List.of("rl:user:42"), List.of());
```

---

## Kafka — Retry/DLT Architectures

### Strategy Options
1) **Blocking retry** in consumer — simple, but stalls partition; avoid for high throughput.
2) **Retry topics** with backoff (e.g., `topic.retry.5s`, `topic.retry.30s`) + **DLT** after N attempts.
3) **Transactional outbox** on producer + **Inbox** on consumer for deduplication.
4) **Poison pill** parking: route to quarantine topic for manual inspection.

```java
public void handle(PaymentEvent evt) {
    try {
        process(evt);
        ack.acknowledge();
    } catch (TransientException e) {
        retryTemplate.send("payments-retry-30s", evt.key(), evt); // delayed retry
    } catch (Exception e) {
        deadLetter.send("payments-dlt", evt.key(), new ErrorEnvelope(evt, e));
    }
}
```

**Headers**
- `x-attempts`, `x-first-seen`, `traceId` for observability and SLA analysis.

**Compaction**
- Use **compacted topics** for dedup state (eventId → processedAt).

---

## Virtual Threads — Adoption Checklist

- Replace server thread pools with virtual-friendly executors for blocking I/O.
- Ensure **drivers/clients** are **blocking-friendly** (JDBC, HTTP) or switch to non-blocking with adapted semantics.
- Watch **synchronized** blocks and thread-local usage; ensure no thread affinity assumptions.
- Limit per-request memory; avoid deep recursion; monitor **carrier** thread utilization.

```java
var exec = Executors.newVirtualThreadPerTaskExecutor();
Future<String> f = exec.submit(() -> http.get("https://api").body());
```

**Metrics**: track tasks submitted, carrier thread usage, blocked thread time.

---

## Workflow & Camunda (Concepts)

- Use BPMN for **long-running, human-in-the-loop** processes (KYC, refunds).
- Keep service logic in code; use workflow for orchestration & visibility.
- Ensure **idempotent** external tasks; store **business keys** for correlation.
- Model **compensations** explicitly (cancel, refund).

```java
@ExternalTaskSubscription("charge-payment")
public void charge(ExternalTask task, ExternalTaskService svc) {
    var key = task.getBusinessKey(); // correlation id
    try {
        payment.charge(...);
        svc.complete(task);
    } catch (TransientException e) {
        svc.handleFailure(task, "retryable", e.getMessage(), 0, 300_000); // 5 min
    }
}
```

**Testing**: unit-test delegates/handlers; integration-test BPMN paths.

---

## Saga Monitoring & Traceability

- Correlate steps with a **sagaId** across services/logs/traces.
- Persist **saga state** (current step, compensations done, failure cause).
- Expose admin endpoints to **query** saga status and **retrigger** steps safely.
- Use **idempotent compensations** (refund only once).

```sql
CREATE TABLE saga_instance (
  id uuid primary key,
  name text not null,
  state jsonb not null,
  updated_at timestamptz not null default now()
);
```

---

## OpenTelemetry — Advanced Tracing

### Context Propagation
- Use **W3C Trace Context** (`traceparent`, `tracestate`) across HTTP and messaging headers.
- For Kafka, propagate via record headers.

```java
Headers h = record.headers();
h.add("traceparent", currentTraceparent().getBytes(StandardCharsets.UTF_8));
```

### Custom Spans & Attributes
```java
Span span = tracer.spanBuilder("chargePayment")
    .setSpanKind(SpanKind.INTERNAL)
    .startSpan();
try (var scope = span.makeCurrent()) {
    span.setAttribute("order.id", orderId);
    span.setAttribute("amount", amount.doubleValue());
    charge();
    span.setStatus(StatusCode.OK);
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR);
    throw e;
} finally {
    span.end();
}
```

### Sampling & Cost
- Use **parent-based 10%** head sampling for prod; raise for incidents.
- Redact PII; use attribute filters; bound event size.

---

## Security Hardening — Checklist

**Transport & Identities**
- Enforce TLS 1.2+ everywhere; HSTS; secure cookies.
- Mutual TLS or **mTLS/sidecars** for service-to-service where feasible.
- OAuth2/OIDC with short‑lived JWTs; rotate refresh tokens; validate `aud`, `iss`, `exp`, `nbf`.

**Data Protection**
- Encrypt PII/PHI at rest (KMS); field-level encryption for highly sensitive data.
- Backups encrypted; tested restore procedures.

**Input/Output**
- Validate all inputs (`@Valid`); sanitize/encode outputs (XSS).
- Limit payload sizes; set timeouts; protect against SSRF with allowlists.

**Secrets**
- Store in Vault/KMS/Secrets Manager; rotate automatically; restrict RBAC.
- Never log secrets; scrub in log appenders.

**Authorization**
- Enforce **RBAC/ABAC**; tenant boundaries; row-level security where applicable.
- Audit logs for sensitive actions.

**Build/Deploy**
- SBOM (CycloneDX); dependency scanning; pin base images; minimal images.
- Sign artifacts (SLSA), verify in CI/CD; image vulnerability scans.
- Enable JVM security manager alternatives not recommended; use container seccomp/AppArmor.

---

## Reference Blueprints

**Payment Command Path**
- REST (idempotency key) → Service (Tx) → DB write + Outbox → CDC (Debezium) → Kafka → Downstream fulfillment → Trace across.

**High Reliability Consumer**
- Kafka listener (manual ack) → Inbox table (unique eventId) → process → produce result event → commit offset.

**Powerful Read Models**
- Kafka → Kafka Streams projections → compacted state stores → serve via gRPC/REST with keyset pagination and Redis L2.

---

## Final Takeaways
- Treat resilience, observability, idempotency as **first-class** design elements.
- Prefer **typed configuration**, **compile-time mappers**, **transactional outbox**, and **Inbox** dedup.
- Keep flows **traceable** end-to-end with OpenTelemetry. Security is continuous — automate checks in CI/CD.

---

# Java & Spring Master Notes

Production-grade Java 17 + Spring Boot 3 notları.
- [Core_Java](./01_Core_Java.md)
- [Spring_Framework](./02_Spring_Framework.md)
- [Spring_Boot_Microservices](./03_Spring_Boot_Microservices.md)
- [Messaging_Kafka_RabbitMQ](./04_Messaging_Kafka_RabbitMQ.md)
- [Caching_Redis](./05_Caching_Redis.md)
- [Data_Access_Performance](./06_Data_Access_Performance.md)
- [System_Design](./07_System_Design.md)
- [Clean_Code_Practices](./08_Clean_Code_Practices.md)
- [Advanced_Topics](./09_Advanced_Topics.md)


# Core Java — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [JVM Architecture & GC (JFR/JDK Tools)](#jvm-architecture-and-gc-jfrjdk-tools)
- [Java Memory Model (JMM)](#java-memory-model-jmm)
- [Concurrency Primitives & Patterns](#concurrency-primitives-and-patterns)
- [Collections & Streams](#collections-and-streams)
- [Exceptions & API Contracts](#exceptions-and-api-contracts)
- [Performance Hygiene](#performance-hygiene)
- [Examples](#examples)
    - [CompletableFuture with Timeout & Retry](#completablefuture-with-timeout-and-retry)
    - [Optimistic read with StampedLock](#optimistic-read-with-stampedlock)

---


## Overview
JVM iç yapısı, Java Memory Model, concurrency primitifleri, koleksiyonlar, generics, streams, exception ve performans notları. Hedef: **Java 17** prod seviyesinde sağlam temel.

## JVM Architecture & GC (JFR/JDK Tools)
- JIT (C2), on-stack replacement, escape analysis → gereksiz allocation azalt.
- GC seçenekleri: G1 (default), ZGC/Shenandoah (düşük latency gereksinimi).
- **JFR** ile method hotspot, alloc rate, safepoint süreleri izle.

## Java Memory Model (JMM)
- Görünürlük: `volatile` **yazar → okuyana flush**, reorder kısıtlar.
- Atomicity: `Atomic*`/`LongAdder` sayaçlar; `synchronized` monitor kilidi.
- Happens-before: lock acquire/release, thread start/join, volatile write→read.

## Concurrency Primitives & Patterns
- **Executors**: bounded thread pools; virtual threads için `Executors.newVirtualThreadPerTaskExecutor()`.
- **CompletableFuture**: compose/timeout; exceptional pipeline.
- **Locks**: `ReentrantLock`, `StampedLock` (optimistic read), `ReadWriteLock`.
- **Coordination**: `CountDownLatch`, `Semaphore`, `Phaser`.
- **Immutable DTO**: paylaşılan veride tercih.

## Collections & Streams
- `List/Set/Map` Big-O, iterasyon maliyeti; `ConcurrentHashMap` segmentless.
- Streams: **stateless** vs **stateful** ara işlemler, **parallel()** sadece CPU-bound saf fonksiyonlarda.

## Exceptions & API Contracts
- Checked sadece kurtarılabilir IO gibi durumlar; diğerleri unchecked.
- API sınırında problem sözleşmesi; stack trace sızdırma yok.

## Performance Hygiene
- Kısa ömürlü objeleri azalt; `StringBuilder`/`record` kullan.
- JMH ile mikro-benchmark; JFR ile gerçek yük profili.

## Examples

### CompletableFuture with Timeout & Retry
```java
static <T> T callWithRetry(Supplier<T> s, int max) {
  for (int i=1;;i++) {
    try { return CompletableFuture.supplyAsync(s).orTimeout(2, TimeUnit.SECONDS).join(); }
    catch (CompletionException e) { if (i>=max) throw e; }
  }
}
```

### Optimistic read with StampedLock
```java
class Point {
  private double x,y; private final StampedLock sl = new StampedLock();
  double distance() {
    long s = sl.tryOptimisticRead();
    double cx = x, cy = y;
    if (!sl.validate(s)) { s = sl.readLock(); try { cx = x; cy = y; } finally { sl.unlockRead(s);} }
    return Math.hypot(cx, cy);
  }
}
```
---
**Next →** [Spring_Framework](02_Spring_Framework.md)


# Spring Framework — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Core DI & Lifecycle](#core-di-and-lifecycle)
- [AOP & Transactional Sınırları](#aop-and-transactional-snrlar)
- [Validation & Binding](#validation-and-binding)
- [Events & Observers](#events-and-observers)
- [Profiles & Conditional Beans](#profiles-and-conditional-beans)
- [Examples](#examples)
    - [Self Invocation Trap](#self-invocation-trap)
    - [Typed Config with Validation](#typed-config-with-validation)

---


## Overview
DI konteyneri, yaşam döngüsü, AOP, @ConfigurationProperties, profil ve event yapısı. Hedef: **Spring 6 / Boot 3** ile temiz, test edilebilir bean'ler.

## Core DI & Lifecycle
- `@Configuration` + `@Bean` vs component scanning; explicit > implicit.
- Bean lifecycle: post-processors → `@PostConstruct`/`InitializingBean` → `SmartLifecycle`.
- Scope: singleton (default), prototype, request/session (web).

## AOP & Transactional Sınırları
- Proxy tabanlı: **self-invocation** tuzağı (aynı bean içinden çağrı → advice çalışmaz).
- `@Transactional` sadece **public** methodlarda ve proxy üzerinden etkin.

## Validation & Binding
- `@ConfigurationProperties` + `@Validated` ile typed config.
- Controller girişinde `@Valid`; method seviyesinde `@Validated`.

## Events & Observers
- `ApplicationEventPublisher` ile domain event köprüsü (outbox’a giden yol).
- Async event için `@Async` + ayrı executor.

## Profiles & Conditional Beans
- `@Profile("prod")`/`@ConditionalOnProperty` ile çevresel varyantlar.
- Default değerleri güvenli belirle; fail-fast yapılandır.

## Examples

### Self Invocation Trap
```java
@Service
class A {
  @Transactional public void m1(){ m2(); } // m2 transactional DEĞİL
  @Transactional public void m2(){}
}
```

### Typed Config with Validation
```java
@Validated @ConfigurationProperties("email")
public record EmailProps(@Email String from, @Min(1) int poolSize) {}
```
---
**← Previous:** [Core_Java](01_Core_Java.md)  
**Next →** [Spring_Boot_Microservices](03_Spring_Boot_Microservices.md)


# Spring Boot Microservices — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Configuration & Secrets](#configuration-and-secrets)
- [Health & Metrics](#health-and-metrics)
- [HTTP Client (WebClient)](#http-client-webclient)
- [API Gateway](#api-gateway)
- [Observability](#observability)
- [Containerization](#containerization)
- [Examples](#examples)
    - [WebClient with Timeouts](#webclient-with-timeouts)
    - [Actuator Readiness Group](#actuator-readiness-group)

---


## Overview
Boot auto-config, config management, health/metrics, gateway, resilience, containerization, config-per-env, feature flags.

## Configuration & Secrets
- `@ConfigurationProperties` + profiles; secrets Vault/Secrets Manager.
- Immutable config objeleri; eksik config → fail-fast.

## Health & Metrics
- Actuator: health groups, readiness/liveness; Micrometer → Prometheus.
- Golden signals: latency p95/p99, error rate, saturation.

## HTTP Client (WebClient)
- Connect/read timeouts, pool limits; Resilience4j ile timeout/retry/circuit.

## API Gateway
- Rate limiting (Redis), auth offloading, request/response rewrite.

## Observability
- OpenTelemetry (OTLP) exporter; traceId propagation.

## Containerization
- Layered jars; distroless image; read-only FS; non-root user.

## Examples

### WebClient with Timeouts
```java
@Bean WebClient client(HttpClient hc){
 return WebClient.builder().clientConnector(new ReactorClientHttpConnector(hc))
        .build();
}
```

### Actuator Readiness Group
```yaml
management.endpoint.health.group.readiness.include: db,kafka,redis
```
---
**← Previous:** [Spring_Framework](02_Spring_Framework.md)  
**Next →** [Messaging_Kafka_RabbitMQ](04_Messaging_Kafka_RabbitMQ.md)

# Messaging — Kafka & RabbitMQ (Senior Developer Edition)

## Table of Contents
- [Overview](#overview)
- [RabbitMQ](#rabbitmq)
- [Kafka](#kafka)
- [Examples](#examples)

---


## Overview
Asenkron iletişim, delivery semantics, idempotency, retry/DLT, publisher confirms, outbox/inbox.

## RabbitMQ
- Exchange→Queue→Consumer; manual ACK, prefetch.
- TTL, DLX, DLQ ile gecikmeli tekrar ve zehirli mesaj park etme.
- Publisher confirms + returns ile güvenli yayın.

## Kafka
- Topics/partitions/offsets; consumer groups ve rebalance.
- Idempotent producer + transactions (EOS).
- Retry topics + DLT; schema registry ile evrim.

## Examples
```java
@RabbitListener(queues="orders.q", concurrency="3-10") /* ... */
```
---
**← Previous:** [Spring_Boot_Microservices](03_Spring_Boot_Microservices.md)  
**Next →** [Caching_Redis](05_Caching_Redis.md)


# Caching & Redis — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Spring Cache](#spring-cache)
- [Redis](#redis)
- [Patterns](#patterns)
- [Examples](#examples)

---


## Overview
Spring Cache, Caffeine+Redis multi-level, TTL/LRU/LFU, stampede/avalanche, Redisson locks.

## Spring Cache
- @Cacheable/@Put/@Evict; transaction-aware proxy.

## Redis
- Json serializer; Pub/Sub invalidation; cluster/sentinel.

## Patterns
- Negative cache, jitter TTL, single-flight.

## Examples
```java
@Cacheable(cacheNames="prices", key="#sku", unless="#result==null")
BigDecimal getPrice(String sku){/*...*/}
```
---
**← Previous:** [Messaging_Kafka_RabbitMQ](04_Messaging_Kafka_RabbitMQ.md)  
**Next →** [Data_Access_Performance](06_Data_Access_Performance.md)


# Data Access & Performance — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Mappings](#mappings)
- [N+1 Önleme](#n1-nleme)
- [Pagination](#pagination)
- [Transactions & Locking](#transactions-and-locking)
- [Examples](#examples)

---


## Overview
JPA/Hibernate ilişkiler, N+1, fetch stratejisi, projections, batch/bulk, pagination, locking, Hikari, migrations, plans.

## Mappings
- LAZY varsayılan; ManyToOne'u LAZY yap.
- ManyToMany yerine join entity.

## N+1 Önleme
- fetch join, entity graph, batch size.

## Pagination
- offset vs keyset.

## Transactions & Locking
- @Transactional sınırları, optimistic/pessimistic.

## Examples
```java
@EntityGraph(attributePaths={"orders"})
Optional<Customer> findById(UUID id);
```
---
**← Previous:** [Caching_Redis](05_Caching_Redis.md)  
**Next →** [System_Design](07_System_Design.md)
,


# System Design — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Idempotency](#idempotency)
- [Backpressure](#backpressure)
- [Release](#release)
- [Observability](#observability)
- [Examples](#examples)

---


## Overview
CAP/BASE, consistency, idempotency, load balancing, backpressure, rate limiting, caching layers, sharding, releases, observability.

## Idempotency
- Keys + dedup store; outbox/inbox.

## Backpressure
- Queues, bulkheads, timeouts, circuit breakers.

## Release
- Blue/green, canary, rolling; DB forward/backward compatible.

## Observability
- RED/USE; tracing/logging.

## Examples
```yaml
management.endpoints.web.exposure.include: health,metrics,prometheus
```
---
**← Previous:** [Data_Access_Performance](06_Data_Access_Performance.md)  
**Next →** [Clean_Code_Practices](08_Clean_Code_Practices.md)


# Clean Code Practices — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Packaging](#packaging)
- [Exceptions](#exceptions)
- [Logging](#logging)
- [Testing](#testing)
- [Examples](#examples)

---


## Overview
SOLID, hexagonal, mapping, validation, exceptions, logging, testing, CI/CD.

## Packaging
- package-by-feature + ports/adapters.

## Exceptions
- Domain → unchecked; map to problem details.

## Logging
- JSON logs, MDC correlation ids.

## Testing
- Unit/slice/integration/e2e pyramid.

## Examples
```java
record ApiError(String code, String message){}
```
---
**← Previous:** [System_Design](07_System_Design.md)  
**Next →** [Advanced_Topics](09_Advanced_Topics.md)



# Advanced Topics — Senior Developer Edition

## Table of Contents
- [Overview](#overview)
- [Examples](#examples)

---


## Overview
Resilience4j ileri, MapStruct ileri, Kafka retry/DLT, Virtual Threads, Camunda/Workflow, Saga monitoring, OpenTelemetry, Security hardening.

## Examples
```java
@Bean Executor vthreads(){ return Executors.newVirtualThreadPerTaskExecutor(); }
```
---
**← Previous:** [Clean_Code_Practices](08_Clean_Code_Practices.md)