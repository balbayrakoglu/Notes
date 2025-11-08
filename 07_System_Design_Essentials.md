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
