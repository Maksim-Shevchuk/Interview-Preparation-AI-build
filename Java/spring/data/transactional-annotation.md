# @Transactional Deep Dive

The single most-asked Spring question for Middle Java Full-Stack developers. Covers propagation, isolation, rollback rules, and the notorious self-invocation problem.

## What @Transactional Does

`@Transactional` demarcates a method (or class) so Spring wraps it in a database transaction. Spring uses **proxy-based AOP** — at runtime, a proxy intercepts the call, opens a transaction (via `PlatformTransactionManager`), invokes the actual method, and commits or rolls back based on outcome.

```java
@Service
public class OrderService {

    @Transactional
    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        inventoryService.decrement(request.items());   // same tx by default
        return order;
    }
}
```

## Proxy Types: JDK Dynamic Proxy vs CGLIB

Spring picks the proxy mechanism automatically:

| Mechanism             | When Used                                                                                        | Limitation                                                                             |
|-----------------------|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| **JDK dynamic proxy** | Bean implements an interface (and `proxyTargetClass = false`)                                    | Only interface methods are intercepted; the proxy cannot be cast to the concrete class |
| **CGLIB proxy**       | Bean has no interface, OR `spring.aop.proxy-target-class = true` (Spring Boot default since 2.x) | Cannot subclass `final` classes or override `final` methods                            |

**Spring Boot defaults to CGLIB** (`spring.aop.proxy-target-class=true`) so `@Transactional` works even when your `@Service` class doesn't implement any interface. The proxy is a CGLIB-generated subclass of your bean.

**Implications:**
- `@Transactional` on a `final` class or `final` method → silently ignored (CGLIB cannot override).
- Injecting the **interface** type (when present) is cleaner; injecting the concrete class works only because of CGLIB.

## Required Setup

For `@Transactional` to work, all the following must be true:

1. **Spring AOP on classpath** (`spring-boot-starter-data-jpa` brings it transitively).
2. **`@EnableTransactionManagement`** — auto-enabled by Spring Boot's `TransactionAutoConfiguration`.
3. **A `PlatformTransactionManager` bean** — `JpaTransactionManager` is auto-configured when JPA is present.
4. **Bean is managed by Spring** (not instantiated with `new`).
5. **Method is `public`** — Spring's proxy ignores non-public methods.
6. **Call comes from outside the bean** — see "Self-Invocation Problem" below.

## Transaction Propagation

Defines how a method participates in an existing transaction. Specified via `@Transactional(propagation = ...)`.

| Propagation          | Behavior                                                           |
|----------------------|--------------------------------------------------------------------|
| `REQUIRED` (default) | Join existing tx; create new if none. Most common.                 |
| `REQUIRES_NEW`       | Always create a new tx; suspend existing.                          |
| `NESTED`             | Create a nested savepoint within existing tx (DB must support it). |
| `MANDATORY`          | Must run within existing tx; throw if none.                        |
| `SUPPORTS`           | Use existing tx if present; else run non-transactional.            |
| `NOT_SUPPORTED`      | Suspend existing tx; run non-transactional.                        |
| `NEVER`              | Throw if a tx exists.                                              |

### When to Use REQUIRES_NEW

Use when you need an operation to commit **independently** of the outer transaction — typically for audit logs, notifications, or "must always succeed" side effects.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest req) {
        orderRepository.save(new Order(req));
        try {
            auditService.log(req);   // MUST commit even if outer tx rolls back
        } catch (Exception e) {
            log.warn("audit failed", e);
        }
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(OrderRequest req) {
        auditRepository.save(new AuditEntry(req));
    }
}
```

**Gotcha:** with JPA, the inner commit triggers a `flush` of the outer persistence context, which can cause `LazyInitializationException` or unexpected writes from the outer tx.

## Isolation Levels

Specify via `@Transactional(isolation = ...)`. Default is `Isolation.DEFAULT` — defers to the database's default (usually `READ_COMMITTED` in PostgreSQL, `REPEATABLE_READ` in MySQL).

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| `READ_UNCOMMITTED` | Yes | Yes | Yes |
| `READ_COMMITTED` | No | Yes | Yes |
| `REPEATABLE_READ` | No | No | Yes |
| `SERIALIZABLE` | No | No | No |

**Practical advice:** don't change isolation unless you have a specific reason. Most apps are fine with the DB default. Higher isolation = more locking = worse throughput.

## Rollback Rules

By default, Spring rolls back on **`RuntimeException` and `Error`**, but **NOT on checked exceptions**.

```java
@Transactional(rollbackFor = Exception.class)             // rollback on any exception
@Transactional(noRollbackFor = BusinessWarningException.class)  // never rollback
```

**Why the asymmetry?** Historical Spring design: checked exceptions usually represent recoverable business conditions (e.g., "insufficient funds"), not system failures. Override with `rollbackFor` when needed.

**Gotcha:** `rollbackFor` is checked **at compile time** via `@AliasFor` — `Exception.class` works, but custom exception subclasses are matched exactly. Use `rollbackFor = {IOException.class, SQLException.class}` to be explicit.

## Self-Invocation Problem ⚠️

**The most common Spring `@Transactional` bug.**

```java
@Service
public class OrderService {

    public void outer() {
        inner();   // ❌ BYPASSES the proxy! inner() runs WITHOUT a transaction
    }

    @Transactional
    public void inner() {
        orderRepository.save(new Order());
    }
}
```

**Why it fails:** the call `this.inner()` goes directly to the target object, bypassing the Spring proxy. The proxy is what triggers transaction management.

### Fixes

**1. Inject self via Spring (proxy-aware):**
```java
@Service
public class OrderService {

    private final OrderService self;  // injected proxy

    public OrderService(@Lazy OrderService self) {
        this.self = self;
    }

    public void outer() {
        self.inner();   // ✅ goes through proxy
    }

    @Transactional
    public void inner() { /* ... */ }
}
```

**2. Split into two beans:**
```java
@Service
public class OrderFacade {
    private final OrderTransactionalService txService;
    public void outer() { txService.inner(); }
}

@Service
public class OrderTransactionalService {
    @Transactional
    public void inner() { /* ... */ }
}
```

**3. Use `AopContext.currentProxy()`** (requires `@EnableAspectJAutoProxy(exposeProxy = true)`):
```java
public void outer() {
    ((OrderService) AopContext.currentProxy()).inner();
}
```

**General rule:** if you have `@Transactional` methods, call them only from **outside** the bean.

## After-Commit Hooks

For side effects that must run **only if the transaction commits** (e.g., send email, publish event, invalidate cache), do NOT put them inside the `@Transactional` method — the logic runs before commit and runs even on rollback.

### `@TransactionalEventListener`

Listens to events, but only fires after a specific transaction phase. Requires the publisher to be inside a transaction.

```java
// Inside @Transactional service
applicationEventPublisher.publishEvent(new OrderPlacedEvent(orderId));

// Listener — fires AFTER the outer tx commits, in a new tx by default
@Component
public class OrderEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.orderId());
    }
}
```

Phases: `BEFORE_COMMIT`, `AFTER_COMMIT` (default), `AFTER_ROLLBACK`, `AFTER_COMPLETION`.

### `TransactionSynchronizationManager`

Programmatic registration when you can't use events:

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            emailService.sendConfirmation(orderId);
        }
    });
```

Both options guarantee: the hook won't run if the tx rolls back.

## Common Pitfalls

### 1. Calling @Transactional Method from Same Class
See above — bypasses the proxy.

### 2. @Transactional on Private/Protected Method
Spring proxy only intercepts `public` methods. Annotation on private is silently ignored.

### 3. @Transactional on Final Class / Final Method
CGLIB proxy cannot subclass `final` classes or override `final` methods. Use JDK dynamic proxy (interface-based) or remove `final`.

### 4. Exception Caught Inside Method
```java
@Transactional
public void process() {
    try {
        doWork();
    } catch (Exception e) {
        log.error("oops", e);
        // ❌ Exception is swallowed → Spring sees a normal return → COMMITS
    }
}
```
The transaction is committed even though `doWork()` failed. Either re-throw or use `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

### 5. Read-Only Transactions for Performance
```java
@Transactional(readOnly = true)
public List<Order> findAll() {
    return orderRepository.findAll();
}
```
Hibernate skips dirty checking → faster. JDBC driver may use a different connection (e.g., read replica).

### 6. Transaction Timeout
```java
@Transactional(timeout = 5)   // seconds
```
Underlying DB statement is cancelled after the timeout. Default = unlimited.

### 7. Wrong DataSource in Multi-DS Apps
With multiple `DataSource` beans, the transaction manager binds to the **first** one found unless explicitly configured. Use `@Transactional("orderTransactionManager")` to disambiguate.

## Code Examples

### Service with Multiple Operations
```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(fromId);
        }

        from.debit(amount);
        to.credit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

### Programmatic Transactions
When `@Transactional` is awkward (dynamic logic, multiple boundaries):
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final TransactionTemplate txTemplate;
    private final OrderRepository orderRepository;

    public void placeOrder(OrderRequest req) {
        txTemplate.execute(status -> {
            orderRepository.save(new Order(req));
            return null;
        });
    }
}
```

## Reactive Transactions (R2DBC)

For reactive Spring (`WebFlux` + R2DBC), `@Transactional` works but is fundamentally different — it manipulates a **`ReactiveTransaction`** holding a `Connection`, not a `ThreadLocal`-bound session.

```java
@Service
public class OrderService {

    @Transactional
    public Mono<Order> placeOrder(OrderRequest req) {
        return orderRepository.save(new Order(req))
                .then(inventoryService.decrement(req.items()));
    }
}
```

**Rules:**
- Method must return `Mono` / `Flux` — Spring subscribes within the transactional scope.
- Operator chaining must not break the reactive chain (no `.block()`, no `CompletableFuture` bridge).
- Under the hood: `TransactionalOperator` / `R2dbcTransactionManager`, with the `Connection` propagated via Reactor `Context` (not `ThreadLocal`).
- `REQUIRES_NEW` suspends by switching `Context`, not by suspending a thread.

## Best Practices Summary

1. **Public methods only.** Annotation on private/protected is silently ignored.
2. **External calls only.** Self-invocation bypasses the proxy. Split beans or inject `@Lazy` self.
3. **No `final`** on `@Transactional` classes/methods (CGLIB limitation).
4. **Don't swallow exceptions** inside a `@Transactional` method — either re-throw or call `setRollbackOnly()`.
5. **Use `readOnly = true`** for read paths — measurable Hibernate speedup.
6. **Side effects after commit** → `@TransactionalEventListener(AFTER_COMMIT)`, never inline.
7. **Don't fight isolation levels** — use the DB default; escalate only with evidence.
8. **For long-running transactions** (batch jobs), set an explicit `timeout` to avoid holding connections/locks.
9. **Multi-datasource** → qualify the manager: `@Transactional("orderTransactionManager")`.
10. **Reactive stack** → return `Mono`/`Flux`, never mix blocking calls inside the chain.

## Common Interview Questions

1. **How does Spring's @Transactional work?**
   → Proxy-based AOP. Spring creates a proxy at bean creation time. The proxy intercepts calls to `@Transactional` methods, opens a transaction via `PlatformTransactionManager`, invokes the target, and commits/rolls back based on outcome.

2. **Why doesn't @Transactional work on internal method calls?**
   → `this.method()` bypasses the Spring proxy. The proxy is what triggers the transaction. Solutions: inject self, split into two beans, or use `AopContext.currentProxy()`.

3. **What's the difference between REQUIRED and REQUIRES_NEW?**
   → `REQUIRED` joins an existing tx or creates a new one. `REQUIRES_NEW` always creates a new tx, suspending any existing one — useful for audit logs that must commit independently.

4. **On which exceptions does Spring roll back by default?**
   → `RuntimeException` and `Error`. NOT checked exceptions. Override with `rollbackFor`.

5. **What isolation level should you use?**
   → Default (DB-specific). Don't change unless you have a specific concurrency requirement; higher isolation = more locking.

6. **Can @Transactional work on private methods?**
   → No. Spring's proxy only intercepts public methods. The annotation is silently ignored on private/protected.

7. **What's the difference between readOnly = true and a normal transaction?**
   → Hibernate skips dirty checking (faster). Some JDBC drivers route to a read replica. Spring signals to the tx manager that no writes are expected.

8. **How do you make @Transactional work in tests?**
   → Spring's `@DataJpaTest` rolls back transactions after each test. `@SpringBootTest` does not — use `@Transactional` on the test class to roll back, or commit explicitly.

9. **What happens if @Transactional is on the class level and also on a method?**
   → Method-level overrides class-level attributes. Both apply, with method taking precedence.

10. **What are the alternatives to @Transactional?**
    → Programmatic: `TransactionTemplate` or `PlatformTransactionManager` directly. Reactive: `@Transactional` works with R2DBC. For multi-resource transactions: JTA / `ChainedTransactionManager` (legacy) or `Atomikos` / `Narayana`.

11. **What's the difference between JDK dynamic proxy and CGLIB?**
    → JDK proxy requires an interface and can only intercept interface methods. CGLIB generates a subclass of the target class (no interface needed) but cannot proxy `final` classes/methods. Spring Boot defaults to CGLIB.

12. **How do you run code only after the transaction commits?**
    → Use `@TransactionalEventListener(phase = AFTER_COMMIT)` on an event listener, or register a `TransactionSynchronization` via `TransactionSynchronizationManager.registerSynchronization(...)`. Both skip the hook on rollback.

13. **Does @Transactional work with reactive code (WebFlux)?**
    → Yes, but the mechanism differs — it uses `TransactionalOperator` / `R2dbcTransactionManager`, propagates the `Connection` via Reactor `Context` (not `ThreadLocal`). The method must return `Mono`/`Flux` and the chain must not be broken by `.block()`.

14. **Why does @Transactional fail silently on `final` methods?**
    → CGLIB creates a subclass at runtime to override the method; `final` methods cannot be overridden, so the proxy's transactional advice never runs. The annotation is silently ignored, not an error.

15. **What's the difference between `FetchGraph` and `LoadGraph` JPA hints?**
    → (Trick question — belongs to `n-plus-one-problem.md`, but commonly asked alongside.) `FetchGraph` loads only graph attributes; `LoadGraph` loads graph attributes plus any EAGER ones from the mapping.

## Related

- `Java/spring/data/n-plus-one-problem.md` — JPA performance
- `Databases/Transactions/` — ACID theory, isolation levels
- `Java/spring/core/aop.md` — how Spring AOP works
- `Java/spring/boot/autoconfiguration.md` — transaction manager setup

## Resources

- **Spring docs:** https://docs.spring.io/spring-framework/reference/data-access.html
- **Baeldung:** https://www.baeldung.com/transaction-configuration-with-jpa-and-spring
- **"Spring in Action" (Craig Walls):** chapters on data access
- **OpenJDK source:** `JpaTransactionManager`, `TransactionAspectSupport`
