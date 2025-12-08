# Quick Cheatsheets — Senior Developer Edition

> **Reference handy for interviews and daily ops.**

---

## 1. JVM & Performance Arguments
**Standard Container / Kubernetes (2GB Limit Example)**
```bash
java -XX:MaxRAMPercentage=75.0 \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -jar app.jar
```

**Debug & JMX**
```bash
-Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=9010 \
-Dcom.sun.management.jmxremote.authenticate=false \
-Dcom.sun.management.jmxremote.ssl=false
```

---

## 2. Spring Boot Common Properties
**Actuator & Metrics**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,loggers
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
```

**HikariCP (DB Pool) Tuning**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10   # Connections = ((core_count * 2) + effective_spindle_count)
      minimum-idle: 10        # Fixed pool size (min=max) prevents spike latency
      idle-timeout: 300000    # 5 mins
      max-lifetime: 1800000   # 30 mins (Must be < DB wait_timeout)
      connection-timeout: 30000 # 30s
```

**JPA Batch Processing**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          order_inserts: true
          order_updates: true
```

---

## 3. Key Annotations Reference
### Core
- `@PostConstruct` / `@PreDestroy`: Lifecycle hooks.
- `@ConfigurationProperties`: Type-safe config.
- `@Value("${prop}")`: Simple injection (prefer ConfigProps).

### Spring MVC & Validations
- `@RestControllerAdvice`: Global exception handling.
- `@Valid` (Boxed) vs `@Validated` (Class/Method level).
- `@NotBlank`, `@Size`, `@Min`, `@Pattern`, `@Email` (Jakarta Validation).

### Spring Cloud / Resilience
- `@Retry(name="backend")`
- `@CircuitBreaker(name="backend", fallbackMethod="fallback")`
- `@RateLimiter`
- `@Bulkhead`

### Testing
- `@SpringBootTest`: Full context (expensive).
- `@WebMvcTest`: Controller slice (fast).
- `@DataJpaTest`: Repository slice (H2 default).
- `@MockBean`: Replaces a bean in the context with a Mockito mock.

---

## 4. Docker & K8s Quick commands
**Docker**
```bash
# Clean up
docker system prune -a --volumes

# Run Redis quick
docker run -d --nameredis -p 6379:6379 redis:alpine

# Build with BuildKit
DOCKER_BUILDKIT=1 docker build -t myapp .
```

**Kubernetes (kubectl)**
```bash
# Debug pod
kubectl run -it --rm debug --image=curlimages/curl -- sh

# Logs with tail
kubectl logs -f -l app=payment-service

# Port forward
kubectl port-forward svc/payment-service 8080:80
```

---

## 5. Kafka Cheatsheet
**Producer Config (Safety)**
- `acks=all`: Durability.
- `enable.idempotence=true`: No dupes, order guaranteed.
- `retries=MAX_INT`: Keep trying.

**Consumer Config**
- `group.id`: Load balancing group.
- `auto.offset.reset=earliest`: Start from beginning if no offset.
- `enable.auto.commit=false`: Manual control for atomic processing.

**CLI**
```bash
# List topics
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Consume from beginning
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning --property print.key=true
```

---

## 6. SQL / JPA Quick Patterns
**Pessimistic Lock (Select for Update)**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdForUpdate(Long id);
```

**Keyset Pagination (No Offset)**
```sql
SELECT * FROM orders
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## 7. Git & Conventional Commits
- `feat: add payment gateway`
- `fix: handle null user in auth`
- `docs: update readme`
- `chore: bump spring boot version`
- `refactor: extract validation logic`
- `test: add integration test for carts`

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Amend last commit message
git commit --amend
```
