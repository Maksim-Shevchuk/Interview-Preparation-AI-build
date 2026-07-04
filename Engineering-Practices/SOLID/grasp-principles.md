# GRASP Principles

**G**eneral **R**esponsibility **A**ssignment **S**oftware **P**atterns — nine principles by Craig Larman that guide
**which class should be responsible for what**. GRASP complements SOLID: while SOLID focuses on code quality and
flexibility, GRASP focuses on the initial decision of **where to put the logic**.

## Quick Reference

| #  | Principle              | Question it answers                                      |
|----|------------------------|----------------------------------------------------------|
| 1  | Information Expert     | Which class should handle this responsibility?           |
| 2  | Creator                | Which class should create instances of another class?    |
| 3  | Controller             | Who handles a system event / use case?                   |
| 4  | Low Coupling           | How to reduce dependencies between classes?              |
| 5  | High Cohesion          | How to keep classes focused and manageable?              |
| 6  | Polymorphism           | How to handle type-based alternatives?                   |
| 7  | Pure Fabrication       | What if no domain class fits the responsibility?         |
| 8  | Indirection            | How to decouple two classes that need to interact?       |
| 9  | Protected Variations   | How to protect against variation / change?               |

---

## 1. Information Expert

> Assign responsibility to the class that has the **information needed** to fulfill it.

The class with the most relevant data should own the behavior.

```java
// ❌ Logic in a service — Order has the data but doesn't use it
public class OrderService {
    public double calculateTotal(Order order) {
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        return total;
    }
}

// ✅ Order has the items — it's the expert
public class Order {
    private List<OrderItem> items;

    public double calculateTotal() {
        return items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
    }
}
```

**Guideline:** Before putting logic in a service, ask: "Does a domain object already have the data for this?" If yes,
the logic belongs there — this leads to a **rich domain model** (DDD) instead of an anemic one.

**Exception:** When the computation crosses multiple objects or involves infrastructure (DB, network), a service is
appropriate.

---

## 2. Creator

> Class `A` should create instances of class `B` if `A` **contains, aggregates, records, or closely uses** `B`.

```java
// ✅ Order creates OrderItems — it contains and aggregates them
public class Order {
    private final List<OrderItem> items = new ArrayList<>();

    public void addItem(Product product, int quantity) {
        items.add(new OrderItem(product, quantity, product.getPrice()));
    }
}

// ❌ An unrelated service creates OrderItems and passes them around
```

**In Spring / DI context:** Factories and DI containers take over creation for service-layer objects. Creator mainly
applies to **domain objects** (entities creating value objects, aggregates creating child entities).

---

## 3. Controller

> Assign the responsibility of handling a **system event** (use case) to a non-UI class that represents the overall
> system or a use-case scenario.

The controller is the **first object behind the UI** that coordinates a use case.

```
UI (React / REST endpoint) ──▶ Controller ──▶ Service(s) ──▶ Domain
```

```java
// In Spring, @Controller / @RestController IS the GRASP controller
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderDto> placeOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.placeOrder(request); // delegates to service
        return ResponseEntity.status(201).body(OrderDto.from(order));
    }
}
```

**Don't put business logic in the controller** — it only translates HTTP → service call → HTTP response. This is also
SRP: the controller's single responsibility is HTTP handling.

### Variants

- **Facade Controller** — one controller per subsystem (e.g., `OrderController` handles all order endpoints).
- **Use-Case Controller** — one controller per use case (e.g., `PlaceOrderController`, `CancelOrderController`).
  More granular, useful for complex use cases.

---

## 4. Low Coupling

> Assign responsibilities to **minimize dependencies** between classes.

Coupling = how much one class knows about or relies on another. Lower coupling → easier to change, test, and reuse.

```java
// ❌ High coupling — OrderService depends on concrete EmailClient
public class OrderService {
    private final GmailEmailClient emailClient = new GmailEmailClient();
}

// ✅ Low coupling — depend on abstraction
public class OrderService {
    private final NotificationService notifications; // interface

    public OrderService(NotificationService notifications) {
        this.notifications = notifications;
    }
}
```

**Types of coupling (from worst to best):**

| Type              | Description                                    | Example                        |
|-------------------|------------------------------------------------|--------------------------------|
| Content           | One class modifies internals of another        | Accessing private fields       |
| Common            | Classes share global state                     | Global variables, singletons   |
| Control           | One class tells another HOW to behave (flags)  | `process(data, useCache=true)` |
| Stamp/Data        | Classes share data structures                  | Passing DTOs                   |
| Message           | Classes communicate via method calls only      | Interface method calls         |

**Low coupling does NOT mean zero coupling.** Some coupling is necessary — the goal is to couple to **stable
abstractions** rather than volatile implementations.

---

## 5. High Cohesion

> Assign responsibilities so that a class remains **focused** — its methods and data are closely related.

Cohesion = how strongly the responsibilities within a class belong together. High cohesion → small, focused classes.

```java
// ❌ Low cohesion — UserService does everything user-related
public class UserService {
    public void register(User user) { /* ... */ }
    public void sendWelcomeEmail(User user) { /* ... */ }
    public void generateReport(User user) { /* ... */ }
    public void exportToCsv(List<User> users) { /* ... */ }
    public void syncWithLDAP() { /* ... */ }
}

// ✅ High cohesion — each class has a focused purpose
public class UserRegistrationService { /* register, validate */ }
public class UserNotificationService { /* emails, push */ }
public class UserReportService { /* reports, exports */ }
public class LDAPSyncService { /* LDAP integration */ }
```

**High cohesion and SRP are two sides of the same coin.** SRP says "one reason to change." High cohesion says "keep
related things together."

### Measuring Cohesion (Informal)

Ask: "If I describe this class in one sentence without using 'and', can I?" If not, it probably has low cohesion.

- "UserService **registers** users **and** sends emails **and** generates reports" → low cohesion.
- "UserRegistrationService validates and persists new user accounts" → high cohesion.

---

## 6. Polymorphism

> When behavior varies by type, assign the responsibility to the types using **polymorphism** instead of conditionals.

This is the OCP (Open/Closed Principle) in action.

```java
// ❌ Type-based conditional
public class NotificationSender {
    public void send(Notification n) {
        if (n.getType() == Type.EMAIL) {
            sendEmail(n);
        } else if (n.getType() == Type.SMS) {
            sendSms(n);
        } else if (n.getType() == Type.PUSH) {
            sendPush(n);
        }
    }
}

// ✅ Polymorphism — each type handles itself
public interface NotificationChannel {
    void send(Notification notification);
}

public class EmailChannel implements NotificationChannel {
    @Override public void send(Notification n) { /* email logic */ }
}

public class SmsChannel implements NotificationChannel {
    @Override public void send(Notification n) { /* SMS logic */ }
}

public class PushChannel implements NotificationChannel {
    @Override public void send(Notification n) { /* push logic */ }
}

// Client code — no conditionals, extensible
public class NotificationSender {
    private final NotificationChannel channel;

    public void send(Notification n) {
        channel.send(n); // polymorphic dispatch
    }
}
```

---

## 7. Pure Fabrication

> When no existing domain class is a good fit for a responsibility, **invent a class** that doesn't represent a domain
> concept.

These are **service classes, repositories, adapters, gateways** — they exist for technical reasons, not because they
model a real-world entity.

```java
// "OrderRepository" doesn't exist in the real-world domain — it's a pure fabrication
// But it's the right place for persistence logic
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
}

// "EventPublisher" is also a fabrication — no domain concept, but useful for decoupling
public interface EventPublisher {
    void publish(DomainEvent event);
}
```

**Why needed:** Putting persistence in `Order` (Information Expert applied naively) would give it too many
responsibilities (low cohesion) and couple it to infrastructure (high coupling). Pure Fabrication resolves the tension.

### Common Pure Fabrications in Spring

| Class                | Purpose                               |
|----------------------|---------------------------------------|
| `*Repository`        | Data access                           |
| `*Service`           | Orchestration / use-case logic        |
| `*Mapper` / `*Converter` | DTO ↔ Entity transformation      |
| `*Client`            | External API communication            |
| `*Factory`           | Complex object creation               |
| `*Validator`         | Validation rules                      |

---

## 8. Indirection

> Introduce an **intermediary object** to decouple two classes that would otherwise be directly coupled.

```
❌  ServiceA ──────────▶ ServiceB     (direct coupling)
✅  ServiceA ──▶ Interface ◀── ServiceB  (indirection via abstraction)
✅  ServiceA ──▶ MessageBroker ──▶ ServiceB  (indirection via middleware)
```

**Examples of indirection:**
- **Interfaces** — decouple caller from implementation.
- **Message brokers** (Kafka, RabbitMQ) — decouple producer from consumer.
- **DTO / Adapter** — decouple layers (controller doesn't expose entity directly).
- **Event bus** — decouple event emitter from handlers.
- **Repository pattern** — decouples domain from persistence technology.

**Indirection is the mechanism; Low Coupling is the goal.**

> "Most problems in computer science can be solved by another level of indirection." — David Wheeler

---

## 9. Protected Variations

> Identify **points of predicted variation** and create a stable interface around them to shield the rest of the system.

Wrap the unstable part behind an abstraction so that changes don't ripple through the codebase.

```java
// Payment provider may change (Stripe → PayPal → custom) — point of variation
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
    PaymentResult refund(String transactionId, Money amount);
}

// Current implementation — can be swapped without touching any client code
@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Money amount, PaymentMethod method) {
        // Stripe SDK calls
    }
}
```

**Protected Variations is the generalization of OCP, DIP, and Indirection.** It's the overarching principle:
identify what changes, hide it behind a stable abstraction.

### Common Points of Variation

| What might change         | Protection strategy                         |
|---------------------------|---------------------------------------------|
| Database vendor           | Repository interface                        |
| Payment provider          | PaymentGateway interface                    |
| Notification channel      | NotificationService interface + strategy    |
| External API format       | Adapter / Anti-corruption layer             |
| Business rules            | Strategy pattern, rule engine               |
| UI framework              | Separate business logic from UI (hooks, services) |

---

## GRASP + SOLID — How They Relate

| GRASP                  | Related SOLID          | Connection                                    |
|------------------------|------------------------|-----------------------------------------------|
| Information Expert     | SRP                    | Expert assigns the right responsibility; SRP limits scope |
| Low Coupling           | DIP                    | Both push toward depending on abstractions    |
| High Cohesion          | SRP                    | Cohesion = focused responsibility             |
| Polymorphism           | OCP                    | Add behavior via new types, not conditionals  |
| Protected Variations   | OCP + DIP              | Shield from change via abstractions           |
| Indirection            | DIP                    | Intermediary objects to reduce coupling       |
| Pure Fabrication       | ISP (indirectly)       | Create technical classes to keep domain clean |

**SOLID** tells you what properties good code should have.
**GRASP** tells you **how to assign responsibilities** to achieve those properties.

---

## Common Interview Questions

1. **What is GRASP?** — Nine responsibility assignment principles by Craig Larman. They guide which class should own
   which behavior: Information Expert, Creator, Controller, Low Coupling, High Cohesion, Polymorphism, Pure
   Fabrication, Indirection, Protected Variations.
2. **What is Information Expert?** — Assign responsibility to the class that has the data needed to fulfill it. The
   class with the information is the "expert."
3. **Low Coupling vs High Cohesion?** — Low Coupling: minimize dependencies between classes. High Cohesion: keep
   each class focused on one purpose. They work together — splitting a low-cohesion class increases cohesion and
   typically maintains or improves coupling.
4. **What is Pure Fabrication? Give an example.** — A class that doesn't represent a domain concept but exists for
   technical reasons (Repository, Mapper, EventPublisher). Needed when putting logic in a domain class would hurt
   cohesion or coupling.
5. **How do GRASP and SOLID relate?** — SOLID defines properties of good design (single responsibility, open for
   extension, etc.). GRASP provides guidelines for achieving those properties by deciding where responsibilities go.
   They are complementary, not competing.
6. **What is Protected Variations?** — Identify what's likely to change, wrap it behind a stable interface. It's the
   general principle behind OCP, DIP, Strategy pattern, and adapter layers.

## Related

- [SOLID Principles](./solid-principles.md) — the five design principles GRASP complements
- [Design Patterns](../Design-Patterns/) — Strategy, Observer, Adapter implement GRASP principles
- [Microservices Architecture](../../Java/spring/cloud/microservices-architecture.md) — SRP/coupling at service level

## Resources

- Craig Larman — *Applying UML and Patterns* (canonical GRASP reference)
- [Baeldung — GRASP Design Principles](https://www.baeldung.com/java-grasp-design-principles)
