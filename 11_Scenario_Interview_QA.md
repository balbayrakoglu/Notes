# Scenario-Based Interview Q&A

> Short, realistic prompts with concise, senior-level answers. Tailored for Java 17 / Spring Boot 3, microservices, payments, and data platforms.

## 1) Design a High-Availability Feature in a Cloud-Native System
**Prompt:** You’re adding a new “scheduled payouts” feature to a payments platform. It must be scalable and fault-tolerant.
**Answer (summary):**
- **Data model:** Payout, PayoutRun, PayoutItem; state machine with immutable events.
- **Scalability:** Stateless workers behind an autoscaling Deployment; idempotent handlers using `(businessKey, step)` constraint.
- **Fault tolerance:** Outbox/Inbox + DLT; retries with backoff + jitter; TimeLimiter and CircuitBreaker per partner.
- **Storage:** Postgres primary + read replica; keyset pagination for large scans.
- **Reliability:** Blue/Green rollout; versioned contracts; shadow traffic.
- **Observability:** Traces over HTTP+Kafka; metrics per stage; audit log of state transitions.

## 2) Ensuring Idempotency (HTTP + MQ)
**Prompt:** How do you avoid duplicate charges in payment processing?
**Answer:**
- **HTTP writes:** `Idempotency-Key` header → store `(op,key) → result` with TTL; return cached result on replay.
- **Kafka:** Producer outbox with `eventId (UUID)`; consumer inbox table with unique `(eventId, handler)`. Ignore duplicates.
- **DB:** Unique `(paymentIntentId, state)` where applicable; use `UPSERT` (ON CONFLICT) patterns.

## 3) Concurrency Bug in Production
**Prompt:** Intermittent double-refunds are reported after a deploy.
**Answer:**
- **Stabilize:** Freeze deploys; enable feature flag to stop refunds.
- **Triage:** Query inbox/outbox for duplicate `refund.created` events; check logs by `traceId`.
- **Fix:** Add `(eventId, handler)` unique index; wrap refund handler in a transaction; commit offset after DB commit.
- **Backfill:** De-duplicate by latest state; replay DLT after patch.
- **Postmortem:** Add contract test + chaos test for at-least-once delivery.

## 4) Data Store Fault Tolerance
**Prompt:** How do you ensure fault tolerance for storage?
**Answer:**
- **RDBMS:** Synchronous replication (same AZ) + async cross-AZ; p99 latency budgets; connection pool guardrails.
- **Cache:** L1 Caffeine + L2 Redis (Sentinel/Cluster). Transaction-aware cache → only write after DB commit.
- **Backup/Restore:** PITR, immutable backups, restore drills, runbook for failover.
- **Schema:** Online migrations, `NULL`-tolerant rollouts, additive first.

## 5) Migrating to Virtual Threads
**Prompt:** You’re evaluating virtual threads in a service with blocking IO.
**Answer:**
- Replace server executors with virtual-thread pools.
- Audit ThreadLocal usage and remove thread-affinity assumptions.
- Keep bounded queues for backpressure; don’t over-parallelize CPU-bound work.
- Load test: compare throughput/latency vs platform threads.

## 6) Kafka Exactly-Once Processing (EOS)
**Prompt:** Process order events exactly once.
**Answer:**
- Enable idempotent producer + transactions; commit consumer offsets within the same transaction.
- Use a stable `eventId`; downstream deduplicate via inbox unique key.
- Avoid long transactions; keep batches small; monitor transactional aborts.

## 7) Designing Rate Limits & Backpressure
**Prompt:** Protect downstream partners with SLAs.
**Answer:**
- **Gateway:** Token-bucket limit per API key/tenant.
- **Service:** Bulkhead isolation per client; TimeLimiter; adaptive concurrency.
- **Queue:** Buffer with max TTL; shed load on saturation (429 + Retry-After).

## 8) JPA N+1 in Hot Endpoint
**Prompt:** A product listing endpoint is slow.
**Answer:**
- Replace entity graph with DTO projection query.
- Add `default_batch_fetch_size` and explicit fetch joins for stable paths.
- Cache read-only attributes; index FKs; verify `EXPLAIN ANALYZE` plan.

## 9) Blue/Green with Backward-Compatible DB
**Prompt:** Deploy a new column used by v2 only.
**Answer:**
1) Add nullable column; write by v2, ignore in v1.
2) Deploy app v2; dual-write if needed.
3) Backfill in background.
4) Flip reads to new column.
5) Remove legacy column in a later migration.

## 10) Troubleshooting Stuck Outbox
**Prompt:** Outbox publisher lags.
**Answer:**
- Use `FOR UPDATE SKIP LOCKED` with small batches + index on `(status, created_at)`.
- Expose “oldest pending” metric; alert on lag.
- Quarantine poison events after N retries and page owners.

---

## Behavioral / Leadership Mini Q&A

**Q:** Your team resists adopting contract tests.  
**A:** Start with a single critical integration; show a production incident that would’ve been prevented; provide a starter kit and CI template; make it the golden path; celebrate the first green build.

**Q:** Handling a sev‑1 incident as on-call.  
**A:** Stabilize (feature flags/rate limit), communicate ETA and blast radius, form a bridge with roles (lead/scribe/commander), rollback if needed, verify with synthetic checks, publish a blameless postmortem with 3 concrete action items.

**Q:** Trade-off: Build vs Buy for observability.  
**A:** Buy collection & storage (vendor/OSS) to move fast; build SLOs, dashboards, and alerts as productized templates; keep data ownership and egress cost in mind.

---

## Java Core Live-Coding Scenarios

### 1) Implement `equals()` and `hashCode()`

**Prompt:** Implement `equals()` and `hashCode()` for an employee-like object.

```java
public final class Employee {
    private final Long id;
    private final String email;

    public Employee(Long id, String email) {
        this.id = id;
        this.email = email;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        if (!(obj instanceof Employee other)) {
            return false;
        }
        return Objects.equals(id, other.id)
            && Objects.equals(email, other.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, email);
    }
}
```

**How to explain it in interview:**
“I compare the same fields in both methods. `equals()` defines logical equality, while `hashCode()` helps hash-based collections find the right bucket. Equal objects must have the same hash code, but the same hash code does not guarantee equality. I would never use random values or mutable fields for hash code because the value must stay stable while the object is inside a `HashMap` or `HashSet`.”

### 2) Design an immutable class with defensive copy

**Prompt:** Create an immutable `Student` class with a list of grades.

```java
public final class Grade {
    private final String course;
    private final int score;

    public Grade(String course, int score) {
        this.course = Objects.requireNonNull(course);
        this.score = score;
    }

    public Grade(Grade other) {
        this(other.course, other.score);
    }
}

public final class Student {
    private final String name;
    private final List<Grade> grades;

    public Student(String name, List<Grade> grades) {
        this.name = Objects.requireNonNull(name);
        this.grades = grades.stream()
            .map(Grade::new)
            .toList();
    }

    public String getName() {
        return name;
    }

    public List<Grade> getGrades() {
        return grades.stream()
            .map(Grade::new)
            .toList();
    }
}
```

**How to explain it in interview:**
“The class is `final`, fields are `private final`, there are no setters, constructor inputs are validated, and mutable collections are copied. If the list contains immutable elements like `String`, `List.copyOf()` is usually enough. If the list contains mutable objects, I deep-copy each element in the constructor and getter.”

### 3) Use `HashMap.computeIfAbsent`

**Prompt:** Group employee names by department.

```java
Map<String, List<String>> employeesByDepartment = new HashMap<>();

for (EmployeeDto employee : employees) {
    employeesByDepartment
        .computeIfAbsent(employee.department(), key -> new ArrayList<>())
        .add(employee.name());
}
```

**How to explain it in interview:**
“`computeIfAbsent` checks whether the key already exists. If not, it creates the value once and stores it. It avoids verbose `containsKey/get/put` code and is a clean pattern for grouping. For concurrent counters, I would use `ConcurrentHashMap` with `LongAdder`.”

### 4) Explain Redis integration in Spring Boot

**Prompt:** How would you add Redis caching to a Spring Boot service?

**Answer:**
- Add `spring-boot-starter-cache` and `spring-boot-starter-data-redis`.
- Enable caching with `@EnableCaching`.
- Configure Redis host, TTL, serialization, and null-caching behavior.
- Use `@Cacheable` for reads, `@CachePut` for updates, and `@CacheEvict` for deletes.
- Use `StringRedisTemplate` directly for counters, rate limits, locks, and Redis-specific structures.
- In production, monitor hit ratio, latency, memory usage, evictions, and key cardinality.

### 5) How do you design Rest Api consistent Error response? 

- I design a single error response contract and enforce it through a global exception handler. 
- Each error contains an HTTP status, a stable internal error code, a safe user-facing message, request path, timestamp, and correlation ID. 
- Validation errors can include structured field-level details. 
- For unexpected errors, I return a generic message to the client and log the full technical details internally with the correlation ID.

**Sample Error body**
```Json
{
"timestamp": "2026-05-26T10:30:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/api/orders",
  "correlationId": "8f4f6e2a-1234-45b1-a19d-10ab92c91e22",
  "details": [
    {
    "field": "customer.email",
    "code": "INVALID_EMAIL",
    "message": "must be a valid email address",
    "rejectedValue": "abc"
    }
  ]
}
```

**SpringBoot centralized Solution**
```Java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiErrorResponse> handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request
    ) {
        List<FieldErrorDetail> details = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(error -> new FieldErrorDetail(
                        error.getField(),
                        error.getCode(),
                        error.getDefaultMessage(),
                        error.getRejectedValue()
                ))
                .toList();

        ApiErrorResponse response = new ApiErrorResponse(
                Instant.now(),
                400,
                "VALIDATION_ERROR",
                "Request validation failed",
                request.getRequestURI(),
                getCorrelationId(),
                details
        );

        return ResponseEntity.badRequest().body(response);
    }
}
```