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



<!-- ===== Auto-Appended from Readme1.md (missing sections) ===== -->


## JVM Architecture & GC (JFR/JDK Tools)

- JIT (C2), on-stack replacement, escape analysis → gereksiz allocation azalt.
- GC seçenekleri: G1 (default), ZGC/Shenandoah (düşük latency gereksinimi).
- **JFR** ile method hotspot, alloc rate, safepoint süreleri izle.



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



# Spring Boot Microservices — Senior Developer Edition




## Health & Metrics

- Actuator: health groups, readiness/liveness; Micrometer → Prometheus.
- Golden signals: latency p95/p99, error rate, saturation.



## HTTP Client (WebClient)

- Connect/read timeouts, pool limits; Resilience4j ile timeout/retry/circuit.



## Containerization

- Layered jars; distroless image; read-only FS; non-root user.



## Kafka

- Topics/partitions/offsets; consumer groups ve rebalance.
- Idempotent producer + transactions (EOS).
- Retry topics + DLT; schema registry ile evrim.



## Spring Cache

- @Cacheable/@Put/@Evict; transaction-aware proxy.



## Redis

- Json serializer; Pub/Sub invalidation; cluster/sentinel.



## Patterns

- Negative cache, jitter TTL, single-flight.



## Mappings

- LAZY varsayılan; ManyToOne'u LAZY yap.
- ManyToMany yerine join entity.



## N+1 Önleme

- fetch join, entity graph, batch size.



## Transactions & Locking

- @Transactional sınırları, optimistic/pessimistic.



## Backpressure

- Queues, bulkheads, timeouts, circuit breakers.



## Release

- Blue/green, canary, rolling; DB forward/backward compatible.



## Packaging

- package-by-feature + ports/adapters.



## Exceptions

- Domain → unchecked; map to problem details.



## Logging

- JSON logs, MDC correlation ids.



## Testing

- Unit/slice/integration/e2e pyramid.
