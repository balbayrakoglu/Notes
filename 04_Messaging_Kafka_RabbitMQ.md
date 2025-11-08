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