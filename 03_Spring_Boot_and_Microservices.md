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
  try {
    reserveInventory(cmd);
    chargePayment(cmd);
    confirmOrder(cmd);
  } catch (Exception ex) {
    refundPayment(cmd);
    releaseInventory(cmd);
  }
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