# Microservices Architecture

Architectural style where an application is structured as a collection of **loosely coupled, independently deployable
services**, each owning a specific business capability. One of the most popular system design interview topics for
senior backend roles.

## Monolith vs Microservices

### Monolithic Architecture

```
┌─────────────────────────────────┐
│           Monolith              │
│  ┌───────┐ ┌───────┐ ┌───────┐ │
│  │ Users │ │Orders │ │Payment│ │     Single deployable unit
│  └───┬───┘ └───┬───┘ └───┬───┘ │     Single database
│      └─────────┴─────────┘     │
│           ┌──────┐             │
│           │  DB  │             │
│           └──────┘             │
└─────────────────────────────────┘
```

### Microservices Architecture

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Users   │   │  Orders  │   │ Payment  │    Independent services
│ Service  │   │ Service  │   │ Service  │    Own databases
│ ┌──────┐ │   │ ┌──────┐ │   │ ┌──────┐ │    Independent deployment
│ │ DB   │ │   │ │ DB   │ │   │ │ DB   │ │
│ └──────┘ │   │ └──────┘ │   │ └──────┘ │
└──────────┘   └──────────┘   └──────────┘
       ▲              ▲              ▲
       └──────────────┼──────────────┘
                ┌─────┴──────┐
                │ API Gateway│
                └────────────┘
```

### Comparison

| Aspect               | Monolith                              | Microservices                          |
|----------------------|---------------------------------------|----------------------------------------|
| Deployment           | All or nothing                        | Independent per service                |
| Scaling              | Scale the whole app                   | Scale individual services              |
| Tech stack           | Single stack                          | Polyglot (different tech per service)  |
| Data                 | Shared database                       | Database per service                   |
| Team structure       | Single team or functional teams       | Cross-functional teams per service     |
| Complexity           | Simple at small scale                 | Distributed system complexity          |
| Failure isolation    | One bug can crash everything          | Failure contained to one service       |
| Development speed    | Fast at start, slows with growth      | Slower start, scales with teams        |
| Testing              | Simple integration testing            | Complex E2E, contract testing          |
| Consistency          | Strong (ACID transactions)            | Eventual consistency                   |

### When to Choose Microservices

**Use microservices when:**
- Team is large (multiple teams need independent velocity).
- Different parts of the system have **different scaling requirements**.
- You need **independent deployment** (frequent releases of specific features).
- Domain is well-understood and can be cleanly **decomposed into bounded contexts**.

**Stay monolithic when:**
- Small team (< 10 developers).
- Startup / MVP — speed of iteration > architectural purity.
- Domain boundaries are unclear.
- You don't have the operational maturity (CI/CD, monitoring, container orchestration).

> "Don't start with microservices. Start with a well-structured monolith and extract services when the pain justifies
> the complexity." — Martin Fowler

## Domain-Driven Design (DDD) and Service Boundaries

Microservices map naturally to DDD's **Bounded Contexts** — each service owns one context.

### Key DDD Concepts

| Concept             | Description                                                       |
|---------------------|-------------------------------------------------------------------|
| **Bounded Context** | A boundary within which a domain model is defined and consistent  |
| **Aggregate**       | A cluster of entities treated as a single unit for data changes   |
| **Aggregate Root**  | The entry point entity that enforces invariants of the aggregate  |
| **Domain Event**    | Something that happened in the domain that other contexts care about |
| **Ubiquitous Language** | Shared vocabulary between developers and domain experts within a context |

### Example: E-Commerce

```
┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐
│   Catalog    │  │   Orders    │  │   Payment    │  │   Shipping    │
│   Context    │  │   Context   │  │   Context    │  │   Context     │
│              │  │             │  │              │  │               │
│ Product      │  │ Order       │  │ Transaction  │  │ Shipment      │
│ Category     │  │ OrderItem   │  │ Refund       │  │ TrackingInfo  │
│ Price        │  │ Cart        │  │ Invoice      │  │ Carrier       │
└─────────────┘  └─────────────┘  └──────────────┘  └───────────────┘
```

**Note:** "Product" may exist in multiple contexts but means different things — in Catalog it has description/images,
in Orders it's just an ID + price snapshot. This is expected and correct.

## Communication Patterns

### Synchronous Communication

```
Service A ──HTTP/gRPC──▶ Service B
           ◀─response───
```

#### REST (HTTP/JSON)

```java
// Spring: Calling another service via RestClient (Spring 6.1+)
@Service
public class OrderService {

    private final RestClient restClient;

    public OrderService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://user-service").build();
    }

    public UserDto getUser(Long userId) {
        return restClient.get()
                .uri("/api/users/{id}", userId)
                .retrieve()
                .body(UserDto.class);
    }
}
```

#### OpenFeign (Declarative REST Client)

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users/{id}")
    UserDto getUser(@PathVariable Long id);
}

// Usage — injected like any Spring bean
@Service
public class OrderService {
    private final UserClient userClient;
    // ...
}
```

#### gRPC

- Binary protocol (Protocol Buffers) — faster serialization, smaller payloads than JSON.
- Supports streaming (server, client, bidirectional).
- Strong typing via `.proto` schema files.
- Use for internal service-to-service communication where performance matters.

### Asynchronous Communication (Event-Driven)

```
Service A ──event──▶ Message Broker ──event──▶ Service B
                     (Kafka, RabbitMQ)    ──▶ Service C
```

```java
// Spring Kafka — producing an event
@Service
public class OrderService {

    private final KafkaTemplate<String, OrderEvent> kafka;

    public void placeOrder(Order order) {
        orderRepository.save(order);
        kafka.send("order-events", new OrderCreatedEvent(order.getId(), order.getUserId()));
    }
}

// Consuming an event in another service
@Component
public class PaymentEventListener {

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void onOrderCreated(OrderCreatedEvent event) {
        paymentService.initiatePayment(event.orderId());
    }
}
```

### Sync vs Async

| Aspect           | Synchronous (REST/gRPC)         | Asynchronous (Events/Messages)     |
|------------------|---------------------------------|------------------------------------|
| Coupling         | Temporal + spatial              | Decoupled                          |
| Latency          | Blocked until response          | Fire and forget                    |
| Failure handling | Cascading failures possible     | Retry from queue, no cascade       |
| Consistency      | Easier to reason about          | Eventual consistency               |
| Debugging        | Simple request tracing          | Harder — distributed events        |
| Use when         | Need immediate response         | Can tolerate delay, need resilience|

**Best practice:** Use sync for queries (read data from another service), async for commands/events (something
happened, react to it).

## API Gateway

Single entry point for all client requests. Routes to appropriate microservices.

```
Client ──▶ API Gateway ──▶ User Service
                       ──▶ Order Service
                       ──▶ Product Service
```

**Responsibilities:**
- **Routing** — direct requests to the correct service.
- **Authentication/Authorization** — validate JWT/OAuth tokens once at the edge.
- **Rate limiting** — protect services from overload.
- **Load balancing** — distribute requests across service instances.
- **Response aggregation** — combine responses from multiple services into one.
- **SSL termination** — handle HTTPS at the gateway, HTTP internally.
- **Caching, logging, monitoring.**

**Implementations:** Spring Cloud Gateway, Kong, NGINX, AWS API Gateway, Envoy (with Istio).

## Service Discovery

Services need to find each other. Two approaches:

### Client-Side Discovery

```
Service A ──query──▶ Service Registry (Eureka) ──▶ returns [host1:8081, host2:8082]
Service A ──request──▶ host1:8081
```

Service A gets the list of instances and load-balances itself (Spring Cloud LoadBalancer).

### Server-Side Discovery

```
Service A ──request──▶ Load Balancer ──▶ routes to healthy instance
```

Load balancer queries the registry (Kubernetes Services, AWS ELB).

**Implementations:** Netflix Eureka, Consul, Kubernetes DNS, AWS Cloud Map.

## Resilience Patterns

### Circuit Breaker

Prevents cascading failures by stopping requests to a failing service:

```
CLOSED ──failures exceed threshold──▶ OPEN ──timeout──▶ HALF-OPEN
  ▲                                                        │
  └───────────success in half-open────────────────────────┘
              failure in half-open──▶ OPEN
```

```java
// Resilience4j
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackGetUser")
public UserDto getUser(Long id) {
    return userClient.getUser(id);
}

public UserDto fallbackGetUser(Long id, Throwable t) {
    return UserDto.unknown(id); // graceful degradation
}
```

### Other Resilience Patterns

| Pattern          | Purpose                                                          |
|------------------|------------------------------------------------------------------|
| **Retry**        | Retry transient failures with backoff                            |
| **Timeout**      | Fail fast instead of waiting forever                             |
| **Bulkhead**     | Isolate resources per service (thread pools / semaphores) — one slow service doesn't exhaust all threads |
| **Rate Limiter** | Limit request rate to protect a service                          |
| **Fallback**     | Return a default/cached value when the primary call fails        |

```java
// Combining patterns with Resilience4j annotations
@Retry(name = "userService", fallbackMethod = "fallback")
@CircuitBreaker(name = "userService", fallbackMethod = "fallback")
@TimeLimiter(name = "userService")
@Bulkhead(name = "userService")
public CompletableFuture<UserDto> getUser(Long id) {
    return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
}
```

## Data Management

### Database per Service

Each microservice owns its data. **No direct database access** across service boundaries.

```
❌ Order Service ──SQL──▶ User Service DB     (breaks encapsulation)
✅ Order Service ──API──▶ User Service ──▶ DB  (proper boundary)
```

**Consequences:**
- No cross-service JOINs — need API calls or data duplication.
- No distributed ACID transactions — need eventual consistency patterns.

### Saga Pattern

Manages distributed transactions across services without 2PC (two-phase commit):

#### Choreography (Event-Driven)

Each service publishes an event → next service reacts:

```
Order Service ──OrderCreated──▶ Payment Service ──PaymentCompleted──▶ Inventory Service
                                PaymentFailed──▶ Order Service (compensate: cancel order)
```

**Pros:** Simple, decoupled.
**Cons:** Hard to track the overall flow, debugging is complex.

#### Orchestration

A central **Saga Orchestrator** coordinates the steps:

```
Orchestrator ──createPayment──▶ Payment Service
             ◀──paymentOK──────
             ──reserveStock──▶ Inventory Service
             ◀──stockReserved──
             ──confirmOrder──▶ Order Service
```

If any step fails, the orchestrator calls **compensating transactions** (rollback):

```
Payment failed → cancel order
Stock reservation failed → refund payment → cancel order
```

**Pros:** Centralized control, easy to understand the flow.
**Cons:** Orchestrator is a single point of complexity.

### CQRS (Command Query Responsibility Segregation)

Separate **write model** (commands) from **read model** (queries):

```
Write side:  Command ──▶ Domain Model ──▶ Event Store / Write DB
                                              │
                                         Event published
                                              │
Read side:                                    ▼
             Query ──▶ Read Model ◀── Projection (denormalized view)
```

**Use when:** Read and write patterns are very different (e.g., complex domain logic for writes, fast denormalized
reads for dashboards).

### Event Sourcing

Instead of storing current state, store **all events** that led to the current state:

```
Events for Order #123:
1. OrderCreated { items: [...], userId: 42 }
2. PaymentReceived { amount: 99.99 }
3. ItemShipped { trackingId: "ABC" }
4. OrderCompleted {}

Current state = replay(events)
```

**Pros:** Full audit trail, time-travel debugging, event replay.
**Cons:** Complexity, eventual consistency, storage growth.

## Observability

The "three pillars" — essential for debugging distributed systems:

### 1. Distributed Tracing

Track a request across multiple services via a **trace ID**:

```
Request ──▶ API Gateway (traceId: abc-123)
               ──▶ Order Service (traceId: abc-123, spanId: 1)
                      ──▶ User Service (traceId: abc-123, spanId: 2)
                      ──▶ Payment Service (traceId: abc-123, spanId: 3)
```

**Tools:** OpenTelemetry (standard), Jaeger, Zipkin.
**Spring:** Micrometer Tracing (replaces Spring Cloud Sleuth) auto-instruments trace/span propagation.

### 2. Centralized Logging

Aggregate logs from all services into one place. Always include `traceId` in log entries.

```
[order-service]   traceId=abc-123 | INFO  | Order created: #42
[payment-service] traceId=abc-123 | INFO  | Payment initiated for order #42
[payment-service] traceId=abc-123 | ERROR | Payment gateway timeout
```

**Stack:** ELK (Elasticsearch + Logstash + Kibana), Grafana Loki, Datadog.

### 3. Metrics and Monitoring

Measure throughput, latency, error rates, resource utilization per service.

**Key metrics (RED method):**
- **R**ate — requests per second.
- **E**rrors — error rate (5xx, exceptions).
- **D**uration — latency percentiles (p50, p95, p99).

**Tools:** Prometheus + Grafana, Spring Boot Actuator + Micrometer.

## Configuration Management

Externalize configuration, don't bake it into the service:

| Approach                  | Description                                    |
|---------------------------|------------------------------------------------|
| **Spring Cloud Config**   | Centralized config server backed by Git        |
| **Kubernetes ConfigMaps** | K8s-native configuration                       |
| **HashiCorp Vault**       | Secrets management (DB passwords, API keys)    |
| **Environment variables** | Simple, 12-factor app style                    |

## Deployment Patterns

| Pattern            | Description                                                        |
|--------------------|--------------------------------------------------------------------|
| **Blue-Green**     | Two environments — switch traffic from blue (old) to green (new)   |
| **Canary**         | Route a small % of traffic to new version, gradually increase      |
| **Rolling Update** | Replace instances one by one (Kubernetes default)                  |
| **Feature Flags**  | Deploy code but toggle features on/off without redeploying         |

## Anti-Patterns

| Anti-Pattern                  | Problem                                                    |
|-------------------------------|------------------------------------------------------------|
| **Distributed Monolith**      | Services are tightly coupled — must deploy together        |
| **Shared Database**           | Breaks data ownership, introduces hidden coupling          |
| **Nano-services**             | Services too small — overhead > benefit                    |
| **Sync chains**               | A→B→C→D sync calls — high latency, cascading failures     |
| **No API versioning**         | Breaking changes break all consumers                       |
| **Missing observability**     | Debugging distributed systems without tracing is impossible|

## Common Interview Questions

1. **Monolith vs microservices — trade-offs?** — Monolith: simpler, faster to start, ACID transactions. Micro:
   independent deployment, independent scaling, team autonomy. Micro adds distributed system complexity (networking,
   consistency, observability).
2. **How to decompose a monolith into microservices?** — Identify bounded contexts (DDD). Start with the strangler
   fig pattern — extract one service at a time from the monolith while maintaining the old system.
3. **How do microservices communicate?** — Sync: REST or gRPC (for queries, immediate responses). Async: message
   broker — Kafka, RabbitMQ (for events, commands, decoupling).
4. **What is a circuit breaker?** — A resilience pattern that stops calling a failing service after a threshold.
   States: Closed (normal) → Open (failing, return fallback) → Half-Open (test with limited requests).
5. **How to handle distributed transactions?** — Saga pattern (choreography or orchestration). Each service does its
   local transaction and publishes events. On failure, compensating transactions roll back previous steps.
6. **What is eventual consistency?** — In a distributed system, data across services will become consistent
   eventually (not immediately). Trade-off of the CAP theorem — microservices favor availability and partition
   tolerance over strong consistency.
7. **How to debug a request across microservices?** — Distributed tracing (OpenTelemetry): a trace ID is propagated
   across all service calls. Combined with centralized logging and metrics (the three pillars of observability).

## Related

- [Spring Cloud Ecosystem](./spring-cloud-ecosystem.md) — Spring-specific implementations
- [@Transactional Deep Dive](../data/transactional-annotation.md) — local transactions within a service
- [N+1 Problem](../data/n-plus-one-problem.md) — data access patterns within a service

## Resources

- Martin Fowler — [Microservices](https://martinfowler.com/articles/microservices.html)
- Sam Newman — *Building Microservices* (O'Reilly)
- Chris Richardson — [microservices.io](https://microservices.io/) (patterns catalog)
- [12-Factor App](https://12factor.net/)
