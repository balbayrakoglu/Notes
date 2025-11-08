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
