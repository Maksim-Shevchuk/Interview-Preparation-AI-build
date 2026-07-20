# Hexagonal Architecture (Ports and Adapters)

Architectural pattern introduced by **Alistair Cockburn** in 2005 (also known as **Ports and Adapters**). It isolates
an application's **business logic** from external concerns (UI, databases, message brokers, third-party APIs) by
treating every external interaction as a **pluggable adapter** that communicates with the core through well-defined
**ports**. The goal: write the domain once, run it anywhere, test it without infrastructure.

A frequent senior-level interview topic — especially in Java/Spring shops — alongside Clean Architecture, Onion, and
DDD.

---

## Quick Reference

| Term                        | Meaning                                                                 |
|-----------------------------|-------------------------------------------------------------------------|
| **Hexagon**                 | The application core; "hexagonal" is symbolic — the number of sides is arbitrary |
| **Domain / Core**           | Business logic and entities; has zero dependencies on the outside world |
| **Port**                    | An interface that defines a contract of the core (a "slot")             |
| **Driving (primary) port**  | Interface the outside world calls to **use** the application            |
| **Driven (secondary) port** | Interface the core calls to **be served** by the outside world          |
| **Adapter**                 | Concrete implementation that plugs into a port                          |
| **Driving adapter**         | REST controller, CLI, gRPC endpoint, Kafka consumer, job scheduler      |
| **Driven adapter**          | JPA repository, Redis client, SMTP mailer, Stripe client, file gateway  |

---

## 1. The Problem It Solves

In a classic **layered architecture** (presentation → service → persistence), business logic tends to **leak**
dependencies: services import JPA repositories, JPA entities, Spring annotations, HTTP objects. Consequences:

- **Domain coupled to infrastructure.** Swapping PostgreSQL for MongoDB, or REST for gRPC, becomes a rewrite.
- **Hard to test.** To unit-test a service you must bootstrap Spring, H2, mocks, etc.
- **Blurry boundaries.** `@Entity` classes double as DTOs and domain models; transactional annotations live next to
  invariants.
- **Lateral coupling.** A "service" class often calls repositories of unrelated aggregates.

Hexagonal architecture reverses the dependency direction: **the core defines what it needs, the outside world fits
into it.**

---

## 2. The Hexagon Metaphor

```
                                   ┌───────────────────────────┐
   Driving adapters                │      Driving ports        │
   (left side — actors             │   (API the core exposes)  │
   that USE the app)               │                           │
                                   │   ┌───────────────────┐   │
  ┌────────────┐                   │   │                   │   │
  │   REST     │──────── use ─────▶│   │                   │   │
  │ Controller │                   │   │                   │   │
  └────────────┘                   │   │      DOMAIN       │   │
                                   │   │      (CORE)       │   │
  ┌────────────┐                   │   │  entities, value  │   │
  │    CLI     │──────── use ─────▶│   │  objects, domain  │   │
  └────────────┘                   │   │  services, use    │   │
                                   │   │  cases             │   │
  ┌────────────┐                   │   │                   │   │
  │  gRPC      │──────── use ─────▶│   │                   │   │
  └────────────┘                   │   └───────────────────┘   │
                                   │                           │
                                   │     Driven ports          │
                                   │  (API the core NEEDS)     │
   Driven adapters                 │                           │
   (right side — tools             └───┬───────┬───────┬────────┘
   the app USES)                       │       │       │
                                       ▼       ▼       ▼
  ┌────────────┐  ◀──── serves ────  JPA     SMTP     Stripe
  │ PostgreSQL │                     adapter  adapter  adapter
  └────────────┘
```

The hexagon shape is **symbolic**: a hexagon has 6 sides, but your application can have any number of ports. The point
is that there is no privileged "top" or "bottom" — every external system sits on an equal footing as a peer adapter.

> **Mental model:** the core is a sealed box with sockets (ports). You plug devices (adapters) into the sockets. The
> box never knows what is plugged in.

---

## 3. The Core Entities

### 3.1 Domain / Application Core

The innermost part. Contains:

- **Entities (Aggregates)** — domain objects with identity and invariants (e.g., `Order`, `Account`).
- **Value Objects** — immutable, side-effect-free descriptors (e.g., `Money`, `EmailAddress`).
- **Domain Services** — operations that don't naturally belong to a single entity (e.g., `TransferService`).
- **Use Cases (Application Services)** — orchestrate entities and ports to fulfill a single intent
  (e.g., `PlaceOrderUseCase`).

Rules of the core:

- **No framework imports.** No `@Entity`, no `@Service`, no `HttpServletRequest`, no JPA annotations. Pure Java/Kotlin
  classes.
- **No I/O.** No `System.out`, no file reads, no HTTP calls.
- **Dependencies point inward only.** The core declares what it needs via ports; it never imports adapters.

### 3.2 Ports

A **port** is an **interface** defined inside the core. It is the only way to cross the hexagon boundary.

Two flavors:

| Kind          | Direction of the call       | Owner of the call    | Example                                    |
|---------------|-----------------------------|----------------------|--------------------------------------------|
| **Driving**   | Outside ➜ Core              | External actor       | `CreateOrderUseCase` (called by controller)|
| **Driven**    | Core ➜ Outside              | Core (use case)      | `OrderRepository`, `EmailGateway`          |

- A **driving port** describes a *capability of the application* ("what you can do with the app"). Implemented by the
  application core itself (use case).
- A **driven port** describes a *capability the application needs from the outside* ("what the app needs"). Implemented
  by an adapter.

### 3.3 Adapters

An **adapter** is a concrete technology that implements (driving) or uses (driven) a port.

**Driving (primary) adapters** — translate external input into calls on driving ports:

- REST controllers (`@RestController`)
- gRPC service implementations
- CLI commands
- Kafka consumers / SQS listeners / scheduled jobs

**Driven (secondary) adapters** — implement driven ports using real technology:

- JPA repositories that implement `OrderRepository`
- SMTP-based `EmailGateway`
- Stripe SDK-based `PaymentGateway`
- File-system or in-memory implementations (useful for tests)

---

## 4. Dependency Rule

```
   Outer (infrastructure) ─────────────▶ Inner (domain)

   Adapters depend on ports.
   Ports and adapters NEVER depend on each other directly — only through the interface.
   The core depends on nothing external.
```

This is the **Dependency Inversion Principle** applied at the architectural level:
high-level policy (core) does not depend on low-level detail (adapter); both depend on an abstraction (the port).

---

## 5. Worked Example (Java)

### 5.1 Domain

```java
// CORE — no framework, no I/O
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.signum() < 0) throw new IllegalArgumentException("Money cannot be negative");
    }
    public Money plus(Money other) { /* ... */ }
}

public final class Account {
    private final AccountId id;
    private Money balance;

    public void withdraw(Money amount) {
        if (balance.amount().compareTo(amount.amount()) < 0) {
            throw new InsufficientFundsException(id);
        }
        this.balance = balance.minus(amount);
    }

    public void deposit(Money amount) { this.balance = balance.plus(amount); }
}
```

### 5.2 Driven Ports (what the core needs)

```java
public interface AccountRepository {
    Optional<Account> findById(AccountId id);
    void save(Account account);
}

public interface AuditLog {
    void record(String event, AccountId id);
}
```

### 5.3 Driving Port (a use case — capability exposed)

```java
public interface TransferMoneyUseCase {
    void execute(TransferCommand command);
}
```

### 5.4 Use Case Implementation (inside the core)

```java
public class TransferMoneyService implements TransferMoneyUseCase {
    private final AccountRepository accounts;   // port, not JPA!
    private final AuditLog audit;               // port

    public TransferMoneyService(AccountRepository accounts, AuditLog audit) {
        this.accounts = accounts;
        this.audit = audit;
    }

    @Override
    public void execute(TransferCommand cmd) {
        Account from = accounts.findById(cmd.from()).orElseThrow(AccountNotFoundException::new);
        Account to   = accounts.findById(cmd.to()).orElseThrow(AccountNotFoundException::new);

        from.withdraw(cmd.amount());   // pure domain logic
        to.deposit(cmd.amount());

        accounts.save(from);
        accounts.save(to);
        audit.record("TRANSFER", cmd.from());
    }
}
```

### 5.5 Driven Adapter (outside the core, in infrastructure)

```java
// JPA adapter — implements the port, depends on the port
@Repository
public class JpaAccountRepository implements AccountRepository {
    private final AccountJpaRepository jpa;

    public JpaAccountRepository(AccountJpaRepository jpa) { this.jpa = jpa; }

    @Override
    public Optional<Account> findById(AccountId id) {
        return jpa.findById(id.value()).map(AccountEntity::toDomain);
    }

    @Override
    public void save(Account account) {
        jpa.save(AccountEntity.fromDomain(account));
    }
}
```

### 5.6 Driving Adapter (REST controller)

```java
@RestController
@RequestMapping("/api/transfers")
public class TransferController {
    private final TransferMoneyUseCase useCase;   // depends on port, not impl

    public TransferController(TransferMoneyUseCase useCase) { this.useCase = useCase; }

    @PostMapping
    public ResponseEntity<Void> transfer(@RequestBody TransferDto dto) {
        useCase.execute(new TransferCommand(dto.from(), dto.to(), new Money(dto.amount(), Currency.USD)));
        return ResponseEntity.accepted().build();
    }
}
```

### 5.7 Wiring (Spring, application config)

```java
@Configuration
public class ApplicationConfig {

    @Bean
    TransferMoneyUseCase transferMoneyUseCase(AccountRepository accounts, AuditLog audit) {
        return new TransferMoneyService(accounts, audit);   // core
    }

    @Bean
    AccountRepository accountRepository(AccountJpaRepository jpa) {
        return new JpaAccountRepository(jpa);   // adapter
    }

    @Bean
    AuditLog auditLog(LoggingAuditLog log) {
        return log;
    }
}
```

### 5.8 Test (no Spring, no DB)

```java
class TransferMoneyServiceTest {
    private final InMemoryAccountRepository repo = new InMemoryAccountRepository();
    private final List<String> events = new ArrayList<>();
    private final AuditLog audit = (event, id) -> events.add(event);

    @Test
    void transfers_between_accounts() {
        repo.save(new Account(new AccountId("A"), new Money("100", USD)));
        repo.save(new Account(new AccountId("B"), new Money("0",  USD)));

        var useCase = new TransferMoneyService(repo, audit);
        useCase.execute(new TransferCommand(new AccountId("A"), new AccountId("B"), new Money("40", USD)));

        assertThat(repo.findById(new AccountId("A")).get().balance().amount()).isEqualByComparingTo("60");
        assertThat(events).containsExactly("TRANSFER");
    }
}
```

> Notice: the entire business flow is tested with **plain JUnit** and a fake adapter — no `@SpringBootTest`, no
> Testcontainers, no mocks of frameworks. This is the headline benefit of the pattern.

---

## 6. Compared with Related Patterns

| Aspect               | Layered                  | Hexagonal                     | Clean Architecture          | Onion                       |
|----------------------|--------------------------|-------------------------------|-----------------------------|-----------------------------|
| Dependency direction | Top ➜ bottom             | Adapter ➜ port ➜ core         | Outer ring ➜ inner ring     | Outer layer ➜ inner layer   |
| Central concept      | Layers                   | Ports & adapters              | Concentric rings            | Layers around a domain      |
| Persistence          | Always bottom layer      | Just another driven adapter   | Outermost detail            | Outermost service           |
| UI/DB swapability    | Hard                     | Easy (that's the point)       | Easy                        | Easy                        |
| Author               | Traditional              | Alistair Cockburn (2005)      | Robert C. Martin (2012)     | Jeffrey Palermo (2008)      |
| Distinct feature     | Separation of concerns   | Symmetry of ports             | Strict layer boundaries     | Domain model in the center  |

All four apply the **Dependency Inversion Principle** at architectural scale. Hexagonal is the simplest to explain;
Clean/Onion layer additional constraints on testability and layering.

---

## 7. Pros and Cons

### Advantages

- **Technology independence.** Swap Postgres for Mongo, REST for gRPC, deploy a CLI version — the core is untouched.
- **Testability.** The whole domain is testable without infrastructure. Use cases are exercised with fake adapters.
- **Clear boundaries.** Each port is a deliberate decision about what crosses the hexagon.
- **Late decisions.** You can start building with in-memory adapters and choose frameworks later.
- **Parallel development.** Front-end and adapter teams can code against port contracts before the core is finished.
- **Suits DDD.** Aggregates and domain services fit naturally inside the hexagon.

### Disadvantages

- **More boilerplate.** Every capability needs an interface (port) and one or more implementations.
- **Indirection.** Reading a request flow jumps: controller ➜ use case interface ➜ service ➜ repository interface ➜
  JPA adapter. Newcomers find this noisy.
- **Overkill for simple CRUD.** If your app is a thin wrapper over a database, the pattern adds friction without payoff.
- **Mapping cost.** Domain objects ↔ JPA entities ↔ DTOs requires explicit conversions (no leaking the entity to the
  controller).
- **Learning curve.** The team must understand dependency inversion, port/adapter distinction, and DDD building blocks.

---

## 8. When to Use / When Not

**Use hexagonal architecture when:**

- Business logic is non-trivial (rules, invariants, workflows), not just CRUD.
- The application must support multiple "fronts" (REST + gRPC + CLI + messaging consumers).
- You need to swap infrastructure (database migrations, cloud vendor changes).
- Long-lived applications where maintenance cost dominates initial build cost.
- Domains where DDD is appropriate (finance, e-commerce checkout, logistics).

**Avoid it when:**

- Pure CRUD / admin tooling where the database is the system.
- A throwaway prototype or MVP where speed matters more than flexibility.
- Small team unfamiliar with DI and architectural patterns.
- The "domain" is genuinely thin — you'll be writing ports for nothing.

---

## 9. Common Interview Questions

1. **What is hexagonal architecture, and why is it called that?**
   Coined by Alistair Cockburn in 2005; the hexagon symbolizes that the application can have many equivalent ports,
   not just "top" and "bottom" layers. Also called Ports and Adapters.

2. **What problem does it solve?**
   It decouples the domain from infrastructure so the core can be tested in isolation and so technologies can be
   swapped without rewriting business logic.

3. **What is the difference between a port and an adapter?**
   A port is an **interface** (contract) defined by the core. An adapter is a **concrete implementation** in
   infrastructure that either uses the core (driving) or serves the core (driven).

4. **Driving vs. driven — which is which?**
   **Driving** (primary) adapters initiate the action: REST controllers, CLI, message consumers. They call the core
   through **driving ports** (use case interfaces). **Driven** (secondary) adapters are called by the core through
   **driven ports** (repository / gateway interfaces): JPA, SMTP, Stripe.

5. **How does it differ from Clean Architecture?**
   Same underlying principle (dependency inversion, isolate the domain). Clean Architecture formalizes concentric
   rings (entities, use cases, interface adapters, frameworks) with strict boundaries; hexagonal emphasizes the
   symmetric concept of ports and is less prescriptive about internal layering of the core.

6. **Why is the domain layer framework-free?**
   So that business logic is expressed purely and so that changes in frameworks (Spring, JPA, Jackson) cannot ripple
   into the domain. The core stays stable across technology changes.

7. **How do entities map to JPA if you can't annotate them?**
   Keep separate persistence entities in the adapter layer. Convert between domain objects and persistence entities at
   the adapter boundary. Yes, this is extra mapping — it is the price of decoupling.

8. **How does it relate to the Dependency Inversion Principle?**
   The DIP states "depend on abstractions, not concretions." Hexagonal architecture applies DIP at the architectural
   scale: adapters depend on ports defined in the core, never the other way around.

9. **Trade-offs of the pattern?**
   More boilerplate (interfaces + implementations + mappings), indirection, and overkill for CRUD apps. Payoff comes
   with complex domains, multiple consumers, or long-lived systems.

10. **Where does Spring `@Transactional` belong?**
    On the **driving adapter** or a dedicated application-layer wrapper around the use case, not inside the domain.
    Transactions are infrastructure concerns and must not pollute the core.

---

## 10. Mental Cheat-Sheet

> **One core, two kinds of ports, two kinds of adapters.**
> The **core** defines **ports** (interfaces). Adapters plug into them.
> **Driving** = someone calls *in* (controller → use case).
> **Driven** = the core calls *out* (use case → repository).
> Dependencies always point **inward**.

```
                        Driving adapters call IN
                                 ▼
                  ┌──────────────────────────────┐
                  │        DRIVING  PORTS        │
                  │   (use-case interfaces)      │
                  │  ┌────────────────────────┐  │
  Driven adapters │  │                        │  │ Driven adapters
  serve OUT       ◀  │     DOMAIN / CORE      │  ▶ are called OUT
                  │  │  entities, use cases,  │  │
                  │  │  domain services       │  │
                  │  └────────────────────────┘  │
                  │        DRIVEN   PORTS        │
                  │  (repository/gateway ifaces) │
                  └──────────────────────────────┘
```
