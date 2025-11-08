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