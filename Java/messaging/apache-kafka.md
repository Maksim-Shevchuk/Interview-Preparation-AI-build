# Apache Kafka

A distributed event streaming platform used for high-throughput, fault-tolerant, real-time data pipelines and
event-driven architectures. One of the most asked backend topics for senior interviews — interviewers expect you to
understand the architecture, not just the Spring API.

## Core Concepts

```
Producer ──▶ Kafka Cluster ──▶ Consumer
                │
         ┌──────┴──────┐
         │   Broker 1  │
         │   Broker 2  │   ← cluster of brokers
         │   Broker 3  │
         └─────────────┘
```

### Key Terms

| Term              | Description                                                              |
|-------------------|--------------------------------------------------------------------------|
| **Broker**        | A single Kafka server. Cluster = multiple brokers                        |
| **Topic**         | A named category/feed of messages. Like a table in a DB                  |
| **Partition**     | A topic is split into partitions — the unit of parallelism and ordering  |
| **Offset**        | Sequential ID of a message within a partition (immutable, monotonically increasing) |
| **Producer**      | Publishes messages to topics                                             |
| **Consumer**      | Reads messages from topics                                               |
| **Consumer Group**| A set of consumers that cooperatively read a topic (each partition assigned to one consumer in the group) |
| **Replica**       | Copy of a partition on another broker for fault tolerance                 |
| **Leader**        | The replica that handles all reads/writes for a partition                 |
| **Follower**      | Replicas that replicate from the leader. Promoted to leader if leader fails |
| **ZooKeeper / KRaft** | Cluster metadata management. KRaft (Kafka Raft) replaces ZooKeeper in modern Kafka (3.3+) |

## Architecture

### Topics and Partitions

```
Topic: "orders" (3 partitions, replication factor = 2)

Partition 0:  [msg0, msg1, msg2, msg3, msg4, ...]  → Broker 1 (leader), Broker 2 (follower)
Partition 1:  [msg0, msg1, msg2, ...]               → Broker 2 (leader), Broker 3 (follower)
Partition 2:  [msg0, msg1, msg2, msg3, ...]         → Broker 3 (leader), Broker 1 (follower)
```

**Key properties of partitions:**
- **Ordering is guaranteed within a partition** — not across partitions.
- **Parallelism unit** — more partitions = more consumers can read in parallel.
- Messages are **appended** (immutable log). Old messages are retained based on retention policy (time or size).
- Each consumer in a consumer group reads from **exclusive partitions** — no two consumers in the same group read the
  same partition.

### How Partition Assignment Works

```
Topic "orders" — 6 partitions
Consumer Group "order-service" — 3 consumers

Consumer 1: Partition 0, 1
Consumer 2: Partition 2, 3
Consumer 3: Partition 4, 5

If Consumer 3 dies → rebalance:
Consumer 1: Partition 0, 1, 4
Consumer 2: Partition 2, 3, 5
```

**Max useful consumers = number of partitions.** If you have 6 partitions and 8 consumers, 2 consumers will be idle.

## Producers

### Message Structure

```
Record {
    topic:     "orders"
    partition: 2                    // explicit or computed from key
    key:       "user-42"            // determines partition (nullable)
    value:     { "orderId": 123 }   // the payload (bytes)
    headers:   { "traceId": "abc" } // metadata
    timestamp: 1719872400000
}
```

### Partition Strategy

| Strategy              | How it works                                    | Ordering guarantee          |
|-----------------------|-------------------------------------------------|-----------------------------|
| Key is `null`         | Round-robin (or sticky partitioner in batches)  | No ordering                 |
| Key is set            | `hash(key) % numPartitions`                     | Same key → same partition → ordered |
| Custom partitioner    | Your logic                                      | Depends on implementation   |

**Common pattern:** Use entity ID as key (e.g., `userId`, `orderId`) to guarantee ordering for that entity.

```java
// All messages for user-42 go to the same partition → ordered
producer.send(new ProducerRecord<>("orders", "user-42", orderJson));
```

### Delivery Guarantees (Producer Side)

Controlled by `acks` configuration:

| `acks`   | Guarantee           | Latency | Description                                           |
|----------|---------------------|---------|-------------------------------------------------------|
| `0`      | Fire and forget     | Lowest  | Don't wait for any acknowledgment. May lose messages  |
| `1`      | Leader acknowledged | Medium  | Wait for leader to write. May lose if leader crashes before replication |
| `all`/`-1` | All in-sync replicas| Highest | Wait for all ISR to write. **No data loss** (with `min.insync.replicas >= 2`) |

**Production default:** `acks=all` + `min.insync.replicas=2` + `replication.factor=3`.

### Idempotent Producer

```properties
enable.idempotence=true   # default since Kafka 3.0
```

Each producer gets a **Producer ID** + **sequence number** per partition. The broker detects and deduplicates
retries → **exactly-once per partition** (no duplicates even on retry).

### Batching and Compression

```properties
batch.size=16384          # batch up to 16 KB before sending
linger.ms=5               # wait up to 5ms to fill a batch
compression.type=snappy   # compress batches (snappy, gzip, lz4, zstd)
```

Larger batches + compression = higher throughput, slightly higher latency.

## Consumers

### Consumer Group

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");       // consumer group
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");   // or "latest"

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record.key(), record.value(), record.offset(), record.partition());
    }
}
```

### Offset Management

The consumer tracks **where it left off** via offsets. Two strategies:

#### Auto-Commit (Default)

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000  # commit every 5 seconds
```

**Risk:** If the consumer crashes after processing but before auto-commit, messages are **reprocessed** (at-least-once).
If it crashes after auto-commit but before processing, messages are **lost** (at-most-once).

#### Manual Commit (Recommended for Production)

```java
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
    consumer.commitSync(); // commit after successful processing → at-least-once
}
```

| Commit strategy        | Semantics         | Risk                                    |
|------------------------|-------------------|-----------------------------------------|
| Auto-commit            | At-most-once      | Messages lost if crash before processing|
| Commit after processing| At-least-once     | Messages reprocessed on crash (most common) |
| Exactly-once           | Exactly-once      | Requires transactions (see below)       |

### `auto.offset.reset`

What to do when the consumer group has **no committed offset** (new group or offset expired):

| Value      | Behavior                                        |
|------------|-------------------------------------------------|
| `earliest` | Start from the beginning of the topic           |
| `latest`   | Start from the latest message (skip history)    |
| `none`     | Throw exception if no offset exists             |

### Consumer Rebalancing

Triggered when: a consumer joins/leaves the group, a consumer crashes (heartbeat timeout), partitions are added.

**Rebalancing strategies:**
- **Eager (default before 2.4)** — revoke all partitions, reassign. All consumers stop briefly.
- **Cooperative (incremental)** — only revoke partitions that need to move. Less disruption.

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

## Delivery Semantics

| Semantics       | How achieved                                                          |
|-----------------|-----------------------------------------------------------------------|
| **At-most-once** | Commit offset before processing. If crash → message skipped          |
| **At-least-once**| Commit offset after processing. If crash → message reprocessed. **Make consumers idempotent** |
| **Exactly-once** | Kafka transactions + idempotent producer + `read_committed` isolation|

### At-Least-Once + Idempotent Consumer (Most Common Pattern)

```java
// Consumer processes messages and writes to DB
// Use a unique message ID (or offset) to deduplicate
void process(ConsumerRecord<String, String> record) {
    String messageId = record.topic() + "-" + record.partition() + "-" + record.offset();

    if (processedMessageRepository.existsById(messageId)) {
        return; // already processed — skip (idempotent)
    }

    Order order = deserialize(record.value());
    orderService.processOrder(order);
    processedMessageRepository.save(messageId); // mark as processed
}
```

### Exactly-Once Semantics (Transactions)

```java
producer.initTransactions();

try {
    producer.beginTransaction();

    // Produce to output topic
    producer.send(new ProducerRecord<>("output-topic", key, value));

    // Commit consumer offsets as part of the transaction
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**Exactly-once works within Kafka** (consume → process → produce). For Kafka-to-external-system, use at-least-once +
idempotent consumer.

## Kafka Internals

### Log Storage

```
/kafka-data/orders-0/          # topic "orders", partition 0
├── 00000000000000000000.log    # segment file (messages)
├── 00000000000000000000.index  # offset → file position index
├── 00000000000000000000.timeindex  # timestamp → offset index
├── 00000000000005242880.log    # next segment (after 1GB or time-based roll)
└── ...
```

- Messages are **appended** to the active segment.
- Old segments are deleted or compacted based on retention policy.
- **Log compaction** — retains only the latest value per key (useful for changelogs, state snapshots).

### Retention

```properties
log.retention.hours=168         # delete messages older than 7 days
log.retention.bytes=1073741824  # or when partition exceeds 1 GB
log.cleanup.policy=delete       # "delete" (time/size) or "compact" (keep latest per key) or "compact,delete"
```

### Replication and ISR

```
Partition 0: Leader (Broker 1) → Follower (Broker 2) → Follower (Broker 3)
                                   ISR                    ISR
```

**ISR (In-Sync Replicas)** — replicas that are caught up with the leader. If a follower falls behind, it's removed
from ISR. `min.insync.replicas` defines how many ISR must acknowledge a write for `acks=all`.

**Leader election:** If the leader crashes, a follower from ISR is promoted. If no ISR available and
`unclean.leader.election.enable=true`, a non-ISR follower can become leader (risking data loss).

## Spring Kafka

### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publish(OrderEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Failed to send: {}", event, ex);
                    } else {
                        log.info("Sent to partition {} offset {}",
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
```

### Consumer

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(
            topics = "order-events",
            groupId = "payment-service",
            concurrency = "3"  // 3 consumer threads
    )
    public void onOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        log.info("Received from partition {} offset {}: {}", partition, offset, event);
        paymentService.processOrder(event);
    }
}
```

### Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
    consumer:
      group-id: payment-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.example.events"
    listener:
      ack-mode: RECORD  # commit after each record (manual per-record)
```

### Error Handling and Retry

```java
@Bean
public DefaultErrorHandler errorHandler() {
    // Retry 3 times with 1s backoff, then send to DLT (dead letter topic)
    return new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3)
    );
}
```

**Dead Letter Topic (DLT):** Messages that fail after all retries are sent to `<topic>.DLT`. Monitor DLT for
investigation and manual reprocessing.

### Manual Acknowledgment

```java
@KafkaListener(topics = "orders", groupId = "order-processor")
public void process(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
    try {
        orderService.process(record.value());
        ack.acknowledge(); // manually commit offset
    } catch (TransientException e) {
        // don't ack → message will be redelivered
        throw e;
    }
}
```

```yaml
spring:
  kafka:
    listener:
      ack-mode: MANUAL  # or MANUAL_IMMEDIATE
```

## Kafka vs RabbitMQ

| Aspect              | Kafka                                | RabbitMQ                             |
|---------------------|--------------------------------------|--------------------------------------|
| Model               | Distributed log (pull-based)         | Message broker (push-based)          |
| Message retention   | Retained after consumption           | Deleted after acknowledgment         |
| Ordering            | Per partition                        | Per queue                            |
| Throughput          | Millions msgs/sec                    | Tens of thousands msgs/sec           |
| Consumer model      | Pull (consumer polls)                | Push (broker delivers)               |
| Replay              | ✅ Re-read from any offset           | ❌ Messages gone after ack           |
| Routing             | Topic + partition key                | Exchanges + routing keys + bindings  |
| Protocol            | Custom binary                        | AMQP (standard)                      |
| Use case            | Event streaming, log aggregation, high-throughput | Task queues, RPC, complex routing |

**Choose Kafka for:** High throughput, event sourcing, replay/audit, stream processing, log aggregation.
**Choose RabbitMQ for:** Task queues, complex routing, request-reply, lower throughput with rich features.

## Schema Registry (Brief)

Manages schemas (Avro, Protobuf, JSON Schema) for Kafka messages:

```
Producer ──schema──▶ Schema Registry ──validate──▶ Kafka
Consumer ──fetch schema──▶ Schema Registry ──deserialize──▶ Application
```

- **Schema evolution** — backward/forward compatibility enforcement.
- **Avro** — compact binary format, schema evolution built-in. Most common with Kafka.
- **Confluent Schema Registry** — stores schemas, validates compatibility on produce.

## Common Interview Questions

1. **What is Kafka and when to use it?** — A distributed event streaming platform. Use for high-throughput messaging,
   event sourcing, log aggregation, real-time data pipelines. Not for simple task queues (use RabbitMQ).
2. **What is a partition and why does it matter?** — A partition is an ordered, immutable log. Partitions enable
   parallelism (more partitions = more consumers) and ordering (messages with the same key go to the same partition).
3. **How is ordering guaranteed?** — Only within a partition. Use a message key (e.g., entity ID) so that related
   messages land in the same partition and are consumed in order.
4. **What are consumer groups?** — A set of consumers that share the work of reading a topic. Each partition is
   assigned to exactly one consumer in the group. Adding consumers increases parallelism up to the number of
   partitions.
5. **At-least-once vs exactly-once?** — At-least-once: commit offset after processing (risk of reprocessing on
   crash — make consumer idempotent). Exactly-once: Kafka transactions (consume + produce + commit in one atomic
   operation). Most systems use at-least-once + idempotent consumers.
6. **What is ISR?** — In-Sync Replicas — followers caught up with the leader. `acks=all` waits for all ISR to
   confirm. `min.insync.replicas` sets the minimum ISR count for writes to succeed.
7. **What happens during consumer rebalancing?** — Triggered when a consumer joins/leaves/crashes. Partitions are
   reassigned. During rebalancing, affected consumers pause. Use cooperative sticky assignor to minimize disruption.
8. **What is a Dead Letter Topic?** — A topic where messages that fail processing after all retries are sent. Used
   for monitoring, debugging, and manual reprocessing of failed messages.
9. **Kafka vs RabbitMQ?** — Kafka: high throughput, log retention, replay, event streaming. RabbitMQ: task queues,
   complex routing, push-based delivery, lower throughput. Kafka retains messages; RabbitMQ deletes after ack.

## Related

- [Microservices Architecture](../spring/cloud/microservices-architecture.md) — async communication, Saga pattern
- [Spring Cloud Ecosystem](../spring/cloud/spring-cloud-ecosystem.md) — Spring Cloud Stream abstraction
- [CompletableFuture](../concurrency/completable-future.md) — async processing of Kafka messages

## Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent — Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)
- [Spring for Apache Kafka Docs](https://docs.spring.io/spring-kafka/reference/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/)
