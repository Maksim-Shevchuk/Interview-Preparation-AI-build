# RabbitMQ

A mature, feature-rich **message broker** implementing the AMQP 0-9-1 protocol. Where Kafka is a distributed log for
high-throughput event streaming, RabbitMQ is a **smart broker** for complex routing, task queues, and request-reply
patterns. This note covers architecture, exchange types, reliability guarantees, and Spring AMQP integration.

## Core Concepts

```
Producer ──▶ Exchange ──routing──▶ Queue ──▶ Consumer
                │                    │
           Routing key          Binding key
           (on message)         (on queue)
```

### Key Terms

| Term            | Description                                                                |
|-----------------|----------------------------------------------------------------------------|
| **Producer**    | Sends messages to an **exchange** (never directly to a queue)              |
| **Exchange**    | Receives messages and routes them to queues based on **bindings**          |
| **Binding**     | Rule that links an exchange to a queue (with an optional binding key)      |
| **Queue**       | Buffer that stores messages until consumed. FIFO within a single queue     |
| **Consumer**    | Reads messages from a queue                                                |
| **Routing key** | A label on the message that the exchange uses for routing decisions        |
| **Virtual Host (vhost)** | Logical isolation — separate exchanges, queues, users per vhost   |
| **Connection**  | TCP connection between app and broker                                      |
| **Channel**     | Lightweight virtual connection inside a TCP connection (multiplexing)      |

### Connection vs Channel

```
Application ──TCP Connection──▶ RabbitMQ Broker
                  │
          ┌───────┼───────┐
          │       │       │
       Channel  Channel  Channel    ← multiplexed on one TCP connection
       (thread1)(thread2)(thread3)
```

- **One TCP connection** per application (expensive to create).
- **One channel per thread** (lightweight, not thread-safe — don't share across threads).
- Spring AMQP manages connection/channel pooling automatically.

## Exchange Types

### 1. Direct Exchange

Routes messages to queues whose **binding key exactly matches** the message's routing key.

```
Producer ──routingKey="order.created"──▶ Direct Exchange
                                            │
                                    ┌───────┴───────┐
                                    │               │
                            bindingKey=         bindingKey=
                            "order.created"     "order.cancelled"
                                    │               │
                                    ▼               ▼
                               Queue A          Queue B
```

```java
// Only Queue A receives this message
rabbitTemplate.convertAndSend("order-exchange", "order.created", orderEvent);
```

**Use case:** Task distribution, routing by exact event type.

**Default exchange:** A nameless direct exchange where every queue is auto-bound with its queue name as routing key.
`rabbitTemplate.convertAndSend("", "queue-name", message)` sends directly to a queue.

### 2. Fanout Exchange

Routes messages to **all bound queues**, ignoring the routing key.

```
Producer ──▶ Fanout Exchange
                  │
          ┌───────┼───────┐
          │       │       │
       Queue A  Queue B  Queue C    ← ALL get the message
```

```java
rabbitTemplate.convertAndSend("notifications-fanout", "", event); // routing key ignored
```

**Use case:** Broadcasting — send to all consumers (email service, push service, audit log).

### 3. Topic Exchange

Routes based on **pattern matching** between routing key and binding key using wildcards.

| Wildcard | Meaning                          | Example pattern         |
|----------|----------------------------------|-------------------------|
| `*`      | Matches exactly **one** word     | `order.*.created`       |
| `#`      | Matches **zero or more** words   | `order.#`               |

```
Producer ──routingKey="order.eu.created"──▶ Topic Exchange
                                               │
                                       ┌───────┼───────────┐
                                       │       │           │
                                  "order.#" "order.eu.*" "order.us.*"
                                       │       │           │
                                       ▼       ▼           ✗ (no match)
                                    Queue A  Queue B
```

```java
// Queue A (binding "order.#") and Queue B (binding "order.eu.*") both receive this
rabbitTemplate.convertAndSend("order-topic", "order.eu.created", event);
```

**Use case:** Flexible multi-criteria routing — geographic, event type, severity.

### 4. Headers Exchange

Routes based on **message headers** instead of routing key. Matches headers against binding arguments.

```java
// Binding: match headers { "format": "pdf", "type": "report" } with x-match=all
// Message must have ALL specified headers to be routed
```

- `x-match=all` — all headers must match.
- `x-match=any` — at least one header must match.

**Use case:** Rare. When routing key is insufficient and you need multi-attribute routing.

### Exchange Comparison

| Exchange  | Routing based on        | Use case                              |
|-----------|-------------------------|---------------------------------------|
| Direct    | Exact routing key match | Point-to-point, task queues           |
| Fanout    | Nothing (broadcast)     | Pub-sub, notifications to all         |
| Topic     | Pattern matching (`*`, `#`) | Flexible pub-sub, multi-criteria  |
| Headers   | Message headers         | Complex multi-attribute routing       |

## Message Flow

### Publish

```
Producer → Channel → Exchange → Binding rules → Queue(s) → stored on disk/memory
```

### Consume

```
Queue → Channel → Consumer → ack/nack/reject → message removed or requeued
```

## Reliability and Delivery Guarantees

### Producer Confirmations (Publisher Confirms)

Ensure the broker **received** the message:

```java
// Spring AMQP — publisher confirms
spring:
  rabbitmq:
    publisher-confirm-type: correlated   # async confirms
    publisher-returns: true              # return unroutable messages
```

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.error("Message not confirmed: {}", cause);
        // retry or save to fallback store
    }
});

rabbitTemplate.setReturnsCallback(returned -> {
    log.warn("Message returned (unroutable): routingKey={}, replyText={}",
            returned.getRoutingKey(), returned.getReplyText());
});
```

| Guarantee level       | Mechanism                                      |
|-----------------------|------------------------------------------------|
| Fire and forget       | No confirms, no persistence. May lose messages |
| Broker confirmed      | Publisher confirms — broker ACKs receipt        |
| Durable + confirmed   | Persistent message + durable queue + confirms  |

### Consumer Acknowledgments

Control when a message is removed from the queue:

```java
// AUTO — ack immediately on delivery (before processing). At-most-once.
// MANUAL — consumer explicitly acks after successful processing. At-least-once.
// NONE — no ack, message removed immediately. Fire-and-forget.
```

```java
@RabbitListener(queues = "orders")
public void process(Order order, Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
    try {
        orderService.process(order);
        channel.basicAck(deliveryTag, false);         // success → ack
    } catch (TransientException e) {
        channel.basicNack(deliveryTag, false, true);  // requeue for retry
    } catch (PermanentException e) {
        channel.basicNack(deliveryTag, false, false); // don't requeue → goes to DLX
    }
}
```

| Method        | Effect                                                       |
|---------------|--------------------------------------------------------------|
| `basicAck`    | Message processed successfully → remove from queue           |
| `basicNack`   | Processing failed. `requeue=true` → back to queue. `requeue=false` → discard or DLX |
| `basicReject`  | Same as `basicNack` but for one message only                |

### Durability Checklist (No Message Loss)

All three must be enabled:

1. **Durable exchange** — survives broker restart.
2. **Durable queue** — survives broker restart.
3. **Persistent message** — `deliveryMode=2` (written to disk).
4. **Publisher confirms** — producer knows broker received it.
5. **Manual ack** — consumer confirms processing.

```java
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("orders")   // durable queue
            .build();
}

@Bean
public DirectExchange orderExchange() {
    return ExchangeBuilder.directExchange("order-exchange")
            .durable(true)                  // durable exchange
            .build();
}

// Messages sent via rabbitTemplate are persistent by default in Spring AMQP
```

## Dead Letter Exchange (DLX)

When a message is rejected (`nack` without requeue), expires (TTL), or the queue exceeds its max length, it's routed
to a **Dead Letter Exchange**.

```
Main Queue ──reject/expire──▶ DLX ──▶ Dead Letter Queue
                                           │
                                     Monitor, investigate,
                                     reprocess manually
```

```java
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("orders")
            .deadLetterExchange("dlx-exchange")
            .deadLetterRoutingKey("orders.dead")
            .ttl(60000)                           // message TTL: 60s
            .maxLength(10000)                      // max queue size
            .build();
}

@Bean
public Queue deadLetterQueue() {
    return QueueBuilder.durable("orders.dlq").build();
}

@Bean
public DirectExchange dlxExchange() {
    return new DirectExchange("dlx-exchange");
}

@Bean
public Binding dlxBinding() {
    return BindingBuilder.bind(deadLetterQueue()).to(dlxExchange()).with("orders.dead");
}
```

### Retry with DLX (Delayed Requeue)

```
Main Queue ──reject──▶ DLX ──▶ Retry Queue (TTL=5s, no consumers)
                                     │
                              TTL expires
                                     │
                              DLX of retry queue = main exchange
                                     │
                                     ▼
                               Main Queue ──▶ Consumer (retry)
```

This creates a delay between retries without blocking the consumer.

## Prefetch (QoS)

Controls how many **unacknowledged messages** the broker sends to a consumer at once.

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10   # consumer receives max 10 unacked messages
```

| Prefetch   | Effect                                                              |
|------------|---------------------------------------------------------------------|
| `1`        | One message at a time. Fair distribution but lower throughput       |
| `10-50`    | Good balance for most applications                                 |
| `0` / unlimited | Broker sends everything. Risk of consumer OOM. Never in production |

**Fair dispatch:** With `prefetch=1`, a slow consumer doesn't get new messages until it acks the current one. Fast
consumers get more work. Essential for task queue patterns with heterogeneous consumers.

## Patterns

### Work Queue (Task Queue)

Multiple consumers share a queue — each message is processed by **one** consumer.

```
Producer ──▶ Queue ──▶ Consumer 1
                   ──▶ Consumer 2  ← round-robin by default
                   ──▶ Consumer 3
```

**Use case:** Background jobs, email sending, image processing.

### Pub/Sub (Fanout)

Every consumer gets every message (via its own queue).

```
Producer ──▶ Fanout Exchange ──▶ Queue 1 ──▶ Consumer A (email)
                               ──▶ Queue 2 ──▶ Consumer B (push)
                               ──▶ Queue 3 ──▶ Consumer C (audit)
```

### Request-Reply (RPC)

Synchronous-style communication over async messaging:

```
Client ──request──▶ Request Queue ──▶ Server
Client ◀──reply──── Reply Queue ◀──── Server
            (correlationId matches request to reply)
```

```java
// Spring AMQP — synchronous RPC
Object response = rabbitTemplate.convertSendAndReceive("rpc-exchange", "rpc.key", request);

// Server side
@RabbitListener(queues = "rpc-queue")
public Result handleRpc(Request request) {
    return computeResult(request); // return value is sent to reply queue
}
```

### Priority Queue

```java
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable("priority-tasks")
            .maxPriority(10)   // priority 0-10
            .build();
}

// Send with priority
rabbitTemplate.convertAndSend("exchange", "key", message, msg -> {
    msg.getMessageProperties().setPriority(8); // higher = processed first
    return msg;
});
```

## Spring AMQP

### Configuration

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    listener:
      simple:
        acknowledge-mode: manual   # AUTO, MANUAL, NONE
        prefetch: 10
        concurrency: 3             # min consumer threads
        max-concurrency: 10        # max consumer threads
        retry:
          enabled: true
          initial-interval: 1000   # 1s
          multiplier: 2            # exponential backoff
          max-attempts: 3
          max-interval: 10000      # cap at 10s
```

### Declaring Topology (Exchanges, Queues, Bindings)

```java
@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-exchange");
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable("order.created.queue")
                .deadLetterExchange("dlx-exchange")
                .build();
    }

    @Bean
    public Queue orderCancelledQueue() {
        return QueueBuilder.durable("order.cancelled.queue").build();
    }

    @Bean
    public Binding createdBinding() {
        return BindingBuilder
                .bind(orderCreatedQueue())
                .to(orderExchange())
                .with("order.created");
    }

    @Bean
    public Binding cancelledBinding() {
        return BindingBuilder
                .bind(orderCancelledQueue())
                .to(orderExchange())
                .with("order.cancelled");
    }
}
```

### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(OrderEvent event) {
        rabbitTemplate.convertAndSend(
                "order-exchange",    // exchange
                "order.created",     // routing key
                event                // payload (serialized to JSON via Jackson2JsonMessageConverter)
        );
    }
}

// JSON serialization config
@Bean
public MessageConverter jsonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}

@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory cf, MessageConverter converter) {
    RabbitTemplate template = new RabbitTemplate(cf);
    template.setMessageConverter(converter);
    return template;
}
```

### Consumer

```java
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final OrderService orderService;

    @RabbitListener(queues = "order.created.queue")
    public void onOrderCreated(OrderEvent event) {
        // With AUTO ack: message acked after this method returns without exception
        // Exception → message requeued (or sent to DLX after retries)
        orderService.processNewOrder(event);
    }

    // Multiple queues
    @RabbitListener(queues = { "order.created.queue", "order.cancelled.queue" })
    public void onOrderEvent(OrderEvent event, @Header("amqp_receivedRoutingKey") String routingKey) {
        switch (routingKey) {
            case "order.created"   -> orderService.processNewOrder(event);
            case "order.cancelled" -> orderService.cancelOrder(event);
        }
    }
}
```

### Error Handling

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory, MessageConverter converter) {

    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(converter);
    factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
    factory.setPrefetchCount(10);

    // Retry policy
    factory.setAdviceChain(RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)  // initial, multiplier, max
            .recoverer(new RejectAndDontRequeueRecoverer()) // after 3 retries → DLX
            .build());

    return factory;
}
```

## Clustering and High Availability

### Clustering

Multiple RabbitMQ nodes form a cluster — share users, vhosts, exchanges, bindings, and **queue metadata** (but not
queue contents by default).

### Quorum Queues (Recommended for HA)

Replicated queues using the **Raft consensus** protocol (RabbitMQ 3.8+). Replace classic mirrored queues.

```java
@Bean
public Queue quorumQueue() {
    return QueueBuilder.durable("orders")
            .quorum()                        // replicated across cluster nodes
            .deliveryLimit(5)                // max redeliveries before DLX
            .build();
}
```

- Data replicated across N nodes (majority must be online for writes).
- Automatic leader election on node failure.
- **Recommended** over classic mirrored queues for production.

## RabbitMQ vs Kafka (Detailed)

| Aspect                | RabbitMQ                            | Kafka                               |
|-----------------------|-------------------------------------|--------------------------------------|
| **Model**             | Smart broker, simple consumer       | Dumb broker, smart consumer          |
| **Delivery**          | Push (broker → consumer)            | Pull (consumer → broker)             |
| **Message lifecycle** | Deleted after ack                   | Retained (by time/size policy)       |
| **Replay**            | ❌ Not possible after ack           | ✅ Re-read from any offset           |
| **Ordering**          | Per queue (FIFO)                    | Per partition                        |
| **Routing**           | Rich (exchanges, bindings, headers) | Simple (topic + partition key)       |
| **Throughput**        | ~10-50K msgs/sec per node           | ~1M+ msgs/sec per node              |
| **Latency**           | Very low (sub-ms)                   | Low but higher (batching)            |
| **Protocol**          | AMQP (standard)                     | Custom binary                        |
| **Request-Reply**     | ✅ Built-in (reply-to, correlation) | ❌ Not designed for it               |
| **Priority**          | ✅ Priority queues                  | ❌ No built-in priority              |
| **Consumer groups**   | Competing consumers on one queue    | Native consumer group concept        |
| **Use case**          | Task queues, RPC, complex routing   | Event streaming, log aggregation     |

**Choose RabbitMQ when:** Complex routing needed, request-reply (RPC), message priority, lower latency requirement,
task queues with fair dispatch.

**Choose Kafka when:** Very high throughput, event sourcing, need to replay/reprocess messages, log aggregation,
stream processing, audit trail.

## Management UI

RabbitMQ ships with a web-based management UI (`rabbitmq-plugins enable rabbitmq_management`):

- **Overview** — connections, channels, queues, message rates.
- **Queue details** — depth, consumers, message rates, purge.
- **Exchange details** — bindings, publish rates.
- **Publish/consume** messages manually (for debugging).
- **User/permission management.**

Default: `http://localhost:15672` (guest/guest).

## Common Interview Questions

1. **What is RabbitMQ and when to use it?** — A message broker implementing AMQP. Use for task queues, RPC,
   complex routing (topic/fanout/headers), when you need message priority, low latency, or request-reply. Not for
   high-throughput event streaming (use Kafka).
2. **What are the exchange types?** — Direct (exact key match), Fanout (broadcast to all queues), Topic (pattern
   matching with `*` and `#` wildcards), Headers (match on message headers). Producer sends to exchange, not queue.
3. **How does routing work?** — Producer sends a message with a routing key to an exchange. The exchange matches the
   routing key against queue bindings. Matching queues receive the message. Different exchange types use different
   matching rules.
4. **How to ensure no message loss?** — Five things: durable exchange, durable queue, persistent messages
   (`deliveryMode=2`), publisher confirms, manual consumer acknowledgment.
5. **What is a Dead Letter Exchange (DLX)?** — An exchange that receives messages that were rejected (nack without
   requeue), expired (TTL), or from a full queue. Used for error monitoring and delayed retry patterns.
6. **What is prefetch/QoS?** — Limits unacknowledged messages per consumer. `prefetch=1` ensures fair dispatch
   (slow consumer doesn't hoard messages). Higher values increase throughput but reduce fairness.
7. **`ack` vs `nack` vs `reject`?** — `ack`: processed successfully, remove. `nack`: failed, `requeue=true` puts
   it back (retry), `requeue=false` discards or routes to DLX. `reject`: same as nack but single message.
8. **RabbitMQ vs Kafka?** — RabbitMQ: smart broker (routing, priority, RPC), push-based, deletes after ack, lower
   throughput. Kafka: dumb broker, pull-based, retains messages, replay, much higher throughput.
9. **What are Quorum Queues?** — Raft-based replicated queues (RabbitMQ 3.8+). Data replicated across cluster
   nodes with leader election. Recommended over classic mirrored queues for high availability.

## Related

- [Apache Kafka](./apache-kafka.md) — event streaming alternative, detailed comparison
- [Microservices Architecture](../spring/cloud/microservices-architecture.md) — async communication patterns
- [Spring Cloud Ecosystem](../spring/cloud/spring-cloud-ecosystem.md) — Spring Cloud Stream abstraction

## Resources

- [RabbitMQ Official Documentation](https://www.rabbitmq.com/docs)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials) — step-by-step patterns
- [Spring AMQP Docs](https://docs.spring.io/spring-amqp/reference/)
- [CloudAMQP — RabbitMQ Best Practices](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html)
