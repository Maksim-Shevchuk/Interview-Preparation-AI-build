# Clean Architecture

Architectural style formalized by **Robert C. Martin ("Uncle Bob")** in the 2012 blog post *"The Clean Architecture"*
and the 2017 book of the same name. It organizes an application into **concentric layers** around a pure domain core
and enforces a single, non-negotiable rule: **dependencies must point inward**. The goal — independently echoed by
Hexagonal, Onion, and Screaming Architecture — is to keep business logic insulated from UI, frameworks, databases,
and external agencies.

One of the most frequently asked senior-backend interview topics, especially in Java/.NET/TypeScript shops, often
discussed alongside Hexagonal Architecture and DDD.

---

## Quick Reference

| Layer (outer ➜ inner)              | AKA                         | Contains                                              | Depends on              |
|------------------------------------|-----------------------------|-------------------------------------------------------|-------------------------|
| **Frameworks & Drivers**           | Infrastructure / Web / DB   | Spring, Express, React, JPA, Kafka clients, SQL      | Everything outside core |
| **Interface Adapters**             | Adapters / Gateways         | Controllers, presenters, DTOs, mappers, repositories  | Use Cases layer         |
| **Use Cases**                      | Application layer           | Application services, orchestrators, port interfaces  | Entities layer          |
| **Entities**                       | Enterprise / Domain layer   | Aggregates, value objects, domain services, invariants| Nothing (pure)          |

> The single rule that makes it "Clean": **source-code dependencies must point only inward, toward the inner layers.**
> Inner circles cannot know anything about outer circles — no names, no types, no function calls.

---

## 1. The Problem It Solves

A typical layered app has the dependency graph: `UI ➜ Service ➜ Repository ➜ DB`. The **business rules depend on the
database**, on Spring, on HTTP. Over time:

- Frameworks leak into the domain (`@Entity` on a domain class, `@Autowired` in a service).
- The system cannot be tested without spinning up the framework, the DB, the broker.
- Replacing a UI, swapping a database, or upgrading a framework becomes risky and expensive.
- Business intent gets buried under plumbing, DTOs, and annotations.

Clean Architecture addresses this by **inverting the dependency direction at every layer boundary** so that high-level
policy (the business) becomes the most stable part of the system, and low-level detail (the database, the framework)
becomes a replaceable plugin.

---

## 2. The Dependency Rule

```
                           Frameworks & Drivers          ← outermost, volatile
                          ┌─────────────────────────┐
                          │   Interface Adapters    │
                          │  ┌───────────────────┐  │
                          │  │    Use Cases      │  │
                          │  │  ┌─────────────┐  │  │
                          │  │  │  Entities   │  │  │  ← innermost, stable
                          │  │  └─────────────┘  │  │
                          │  └───────────────────┘  │
                          └─────────────────────────┘

   Source-code dependencies point ONLY inward ────────▶
```

Equivalent statements of the rule:

- Inner layers declare **what** they need; outer layers provide **how**.
- "Outer = mechanism, Inner = policy."
- Implemented through **interfaces defined in the inner layer**, implemented in the outer layer (DIP at scale).
- Inner layers must not import anything from outer layers — not annotations, not classes, not even exceptions.

---

## 3. The Four Layers

### 3.1 Entities (Enterprise Business Rules)

The most inner layer. Holds the **business rules that would exist even if there were no application** — they would
apply in any software (or even a manual procedure) used by the enterprise.

Contents:

- **Aggregates** (e.g., `Order`, `Customer`, `Loan`) — domain objects with identity and **invariants** enforced inside
  the class.
- **Value Objects** (e.g., `Money`, `Address`, `Email`) — immutable, equality-by-value descriptors.
- **Entity-level domain services** for operations that span multiple aggregates of the same enterprise concept.

Properties:

- Pure code, no framework imports, no I/O.
- Should be unit-testable with no mocks.
- Stable: changes here ripple everywhere, so the design here must be deliberate.

```java
public final class Loan {
    private final LoanId id;
    private Principal principal;
    private InterestRate rate;
    private LocalDate disbursedOn;

    public Money monthlyInstallment() {
        if (isRepaid()) return Money.ZERO;
        // pure arithmetic — no DB, no Spring
        return AmortizationFormulas.annuity(principal, rate, remainingMonths());
    }

    public void applyEarlyRepayment(Money amount) {
        if (amount.greaterThan(outstandingBalance()))
            throw new OverpaymentException(id);
        this.principal = principal.reduceBy(amount);
    }
}
```

### 3.2 Use Cases (Application Business Rules)

The layer that **defines what the application does**. Each use case = one user-intent flow (e.g.,
"Place Order", "Refund Payment", "Approve Loan"). Also called **interactors**.

Contents:

- Application services / use-case classes (`PlaceOrderUseCase`, `RefundPaymentInteractor`).
- **Port interfaces** the use case depends on (e.g., `OrderRepository`, `PaymentGateway`, `EventPublisher`) — declared
  here, implemented in outer layers.
- Application-level DTOs / commands / queries (input/output models).

Properties:

- Orchestrates entities and ports to fulfill a single intent.
- Implements business **workflow** rules (ordering of steps, transactions, state transitions), not enterprise rules.
- Should not know about HTTP, SQL, JSON.

```java
public class PlaceOrderUseCase {
    private final OrderRepository orders;        // port — interface lives here
    private final PaymentGateway payments;       // port
    private final EventPublisher  events;        // port

    public PlaceOrderResponse execute(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.customerId(), cmd.items());
        payments.charge(order.total(), cmd.paymentToken());
        orders.save(order);
        events.publish(new OrderPlaced(order.id()));
        return new PlaceOrderResponse(order.id());
    }
}
```

> **Subtle distinction:** Entities enforce rules of *the business*; Use Cases enforce rules of *the application*. "A
> loan's monthly installment follows the annuity formula" = entity rule. "After placing an order, send a confirmation
> email and publish an event" = use-case rule.

### 3.3 Interface Adapters

Translates data between formats convenient for the Use Cases/Entities and formats convenient for the outermost layer.

Contents:

- **Controllers** (REST, gRPC, MVC) — parse the inbound request, call a use case, format the response.
- **Presenters** — format use-case output for the UI / response.
- **Gateways / Repository implementations** — adapters that implement the port interfaces from the Use Cases layer
  using real technology (JPA, JDBC, HTTP clients).
- **DTOs, View Models, ORM entities, mappers** — anything that is a shape conversion.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final PlaceOrderUseCase placeOrder;

    @PostMapping
    public ResponseEntity<OrderResponse> place(@RequestBody OrderRequest req) {
        var cmd = new PlaceOrderCommand(req.customerId(), req.items());
        var out = placeOrder.execute(cmd);
        return ResponseEntity.ok(OrderResponse.from(out));
    }
}

// Adapter implementing a port declared in Use Cases
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa;
    public Optional<Order> findById(OrderId id)  { return jpa.findById(id.value()).map(OrderEntity::toDomain); }
    public void save(Order order)                 { jpa.save(OrderEntity.fromDomain(order)); }
}
```

### 3.4 Frameworks & Drivers

The outermost circle — the **most volatile** part. All technology lives here.

- Web frameworks (Spring MVC, Express, ASP.NET, Spark).
- Database tooling (Hibernate, JDBC templates, drivers, migration tools).
- Messaging clients (Kafka producers/consumers, AMQP, SQS).
- External SDKs (Stripe, Twilio, AWS SDK).
- UI frameworks (React, Angular, templates).

This layer is essentially a thin set of **glue adapters** that wire frameworks to the Interface Adapters layer. It
contains as little logic as possible.

---

## 4. How Data Crosses Boundaries

Each layer boundary is crossed using **plain data structures** (DTOs) — never by passing entities that carry
behaviour, and never by passing framework objects (`HttpServletRequest`, `HttpResponse`).

Rules:

1. Use cases accept and return their **own** input/output models (defined in the Use Cases layer).
2. Controllers convert HTTP DTOs ➜ use-case input models.
3. Presenters convert use-case output models ➜ response DTOs.
4. Repository adapters convert ORM entities ➜ domain entities (and vice versa).
5. **No entity crosses outward.** If a controller needs an entity's data, it goes through a DTO.

```
   HTTP JSON ──▶ Request DTO ──▶ Use Case Input Model ──▶ Entities
                                                       ◀──
   HTTP JSON ◀── Response DTO ◀── Use Case Output Model ◀──
```

This is verbose. It is also the price of strict isolation — and the reason Clean Architecture is most valuable for
complex domains, not CRUD.

---

## 5. Worked Example (Java)

### 5.1 Entities

```java
public final class Money {
    public static final Money ZERO = new Money(BigDecimal.ZERO);
    private final BigDecimal amount;
    public Money add(Money o)  { return new Money(amount.add(o.amount)); }
    public boolean isPositive() { return amount.signum() > 0; }
}

public final class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private Money total;

    public static Order create(CustomerId customer, List<OrderLine> lines) {
        if (lines.isEmpty()) throw new EmptyOrderException();
        return new Order(OrderId.next(), customer, sum(lines));
    }

    public Money total() { return total; }
}
```

### 5.2 Use Cases layer

```java
// Port: declared in this layer, implemented in Interface Adapters
public interface OrderRepository {
    void save(Order order);
}

// Port
public interface PaymentGateway {
    void charge(Money amount, PaymentToken token);
}

// Input / output models
public record PlaceOrderCommand(CustomerId customerId, List<OrderLine> items, PaymentToken token) {}
public record PlaceOrderResponse(OrderId orderId) {}

// Interactor
public class PlaceOrderUseCase {
    private final OrderRepository orders;
    private final PaymentGateway payments;
    public PlaceOrderUseCase(OrderRepository o, PaymentGateway p) { this.orders = o; this.payments = p; }

    public PlaceOrderResponse execute(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.customerId(), cmd.items());
        payments.charge(order.total(), cmd.token());
        orders.save(order);
        return new PlaceOrderResponse(order.id());
    }
}
```

### 5.3 Interface Adapters

```java
// Adapter implementing the port from Use Cases
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa;
    public JpaOrderRepository(OrderJpaRepository jpa) { this.jpa = jpa; }
    @Override public void save(Order order) { jpa.save(OrderEntity.fromDomain(order)); }
}

// REST adapter
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final PlaceOrderUseCase placeOrder;
    public OrderController(PlaceOrderUseCase u) { this.placeOrder = u; }

    @PostMapping
    public ResponseEntity<OrderDto> place(@RequestBody OrderRequestDto req) {
        var cmd = new PlaceOrderCommand(CustomerId.of(req.customer()), toLines(req), PaymentToken.of(req.token()));
        var out = placeOrder.execute(cmd);
        return ResponseEntity.ok(OrderDto.from(out));
    }
}
```

### 5.4 Frameworks & Drivers

```java
// Spring Boot wiring
@SpringBootApplication
public class Application {
    public static void main(String[] args) { SpringApplication.run(Application.class, args); }
}

@Configuration
class ApplicationConfig {
    @Bean PlaceOrderUseCase placeOrder(OrderRepository o, PaymentGateway p) { return new PlaceOrderUseCase(o, p); }
    @Bean OrderRepository  orderRepo(OrderJpaRepository jpa)                { return new JpaOrderRepository(jpa); }
    @Bean PaymentGateway   paymentGateway(StripeClient client)             { return new StripePaymentGateway(client); }
}
```

### 5.5 Test

```java
class PlaceOrderUseCaseTest {
    private final InMemoryOrderRepository orders = new InMemoryOrderRepository();
    private final List<Money> charged = new ArrayList<>();
    private final PaymentGateway payments = (amt, tok) -> charged.add(amt);

    @Test
    void places_order_and_charges_total() {
        var useCase = new PlaceOrderUseCase(orders, payments);
        var cmd = new PlaceOrderCommand(CustomerId.of("c1"), List.of(new OrderLine("SKU", Money.ZERO)), PaymentToken.of("tok"));
        var out = useCase.execute(cmd);
        assertThat(out.orderId()).isNotNull();
        assertThat(charged).isNotEmpty();
    }
}
```

> No Spring context, no DB, no Stripe. The use case is exercised as pure logic against in-memory fake adapters.

---

## 6. Compared with Related Patterns

| Aspect               | Layered                  | Hexagonal                  | Clean Architecture          | Onion                       |
|----------------------|--------------------------|----------------------------|-----------------------------|-----------------------------|
| Central shape        | Horizontal layers        | Hexagon with symmetric ports | Concentric circles          | Concentric layers           |
| Dependency rule      | Top ➜ bottom             | Adapter ➜ port ➜ core       | Outer ring ➜ inner ring     | Outer service ➜ inner domain|
| Persistence          | Bottom layer             | Just a driven adapter      | Outermost ring (a detail)   | Outermost service           |
| Distinct feature     | Simple separation        | Symmetry of ports          | Strict, named, layered rings| Domain model at center      |
| Layering of core     | Often mixed              | Free inside the hexagon    | Use Cases vs Entities       | Domain Services, Services   |
| Author / Year        | Traditional              | Cockburn, 2005             | R. C. Martin, 2012          | Palermo, 2008               |

> In practice Hexagonal and Clean Architecture overlap ~80%. Hexagonal focuses on **symmetry** (every external system
> is just an adapter). Clean Architecture focuses on **layering** (Entities vs Use Cases vs Adapters vs Frameworks).
> A real project often combines both vocabularies.

---

## 7. Pros and Cons

### Advantages

- **Testability.** Use cases and entities are testable without Spring, DB, or any framework.
- **Framework independence.** Frameworks are details — they can be swapped or upgraded without touching business rules.
- **Database independence.** SQL/NoSQL/Event-store swaps are isolated to a single adapter ring.
- **UI independence.** Replace REST with gRPC, or server-rendered HTML with SPA, without rewriting the core.
- **Long-term maintainability.** Business policy — the most stable asset — is the most independent.
- **Screaming architecture.** Folder structure reveals what the application *does*, not what framework it uses.

### Disadvantages

- **Verbosity.** Each use case needs an input model, an output model, an interface, an implementation, a mapper, and
  possibly DTOs. CRUD ops look heavy.
- **Steep learning curve.** Team must understand DIP, ports, layering, mapping boundaries.
- **Mapping overhead.** Domain ↔ ORM entity ↔ DTO conversions are tedious and error-prone.
- **Overkill for simple apps.** For thin CRUD/admin apps the pattern adds friction without payoff.
- **Indirection.** Tracing a request walks through 3–4 layers; tooling (debuggers, IDE navigation) must follow
  interfaces, not concrete calls.

---

## 8. When to Use / When Not

**Use Clean Architecture when:**

- The domain is complex — lots of rules, invariants, workflows, state machines.
- The application is **long-lived** (years of evolution), where maintenance dominates initial cost.
- You need multiple **fronts**: REST + gRPC + messaging consumers + CLI + scheduled jobs.
- You may need to swap infrastructure (DB, vendor, framework) without rewriting the core.
- The team values DDD and is comfortable with abstraction.

**Avoid it when:**

- The app is mostly CRUD / thin wrapper over a database.
- It's a prototype, MVP, or throwaway script.
- The team is small or unfamiliar with DI and architectural patterns.
- Time-to-market dominates maintainability concerns (e.g., hackathons, internal tools).

---

## 9. Common Interview Questions

1. **What is Clean Architecture?**
   An architectural style by Robert C. Martin organizing an application into concentric layers — Entities, Use Cases,
   Interface Adapters, Frameworks & Drivers — with the strict rule that dependencies point only inward, so that
   business rules do not depend on UI, frameworks, or databases.

2. **What is the Dependency Rule?**
   Source-code dependencies must point only inward, toward the inner layers. The inner circles cannot know anything
   about outer circles — no names, no types, no function calls.

3. **List the four layers and what each contains.**
   - **Entities**: enterprise business rules — aggregates, value objects, invariants.
   - **Use Cases**: application business rules — orchestration of entities via ports.
   - **Interface Adapters**: controllers, presenters, gateways, DTOs, mappers.
   - **Frameworks & Drivers**: Spring, JPA, Kafka, React, external SDKs.

4. **Difference between Entities and Use Cases?**
   Entities hold rules that would exist even **without the application** (enterprise rules). Use Cases hold rules that
   define **what the application does** — the workflow that orchestrates entities and ports.

5. **How does Clean Architecture relate to DIP (Dependency Inversion)?**
   Each layer boundary is a place where DIP is applied: the inner layer declares an interface (port), the outer layer
   implements it. This is the mechanism by which the dependency direction is inverted.

6. **What is a port? Where does it live?**
   A port is an interface declared by the inner layer that defines a capability it needs. It lives in the **Use Cases**
   layer (for driven ports like repositories) or is exposed by Use Cases themselves (for driving ports). Outer layers
   implement or consume it.

7. **How does data cross the layer boundaries?**
   Using plain data structures (DTOs / input & output models) defined in the inner layer. Entities and framework
   objects must not leak across boundaries.

8. **Difference from Hexagonal Architecture?**
   Hexagonal emphasizes the **symmetry** of ports — every external system is just an adapter, with no privileged
   direction. Clean Architecture emphasizes **strict concentric layering** (Entities vs Use Cases vs Adapters vs
   Frameworks). They overlap heavily and are often combined.

9. **Where does `@Transactional` belong?**
   In the outer layers — on the use-case boundary or on an adapter, **not inside Entities**. Transactions are
   infrastructure details; the inner layers must remain free of such annotations.

10. **Trade-offs?**
    More boilerplate (DTOs, mappers, interfaces), learning curve, indirection, mapping overhead. Worth it for complex,
    long-lived, multi-frontend applications; overkill for CRUD/admin tools.

11. **Why is the database "a detail"?**
    Because the choice of SQL/NoSQL/event-store affects only the outermost layer. The application's business rules
    should not be shaped by which storage technology was picked. Storage is replaceable; the business is not.

12. **What does "Screaming Architecture" mean?**
    Looking at the top-level package structure should tell you **what the application does** ("orders, loans,
    shipments"), not which framework it uses ("controllers, services, models, Spring"). The architecture should
    "scream" the intent of the system.

---

## 10. Mental Cheat-Sheet

> **Four rings, one rule.**
> Entities ➜ Use Cases ➜ Interface Adapters ➜ Frameworks & Drivers.
> **Dependencies point only inward.** Inner = stable policy. Outer = volatile detail.
> Cross every boundary with a DTO. Define every dependency as a port implemented in an outer ring.

```
                            ┌─────────────────────────────────────┐
                            │   Frameworks & Drivers              │  Spring, JPA, Kafka, React
                            │  ┌─────────────────────────────┐    │
                            │  │  Interface Adapters         │    │  Controllers, presenters,
                            │  │  ┌──────────────────────┐   │    │  gateway impls, DTOs, mappers
                            │  │  │  Use Cases           │   │    │
                            │  │  │  ┌────────────────┐  │   │    │  Application services,
                            │  │  │  │   Entities     │  │   │    │  use-case ports, I/O models
                            │  │  │  └────────────────┘  │   │    │
                            │  │  └──────────────────────┘   │    │  Aggregates, value objects,
                            │  └─────────────────────────────┘    │  invariants (pure)
                            └─────────────────────────────────────┘
                                  ◀──── dependencies point IN
```

### Suggested Project Layout

```
src/main/java/com/acme/
├── domain/                    # Entities (enterprise rules, pure)
│   ├── model/                 #   Order, Money, OrderLine
│   └── service/               #   domain services spanning aggregates
├── application/               # Use Cases (application rules)
│   ├── port/
│   │   ├── in/                #   driving ports (use-case interfaces)
│   │   └── out/               #   driven ports (repository, gateway)
│   ├── usecase/               #   use-case implementations
│   └── model/                 #   command / query / response models
├── adapter/                   # Interface Adapters
│   ├── in/                    #   REST controllers, CLI, gRPC, consumers
│   ├── out/                   #   JPA repositories, SMTP, Stripe
│   └── mapper/                #   DTO ↔ domain conversions
└── infrastructure/            # Frameworks & Drivers (config, wiring)
    ├── persistence/           #   JPA entities, Spring Data interfaces
    ├── messaging/             #   Kafka producer/consumer configs
    └── config/                #   @Configuration, beans
```
