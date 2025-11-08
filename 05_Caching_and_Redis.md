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