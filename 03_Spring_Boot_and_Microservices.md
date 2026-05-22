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

# Saga Pattern — Distributed Transactions

## Overview

The **Saga Pattern** is used to manage distributed transactions across multiple microservices.

In a monolithic application, one database transaction can usually handle the full business operation. In a microservice architecture, each service owns its own database. Because of that, one global `@Transactional` boundary cannot safely cover all services.

Instead of using one large distributed transaction, Saga breaks the business process into multiple **local transactions**.

Each service performs its own local transaction and then triggers the next step by publishing an event or receiving a command.

If one step fails, the system runs **compensation actions** for the previous successful steps.

> Saga does not perform a technical rollback across services.  
> Saga performs business-level compensation.

---

## Simple Example

Successful flow:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Charge Payment
    ↓
Confirm Order
```

Failure flow:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Payment Failed
    ↓
Release Inventory
    ↓
Cancel Order
```

In this case, `Release Inventory` and `Cancel Order` are compensation actions.

---

## Why Saga Is Needed

Assume we have these services:

```text
Order Service       → order_db
Inventory Service   → inventory_db
Payment Service     → payment_db
Shipping Service    → shipping_db
```

A common mistake is thinking this works as one transaction:

```java
@Transactional
public void createOrder(CreateOrderCommand command) {
    orderRepository.save(command.toOrder());

    inventoryClient.reserveInventory(command);
    paymentClient.chargePayment(command);
}
```

The problem is that `@Transactional` only manages the local database transaction of the current service.

It cannot rollback changes made inside:

```text
Inventory Service
Payment Service
Shipping Service
```

So if the payment call fails after inventory is reserved, the system needs a separate business action to release the inventory.

That is where Saga helps.

---

## Local Transaction vs Distributed Transaction

### Local Transaction

A **local transaction** belongs to one service and one database.

Example:

```text
Order Service saves order into order_db
```

### Distributed Transaction

A **distributed transaction** tries to coordinate multiple services and databases as one transaction.

Example:

```text
Order DB + Inventory DB + Payment DB
```

In microservices, distributed transactions are usually avoided because they increase coupling, complexity, latency, and operational risk.

Saga is a more practical alternative for most business workflows.

---

## Compensation Is Not Rollback

This is one of the most important Saga details.

Rollback means:

```text
Undo database changes inside the same transaction
```

Compensation means:

```text
Run another business operation to correct or reverse a previous action
```

Examples:

```text
Payment charged       → Refund payment
Inventory reserved    → Release inventory
Order created         → Cancel order
```

Compensation actions must also be **idempotent**, because they may be retried.

Example:

```java
public void releaseInventory(String orderId) {
    Reservation reservation = reservationRepository.findByOrderId(orderId);

    if (reservation.isReleased()) {
        return;
    }

    reservation.release();
    reservationRepository.save(reservation);
}
```

If the same compensation command is received twice, the system should not release inventory twice.

---

## Saga Styles

There are two common Saga styles:

1. **Orchestration**
2. **Choreography**

---

# Orchestration

## What Is Orchestration?

In **Orchestration**, a central component controls the workflow.

This component is usually called:

```text
Saga Orchestrator
Process Manager
Workflow Coordinator
```

The orchestrator decides:

```text
Which step should run next
What should happen on success
What should happen on failure
Which compensation should be triggered
When to retry
When to mark the Saga as failed
```

---

## Orchestration Code Example

```java
public void createOrderSaga(CreateOrderCommand command) {
    try {
        Order order = orderService.createOrder(command);

        inventoryService.reserveInventory(order);

        paymentService.chargePayment(order);

        orderService.confirmOrder(order);

    } catch (Exception ex) {
        paymentService.refundPayment(command);
        inventoryService.releaseInventory(command);
        orderService.cancelOrder(command);
    }
}
```

This code explains the idea, but production systems usually need a persistent Saga state.

---

## Orchestration Successful Flow

```text
1. Order Service creates an order with PENDING status
2. Orchestrator sends ReserveInventory command
3. Inventory Service reserves stock
4. Orchestrator receives InventoryReserved event
5. Orchestrator sends ChargePayment command
6. Payment Service charges the customer
7. Orchestrator receives PaymentCharged event
8. Orchestrator sends ConfirmOrder command
9. Order becomes CONFIRMED
```

---

## Orchestration Failure Flow

```text
1. Order is created
2. Inventory is reserved
3. Payment fails
4. Orchestrator starts compensation
5. Orchestrator sends ReleaseInventory command
6. Inventory is released
7. Order becomes CANCELLED
```

---

## Saga State Table

For orchestration, storing Saga state is very important.

Example table:

```text
saga_id
order_id
status
current_step
retry_count
last_error
created_at
updated_at
```

Example statuses:

```java
public enum SagaStatus {
    STARTED,
    ORDER_CREATED,
    INVENTORY_RESERVED,
    PAYMENT_CHARGED,
    ORDER_CONFIRMED,
    COMPENSATING,
    COMPENSATED,
    FAILED
}
```

Example records:

```text
saga_id    order_id    status              current_step
SAGA-1     ORD-1       PAYMENT_CHARGED     CONFIRM_ORDER
SAGA-2     ORD-2       COMPENSATING        RELEASE_INVENTORY
SAGA-3     ORD-3       FAILED              PAYMENT
```

This gives visibility into long-running business processes.

---

## Orchestration Pros

```text
+ Easier to understand
+ Central workflow visibility
+ Easier debugging
+ Better retry handling
+ Better compensation handling
+ Good for complex business flows
+ Easier auditability
```

---

## Orchestration Cons

```text
- The orchestrator can become too large
- Risk of creating a "god service"
- More process-level coupling
- Orchestrator must be highly reliable
```

---

# Choreography

## What Is Choreography?

In **Choreography**, there is no central orchestrator.

Each service reacts to events and publishes new events.

Example:

```text
Order Service publishes OrderCreated
Inventory Service listens and reserves stock
Payment Service listens and charges payment
Shipping Service listens and prepares delivery
```

---

## Choreography Successful Flow

```text
OrderCreated
    ↓
InventoryReserved
    ↓
PaymentCharged
    ↓
OrderConfirmed
```

---

## Choreography Failure Flow

```text
OrderCreated
    ↓
InventoryReserved
    ↓
PaymentFailed
    ↓
InventoryReleased
    ↓
OrderCancelled
```

---

## Choreography Code Example

Inventory service:

```java
@KafkaListener(topics = "order-created")
public void handle(OrderCreatedEvent event) {
    inventoryService.reserve(event.orderId(), event.items());

    kafkaTemplate.send(
        "inventory-reserved",
        new InventoryReservedEvent(event.orderId())
    );
}
```

Payment service:

```java
@KafkaListener(topics = "inventory-reserved")
public void handle(InventoryReservedEvent event) {
    try {
        paymentService.charge(event.orderId());

        kafkaTemplate.send(
            "payment-charged",
            new PaymentChargedEvent(event.orderId())
        );

    } catch (Exception ex) {
        kafkaTemplate.send(
            "payment-failed",
            new PaymentFailedEvent(event.orderId())
        );
    }
}
```

Compensation listener:

```java
@KafkaListener(topics = "payment-failed")
public void handle(PaymentFailedEvent event) {
    inventoryService.release(event.orderId());

    kafkaTemplate.send(
        "inventory-released",
        new InventoryReleasedEvent(event.orderId())
    );
}
```

---

## Choreography Pros

```text
+ No central coordinator
+ Services are loosely coupled
+ Natural fit with event-driven architecture
+ Works well with Kafka
+ Simple for small workflows
```

---

## Choreography Cons

```text
- Harder to understand the full flow
- Harder debugging
- Event chains can become messy
- Cyclic dependencies may appear
- Compensation logic is distributed
- Harder to monitor complex processes
```

---

# Orchestration vs Choreography

| Topic | Orchestration | Choreography |
|---|---|---|
| Coordinator | Central orchestrator | No central coordinator |
| Flow visibility | High | Lower |
| Coupling | Process-level coupling | Event-level coupling |
| Debugging | Easier | Harder |
| Compensation | Centralized | Distributed |
| Best for | Complex workflows | Simple event flows |
| Risk | God orchestrator | Event chaos |

---

# When to Choose Orchestration

Use orchestration when:

```text
- The workflow has many steps
- The business process is complex
- Compensation logic is important
- Retry and timeout handling are required
- Auditability is needed
- The current state of the process must be visible
- Debugging and monitoring are important
```

Example:

```text
Order
  → Inventory Reservation
  → Payment
  → Fraud Check
  → Invoice
  → Shipping
```

For this kind of flow, orchestration is usually easier to manage.

---

# When to Choose Choreography

Use choreography when:

```text
- The workflow is simple
- There are only a few steps
- Services are naturally event-driven
- Failure handling is not complex
- Loose coupling is more important than central visibility
```

Example:

```text
UserRegistered
    ↓
SendWelcomeEmail
    ↓
CreateDefaultSettings
```

This kind of flow can work well with choreography.

---

# Production Considerations

Saga alone is not enough for a production-grade system.

A real implementation should also consider:

```text
Idempotency
Retry mechanism
Dead Letter Queue
Outbox Pattern
Correlation ID
Saga state table
Timeout handling
Compensation tracking
Observability
Monitoring and alerting
```

---

## Idempotency

In event-driven systems, the same event can be consumed more than once.

For example, Kafka provides at-least-once delivery by default in many setups.

That means this can be dangerous:

```java
public void chargePayment(OrderCreatedEvent event) {
    paymentService.charge(event.orderId());
}
```

If the same event is processed twice, the customer may be charged twice.

Better:

```java
public void chargePayment(OrderCreatedEvent event) {
    if (paymentRepository.existsByOrderId(event.orderId())) {
        return;
    }

    paymentService.charge(event.orderId());
}
```

The operation should be safe to repeat.

---

## Outbox Pattern

A common problem is this:

```text
Database transaction succeeds
Kafka publish fails
```

Example:

```java
orderRepository.save(order);
kafkaTemplate.send("order-created", event);
```

If the order is saved but the event is not published, other services will not know about the order.

The **Outbox Pattern** solves this problem.

Flow:

```text
1. Save the order
2. Save the event into an outbox table in the same database transaction
3. A background publisher reads the outbox table
4. The publisher sends the event to Kafka
5. The event is marked as SENT
```

Example:

```java
@Transactional
public void createOrder(CreateOrderCommand command) {
    Order order = orderRepository.save(Order.create(command));

    outboxRepository.save(
        OutboxEvent.create(
            "OrderCreated",
            order.getId(),
            toJson(order)
        )
    );
}
```

This keeps the database write and event creation atomic.

---

## Retry and DLQ

Saga steps can fail because of temporary issues:

```text
Network timeout
Payment provider unavailable
Database lock
Kafka broker issue
```

For temporary failures, retry can help.

But retry should not continue forever.

A common approach:

```text
1. Retry with backoff
2. If still failing, send to DLQ
3. Mark Saga as FAILED or WAITING_FOR_MANUAL_REVIEW
4. Alert the team
```

Example statuses:

```text
RETRYING
FAILED
WAITING_FOR_MANUAL_REVIEW
COMPENSATING
COMPENSATED
```

---

## Correlation ID

Every Saga should have a correlation ID.

This ID connects logs, events, commands, and database records.

Example:

```text
correlationId = orderId or sagaId
```

This helps answer:

```text
Which events belong to the same business transaction?
Where did the flow fail?
Which service processed the event?
Which compensation was triggered?
```

---

# Interview Answer

A good interview answer:

> The Saga Pattern is used to manage distributed transactions across microservices. Instead of using one global transaction, each service performs its own local transaction. After a successful step, the next step is triggered by a command or an event. If one step fails, previous successful steps are compensated with business-level rollback actions, such as refunding a payment or releasing inventory. There are two main styles: orchestration and choreography. In orchestration, a central orchestrator controls the workflow, which is better for complex flows. In choreography, services react to events without a central coordinator, which works better for simpler event-driven flows. In production, Saga should be combined with idempotency, retries, DLQ, correlation IDs, observability, and often the Outbox Pattern.

---

# Key Takeaways

```text
- Saga is used for distributed business transactions
- Each step is a local transaction
- Saga does not rollback like a database transaction
- Saga uses compensation actions
- Orchestration gives better control and visibility
- Choreography gives looser coupling
- Complex workflows usually fit orchestration better
- Simple event flows can use choreography
- Idempotency is mandatory
- Outbox Pattern helps prevent DB and Kafka inconsistency
- Retry, DLQ, and monitoring are required in production
```


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


## Redis in a Microservice Architecture

Redis is usually not the source of truth. In a Spring Boot microservice, I use it as a distributed cache, rate-limit counter, short-lived dedup store, session/token store, or pub/sub mechanism for cache invalidation.

### Typical Spring Boot setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```java
@EnableCaching
@SpringBootApplication
class OrdersApplication { }
```

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 30m
      cache-null-values: false
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2s
```

### Service-layer usage

```java
@Service
public class ProductQueryService {
    private final ProductRepository repository;

    public ProductQueryService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(cacheNames = "products", key = "#id")
    public ProductDto getProduct(UUID id) {
        return repository.findDtoById(id)
            .orElseThrow(() -> new NotFoundException("Product not found"));
    }

    @CacheEvict(cacheNames = "products", key = "#id")
    public void evictProduct(UUID id) {
        // called after product update/delete
    }
}
```

### Interview explanation

“In microservices, Redis is an optimization layer, not the system of record. I first write to the database, then update or evict cache entries. I define TTLs per domain, disable null caching unless I intentionally use negative caching, and monitor hit/miss ratio and memory usage. For multi-node deployments, if I use local L1 cache plus Redis L2, I publish invalidation events so every node evicts stale local entries.”

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
---

## Interview Add-on: Redis in a Spring Boot Microservice

This section connects the microservice architecture notes with the dedicated Redis/Caching chapter. In interviews, Redis is usually not discussed only as a library dependency; it is discussed as a design choice for latency, scalability, rate limiting, session/token state, distributed locks, cache invalidation, and failure handling.

### Typical Spring Boot integration flow

1. Add dependencies:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

2. Enable caching:

```java
@EnableCaching
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

3. Configure Redis connection and TTLs in `application.yml`:

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 30m
      cache-null-values: false
  data:
    redis:
      host: localhost
      port: 6379
```

4. Use cache annotations in the service layer, not in the controller:

```java
@Service
public class ProductQueryService {

    private final ProductRepository repository;

    public ProductQueryService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(cacheNames = "products", key = "#id")
    public ProductDto findById(UUID id) {
        return repository.findDtoById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
    }
}
```

### How to explain it in an interview

“I use Redis when the data is read frequently, expensive to calculate or fetch, and can tolerate short-lived staleness. I configure TTLs per cache, disable null caching unless I intentionally want negative caching, and evict/update the cache on writes. For distributed systems, I also think about cache invalidation across nodes, metrics, serialization format, and what happens when Redis is unavailable. Redis improves latency, but the database remains the source of truth unless the use case is explicitly Redis-native, such as counters, rate limiting, or distributed locks.”

