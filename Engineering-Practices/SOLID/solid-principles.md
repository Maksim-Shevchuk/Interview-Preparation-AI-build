# SOLID Principles

Five design principles introduced by Robert C. Martin that make object-oriented code **more maintainable, flexible, and
testable**. One of the most frequently asked interview topics — interviewers expect you to name all five, explain each
with an example, and recognize violations in code.

## Quick Reference

| Letter | Principle                        | One-liner                                                |
|--------|----------------------------------|----------------------------------------------------------|
| **S**  | Single Responsibility            | A class should have only one reason to change            |
| **O**  | Open/Closed                      | Open for extension, closed for modification              |
| **L**  | Liskov Substitution              | Subtypes must be substitutable for their base types      |
| **I**  | Interface Segregation            | Prefer many small interfaces over one fat interface      |
| **D**  | Dependency Inversion             | Depend on abstractions, not on concrete implementations  |

---

## S — Single Responsibility Principle (SRP)

> A class should have **one and only one reason to change** — meaning it should have only one job.

"Reason to change" = one actor/stakeholder whose requirements could cause the class to be modified.

### Violation

```java
public class UserService {
    public void registerUser(User user) {
        // validate input
        if (user.getEmail() == null) throw new ValidationException("Email required");

        // save to DB
        jdbcTemplate.update("INSERT INTO users ...", user.getName(), user.getEmail());

        // send welcome email
        emailClient.send(user.getEmail(), "Welcome!", "...");

        // write audit log
        auditLogger.log("USER_REGISTERED", user.getId());
    }
}
```

This class has **four reasons to change**: validation rules, persistence logic, email format, logging format.

### Fix

```java
public class UserService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final NotificationService notifications;
    private final AuditService audit;

    public void registerUser(User user) {
        validator.validate(user);
        repository.save(user);
        notifications.sendWelcome(user);
        audit.log("USER_REGISTERED", user.getId());
    }
}
```

Each concern lives in its own class. `UserService` only **orchestrates** the registration flow.

### SRP in Practice

- **Controller** — handles HTTP request/response mapping, delegates to service.
- **Service** — business logic and orchestration.
- **Repository** — data access only.
- **React component** — renders UI; logic extracted to custom hooks.

---

## O — Open/Closed Principle (OCP)

> Software entities should be **open for extension** but **closed for modification**.

You should be able to add new behavior **without changing existing code**.

### Violation

```java
public class DiscountCalculator {
    public double calculate(Order order) {
        if (order.getType() == OrderType.REGULAR) {
            return order.getTotal() * 0.05;
        } else if (order.getType() == OrderType.PREMIUM) {
            return order.getTotal() * 0.10;
        } else if (order.getType() == OrderType.VIP) {
            return order.getTotal() * 0.15;
        }
        // Adding a new order type requires modifying this class
        return 0;
    }
}
```

### Fix — Strategy Pattern

```java
public interface DiscountStrategy {
    double calculate(Order order);
}

public class RegularDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) { return order.getTotal() * 0.05; }
}

public class PremiumDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) { return order.getTotal() * 0.10; }
}

// Adding a new discount = adding a new class, no existing code changes
public class VipDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) { return order.getTotal() * 0.15; }
}

public class DiscountCalculator {
    private final DiscountStrategy strategy;

    public DiscountCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculate(Order order) {
        return strategy.calculate(order);
    }
}
```

### OCP in Practice

- **Spring:** adding a new `@Service` implementation without touching existing code. DI selects the right bean.
- **React:** composing behavior through props and children instead of `if/else` chains inside components.
- Design patterns that enable OCP: Strategy, Decorator, Template Method, Observer.

---

## L — Liskov Substitution Principle (LSP)

> Objects of a superclass should be replaceable with objects of a subclass **without breaking the program**.

If `B extends A`, then anywhere you use `A`, you should be able to use `B` without unexpected behavior.

### Classic Violation — Rectangle/Square

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int getArea()         { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w; // forces height = width — breaks Rectangle's contract
    }

    @Override
    public void setHeight(int h) {
        this.width = h;
        this.height = h;
    }
}

// Client code that works with Rectangle:
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.getArea() == 50; // FAILS for Square (area = 100)
}
```

**Fix:** Don't make `Square` extend `Rectangle`. Use a common interface `Shape` with `getArea()`, or make them
immutable (constructor sets both dimensions).

### Practical Violation

```java
public class ReadOnlyList<T> extends ArrayList<T> {
    @Override
    public boolean add(T element) {
        throw new UnsupportedOperationException(); // breaks ArrayList contract
    }
}
```

Code expecting `List` will break when it gets `ReadOnlyList`. **Fix:** Use `Collections.unmodifiableList()` or return
the `List` interface with clear documentation.

### LSP Rules

A subclass must:
- **Not strengthen preconditions** — don't accept fewer inputs than the parent.
- **Not weaken postconditions** — don't return less than what the parent promises.
- **Not throw new exceptions** that the parent didn't throw (checked exceptions in Java).
- **Preserve invariants** — if the parent guarantees something, the child must too.

---

## I — Interface Segregation Principle (ISP)

> No client should be forced to depend on methods it does not use. Prefer **many small, specific interfaces** over
> one large interface.

### Violation

```java
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// Robot doesn't eat or sleep — forced to implement meaningless methods
public class Robot implements Worker {
    @Override public void work()  { /* OK */ }
    @Override public void eat()   { throw new UnsupportedOperationException(); } // violation
    @Override public void sleep() { throw new UnsupportedOperationException(); } // violation
}
```

### Fix

```java
public interface Workable {
    void work();
}

public interface Feedable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class Human implements Workable, Feedable, Sleepable {
    @Override public void work()  { /* ... */ }
    @Override public void eat()   { /* ... */ }
    @Override public void sleep() { /* ... */ }
}

public class Robot implements Workable {
    @Override public void work() { /* ... */ }
    // No forced empty methods
}
```

### ISP in Practice

- **Spring:** `CrudRepository` vs `JpaRepository` — use the smallest interface that covers your needs.
- **TypeScript / React:** Props interfaces should be focused. Don't pass a fat `User` object when a component only
  needs `{ name: string }`.

```tsx
// ❌ Fat prop — component depends on entire User
function Avatar({ user }: { user: User }) {
    return <img src={user.avatarUrl} alt={user.name} />;
}

// ✅ Narrow prop — only what's needed
function Avatar({ avatarUrl, name }: { avatarUrl: string; name: string }) {
    return <img src={avatarUrl} alt={name} />;
}
```

---

## D — Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. **Both should depend on abstractions.**
> Abstractions should not depend on details. Details should depend on abstractions.

### Violation

```java
public class OrderService {
    private final MySQLOrderRepository repository = new MySQLOrderRepository(); // depends on concrete class

    public void placeOrder(Order order) {
        repository.save(order);
    }
}
```

`OrderService` (high-level) is tightly coupled to `MySQLOrderRepository` (low-level). Can't switch to PostgreSQL or
mock for testing.

### Fix

```java
// Abstraction — owned by the high-level module
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
}

// Low-level detail implements the abstraction
@Repository
public class MySQLOrderRepository implements OrderRepository {
    @Override public void save(Order order) { /* MySQL-specific */ }
    @Override public Optional<Order> findById(Long id) { /* ... */ }
}

// High-level depends on abstraction
@Service
public class OrderService {
    private final OrderRepository repository; // interface, not concrete class

    public OrderService(OrderRepository repository) { // injected by Spring
        this.repository = repository;
    }
}
```

**Direction of dependency:**

```
❌ Without DIP:  OrderService ──depends on──▶ MySQLOrderRepository

✅ With DIP:     OrderService ──depends on──▶ OrderRepository (interface)
                 MySQLOrderRepository ──implements──▶ OrderRepository (interface)
```

Both point toward the abstraction. The high-level module **owns** the interface.

### DIP in Practice

- **Spring DI** — the entire framework is built on DIP. Beans depend on interfaces, Spring injects implementations.
- **Testing** — depend on interface → inject mock in tests.
- **React** — dependency injection via props, Context, or hooks (e.g., inject a data-fetching function instead of
  hardcoding `fetch`).

---

## SOLID Summary — When Each Applies

| Principle | Smell / Trigger                                        | Refactoring                          |
|-----------|--------------------------------------------------------|--------------------------------------|
| SRP       | Class does too many things, changes for multiple reasons| Extract classes by responsibility    |
| OCP       | Adding features requires modifying existing code       | Introduce abstractions (strategy, DI)|
| LSP       | Subclass throws `UnsupportedOperationException`        | Rethink inheritance, prefer composition |
| ISP       | Implementors leave methods empty or throw              | Split into smaller interfaces        |
| DIP       | Class creates its own dependencies (`new ...`)         | Inject dependencies via constructor  |

## Common Interview Questions

1. **Name and explain all SOLID principles.** — See quick reference above. Be ready to give a one-liner and a code
   example for each.
2. **What is SRP? Give an example.** — One class, one responsibility, one reason to change. Example: separate
   validation, persistence, and notification into distinct classes.
3. **OCP — how to add behavior without modifying code?** — Use abstractions: Strategy pattern, polymorphism, DI.
   New behavior = new class implementing an existing interface.
4. **Give an example of LSP violation.** — Square extending Rectangle (overriding setWidth breaks Rectangle's
   contract). Or a read-only collection extending a mutable one.
5. **DIP vs DI (Dependency Injection)?** — DIP is the **principle** (depend on abstractions). DI is a **technique**
   to implement it (inject dependencies via constructor/setter). Spring's IoC container provides DI.
6. **How does SOLID relate to testability?** — SRP: smaller units are easier to test. DIP: inject mocks via
   interfaces. ISP: test only what you use. OCP: test new behavior without modifying old tests.

## Related

- [GRASP Principles](./grasp-principles.md) — complementary design principles for assigning responsibilities
- [Design Patterns](../Design-Patterns/) — patterns that implement SOLID (Strategy, Observer, Decorator)
- [Microservices Architecture](../../Java/spring/cloud/microservices-architecture.md) — SRP and DIP at service level

## Resources

- Robert C. Martin — *Clean Architecture* (chapters on SOLID)
- Robert C. Martin — [The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
- [Baeldung — SOLID Principles in Java](https://www.baeldung.com/solid-principles)
