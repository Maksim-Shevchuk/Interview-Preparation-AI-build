# Spring Core

Spring Core is the foundation of the entire Spring ecosystem. It provides the IoC container, dependency injection,
bean lifecycle management, AOP, and the event system. Almost every Spring interview starts with these concepts.

---

## Inversion of Control (IoC)

In traditional code, the application creates and manages its own dependencies. With IoC, the **container** takes
control — it creates objects, wires them together, manages their lifecycle, and disposes of them.

```java
// without IoC — tight coupling
public class OrderService {
    private final OrderRepository repo = new JdbcOrderRepository(dataSource);
}

// with IoC — dependency is injected
@Service
public class OrderService {
    private final OrderRepository repo;

    public OrderService(OrderRepository repo) {   // constructor injection
        this.repo = repo;
    }
}
```

The IoC container in Spring is represented by the `ApplicationContext` interface.

### ApplicationContext vs BeanFactory

| Feature                      | `BeanFactory`          | `ApplicationContext`             |
|------------------------------|------------------------|----------------------------------|
| Bean instantiation           | Lazy (on first access) | Eager (at startup by default)    |
| Event publishing             | No                     | Yes                              |
| Internationalization (i18n)  | No                     | Yes                              |
| AOP integration              | Manual                 | Automatic                        |
| Environment / profiles       | No                     | Yes                              |
| `BeanPostProcessor` auto-registration | No            | Yes                              |

`ApplicationContext` extends `BeanFactory`. In practice, always use `ApplicationContext` — `BeanFactory` is for
memory-constrained environments only.

Common implementations:
- `AnnotationConfigApplicationContext` — Java-based config.
- `ClassPathXmlApplicationContext` — XML-based config (legacy).
- `GenericWebApplicationContext` — web applications (Spring Boot uses this internally).

---

## Dependency Injection (DI)

### Injection Types

**Constructor injection (preferred):**

```java
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentService payments;

    // @Autowired is optional on a single constructor since Spring 4.3
    public OrderService(OrderRepository repo, PaymentService payments) {
        this.repo = repo;
        this.payments = payments;
    }
}
```

**Setter injection:**

```java
@Service
public class ReportService {
    private Formatter formatter;

    @Autowired
    public void setFormatter(Formatter formatter) {
        this.formatter = formatter;
    }
}
```

**Field injection (discouraged):**

```java
@Service
public class NotificationService {
    @Autowired
    private EmailSender sender;   // hard to test, hides dependencies
}
```

### Why Constructor Injection Is Preferred

1. **Immutability** — fields can be `final`.
2. **Explicit dependencies** — all required dependencies are visible in the constructor signature.
3. **Testability** — easy to pass mocks in unit tests without reflection.
4. **Fail-fast** — missing dependency causes an error at startup, not at runtime.
5. **No reflection** — Spring can invoke the constructor directly.

### `@Autowired` Resolution Order

When Spring looks for a bean to inject:

1. **By type** — find all beans matching the parameter type.
2. **By qualifier** — if `@Qualifier("name")` is present, filter by name.
3. **By name** — if multiple candidates exist, match the parameter name to a bean name.
4. **`@Primary`** — if still ambiguous, prefer the `@Primary` bean.
5. **Exception** — if still ambiguous, throw `NoUniqueBeanDefinitionException`.

### Resolving Ambiguity

```java
// @Primary — marks a default bean
@Primary
@Repository
public class JpaOrderRepository implements OrderRepository { }

@Repository
public class ElasticOrderRepository implements OrderRepository { }

// @Qualifier — explicit selection
@Service
public class SearchService {
    public SearchService(@Qualifier("elasticOrderRepository") OrderRepository repo) { }
}

// Inject all implementations
@Service
public class AggregateService {
    public AggregateService(List<OrderRepository> repos) { }   // all beans of this type
    // or Map<String, OrderRepository> — bean name as key
}
```

### `@Lazy`

Defers bean initialization until first access:

```java
@Service
@Lazy
public class ExpensiveService { }   // created on first injection/access, not at startup

// or on injection point
public OrderService(@Lazy ExpensiveService service) { }   // injects a proxy, real bean created on first call
```

---

## Bean Definition

### Stereotype Annotations

| Annotation      | Semantics                                | Layer          |
|-----------------|------------------------------------------|----------------|
| `@Component`    | Generic Spring-managed bean              | Any            |
| `@Service`      | Business logic                           | Service        |
| `@Repository`   | Data access + exception translation      | Persistence    |
| `@Controller`   | Spring MVC controller                    | Presentation   |
| `@RestController` | `@Controller` + `@ResponseBody`       | REST API       |
| `@Configuration`  | Declares `@Bean` factory methods       | Configuration  |

`@Service`, `@Repository`, `@Controller` are specializations of `@Component` — functionally identical for DI, but
provide **semantic meaning** and enable layer-specific features (e.g., `@Repository` adds persistence exception
translation via `PersistenceExceptionTranslationPostProcessor`).

### `@Bean` Methods

For beans you don't own (third-party libraries) or need custom creation logic:

```java
@Configuration
public class AppConfig {

    @Bean
    public RestClient restClient() {
        return RestClient.builder()
                .baseUrl("https://api.example.com")
                .build();
    }

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .build();
    }
}
```

### `@Configuration` — Full vs Lite Mode

```java
@Configuration       // full mode — CGLIB proxy, @Bean method calls are intercepted
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(commonDep());   // returns the SAME bean (singleton)
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(commonDep());   // returns the SAME bean (singleton)
    }

    @Bean
    public CommonDep commonDep() {
        return new CommonDep();
    }
}
```

With `@Configuration`, inter-`@Bean` method calls go through the CGLIB proxy, so `commonDep()` always returns the
**same singleton** instance.

With `@Component` (lite mode), each call to `commonDep()` creates a **new instance** — no proxy interception.

### Component Scanning

```java
@SpringBootApplication   // includes @ComponentScan for the package and sub-packages
public class MyApp { }

// explicit scanning
@ComponentScan(basePackages = "com.example.myapp")
```

---

## Bean Scopes

| Scope         | Instances              | Lifecycle                             |
|---------------|------------------------|---------------------------------------|
| `singleton`   | One per ApplicationContext (default) | Container startup → shutdown |
| `prototype`   | New instance per injection/lookup    | Created on demand, NOT destroyed by container |
| `request`     | One per HTTP request   | Request start → end (web only)        |
| `session`     | One per HTTP session   | Session creation → invalidation       |
| `application` | One per ServletContext | Application startup → shutdown        |
| `websocket`   | One per WebSocket session | WebSocket session lifetime         |

### Singleton ↔ Prototype Injection Problem

Injecting a `prototype` bean into a `singleton` — the prototype instance is resolved **once** at singleton creation
time and is reused forever. The prototype scope is effectively lost.

**Solutions:**

```java
// 1. ObjectFactory / ObjectProvider (preferred)
@Service
public class OrderService {
    private final ObjectProvider<ShoppingCart> cartProvider;

    public OrderService(ObjectProvider<ShoppingCart> cartProvider) {
        this.cartProvider = cartProvider;
    }

    public void process() {
        ShoppingCart cart = cartProvider.getObject();   // new instance each time
    }
}

// 2. @Lookup method injection
@Service
public abstract class OrderService {
    @Lookup
    protected abstract ShoppingCart createCart();   // Spring overrides this to return a new prototype

    public void process() {
        ShoppingCart cart = createCart();
    }
}

// 3. Scoped proxy
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class ShoppingCart { }
```

---

## Bean Lifecycle

### Full Lifecycle Sequence

```
1.  Bean definition loaded (from @Component, @Bean, XML)
2.  BeanFactoryPostProcessor runs (can modify bean definitions)
        ↓
3.  Bean instantiated (constructor called)
4.  Dependencies injected (setter / field injection)
5.  BeanPostProcessor.postProcessBeforeInitialization()
6.  @PostConstruct method
7.  InitializingBean.afterPropertiesSet()
8.  Custom init-method (@Bean(initMethod = "init"))
9.  BeanPostProcessor.postProcessAfterInitialization()  ← proxies created here (AOP, @Transactional)
        ↓
    === Bean is ready for use ===
        ↓
10. @PreDestroy method
11. DisposableBean.destroy()
12. Custom destroy-method (@Bean(destroyMethod = "cleanup"))
```

### Lifecycle Callbacks

```java
@Service
public class CacheService {

    @PostConstruct
    public void init() {
        // called after injection, before bean is available
        // use for: warm up cache, validate config, start background tasks
    }

    @PreDestroy
    public void shutdown() {
        // called before container destroys the bean
        // use for: flush cache, close connections, release resources
    }
}
```

- `@PostConstruct` / `@PreDestroy` (Jakarta annotations) — preferred approach.
- `InitializingBean` / `DisposableBean` — Spring interfaces, couple your code to Spring.
- `@Bean(initMethod, destroyMethod)` — for third-party beans you can't annotate.

### `BeanPostProcessor`

Intercepts bean creation — runs for **every** bean in the context. This is how Spring implements `@Autowired`,
`@Transactional`, `@Async`, `@Scheduled`, and AOP proxies.

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof MyService) {
            // wrap in a proxy, add logging, etc.
        }
        return bean;   // must return the bean (or a proxy wrapping it)
    }
}
```

### `BeanFactoryPostProcessor`

Runs **before** any beans are instantiated — can modify bean **definitions** (not instances):

```java
@Component
public class PropertyOverrider implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition def = factory.getBeanDefinition("dataSource");
        def.getPropertyValues().add("url", "jdbc:postgresql://new-host/db");
    }
}
```

`PropertySourcesPlaceholderConfigurer` (resolves `${...}` placeholders) is a `BeanFactoryPostProcessor`.

---

## Spring AOP (Aspect-Oriented Programming)

AOP separates cross-cutting concerns (logging, transactions, security, caching) from business logic.

### Terminology

| Term          | Meaning                                                       |
|---------------|---------------------------------------------------------------|
| **Aspect**    | A module of cross-cutting concern (`@Aspect` class)           |
| **Join point**| A point during execution (in Spring AOP — always a method call) |
| **Advice**    | Action taken at a join point (before, after, around)          |
| **Pointcut**  | Expression that matches join points                           |
| **Target**    | The object being proxied                                      |
| **Proxy**     | The wrapper object created by Spring (JDK dynamic or CGLIB)   |
| **Weaving**   | Linking aspects to target objects (at runtime in Spring)       |

### Advice Types

```java
@Aspect
@Component
public class LoggingAspect {

    // runs before the method
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint jp) {
        log.info("Calling: {}", jp.getSignature());
    }

    // runs after method returns successfully
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint jp, Object result) {
        log.info("Returned: {}", result);
    }

    // runs after method throws
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) {
        log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage());
    }

    // runs after method (regardless of outcome) — like finally
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint jp) { }

    // wraps the method — full control
    @Around("@annotation(Timed)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        Object result = pjp.proceed();
        long duration = System.nanoTime() - start;
        log.info("{} took {} ms", pjp.getSignature(), duration / 1_000_000);
        return result;
    }
}
```

### Pointcut Expressions

```java
// match by method signature
execution(* com.example.service.*.*(..))
//        │  └── package ────────┘ │ └ any args
//        └── any return type      └── any method

// match by annotation on method
@annotation(com.example.Timed)

// match by annotation on class
@within(org.springframework.stereotype.Service)

// match by bean name
bean(*Service)

// combine
@Pointcut("execution(* com.example.service.*.*(..)) && !execution(* *.get*(..))")
public void serviceMethods() { }
```

### Proxy Mechanism

Spring AOP creates a proxy that wraps the target bean:

```
Caller  →  Proxy  →  Advice chain  →  Target method
```

**Self-invocation problem:** when a method calls another method on the **same object**, it bypasses the proxy — AOP
advice is NOT applied. This is the same issue as with `@Transactional`.

```java
@Service
public class OrderService {

    @Timed
    public void processOrder() { }

    public void batchProcess() {
        processOrder();   // direct call — @Timed advice is NOT applied
    }
}
```

**Solutions:**
1. Extract the method into a separate bean.
2. Inject `self` reference: `@Lazy private OrderService self;` then call `self.processOrder()`.
3. Use `AopContext.currentProxy()` (requires `exposeProxy = true`).

### Spring AOP vs AspectJ

| Feature              | Spring AOP                | AspectJ                       |
|----------------------|---------------------------|-------------------------------|
| Weaving              | Runtime (proxies)         | Compile-time or load-time     |
| Join points          | Method execution only     | Fields, constructors, etc.    |
| Self-invocation      | Not intercepted           | Intercepted                   |
| Performance          | Proxy overhead            | No runtime overhead           |
| Complexity           | Simple setup              | Requires AspectJ compiler/agent |

---

## Profiles and Environment

### `@Profile`

Activate beans conditionally based on the active profile:

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:postgresql://prod-host/db")
                .build();
    }
}

// negation
@Profile("!prod")   // active in any profile except prod
```

Activate profiles:
- `spring.profiles.active=dev,metrics` in `application.properties`.
- `-Dspring.profiles.active=dev` as JVM argument.
- `@ActiveProfiles("test")` in tests.

### `@Conditional`

More granular than profiles — register a bean based on arbitrary conditions:

```java
@Bean
@ConditionalOnProperty(name = "feature.cache.enabled", havingValue = "true")
public CacheManager cacheManager() { }

@Bean
@ConditionalOnClass(name = "io.lettuce.core.RedisClient")
public RedisTemplate<String, Object> redisTemplate() { }

@Bean
@ConditionalOnMissingBean(CacheManager.class)
public CacheManager defaultCacheManager() { }
```

Spring Boot auto-configuration is built entirely on `@Conditional`.

---

## Events

### Application Events

Spring provides a publish-subscribe event system decoupled from the emitter:

```java
// define event
public record OrderCreatedEvent(Long orderId, String customerEmail) { }

// publish
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public OrderService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        publisher.publishEvent(new OrderCreatedEvent(order.getId(), request.email()));
        return order;
    }
}

// listen
@Component
public class NotificationListener {

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.customerEmail(), event.orderId());
    }
}
```

### `@TransactionalEventListener`

Listens only after a transaction phase completes:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    // runs only if the transaction that published the event commits successfully
    // avoids sending emails for rolled-back orders
}
```

| Phase              | When it runs                           |
|--------------------|----------------------------------------|
| `AFTER_COMMIT`     | After successful commit (default)      |
| `AFTER_ROLLBACK`   | After rollback                         |
| `AFTER_COMPLETION` | After commit or rollback               |
| `BEFORE_COMMIT`    | Before commit (still in the transaction) |

### `@Async` Events

By default, `@EventListener` runs **synchronously** in the publisher's thread. To run asynchronously:

```java
@Async
@EventListener
public void onOrderCreated(OrderCreatedEvent event) { }
```

Requires `@EnableAsync` on a `@Configuration` class.

---

## SpEL (Spring Expression Language)

A powerful expression language for querying and manipulating objects at runtime:

```java
@Value("${app.name}")                         // property placeholder
@Value("#{systemProperties['user.home']}")     // SpEL — system property
@Value("#{T(java.lang.Math).random()}")        // static method call
@Value("#{orderService.defaultCurrency}")      // bean property
@Value("#{2 * 3 + 1}")                         // arithmetic
@Value("#{someList.?[age > 18]}")              // collection filtering
```

Used in `@Value`, `@Cacheable(key = "...")`, `@PreAuthorize("...")`, `@ConditionalOnExpression("...")`.

---

## Common Interview Questions

### What is IoC and what problem does it solve?

IoC inverts the responsibility of creating and managing dependencies from the application to the container. This
provides: loose coupling (classes depend on abstractions, not implementations), testability (inject mocks), and
centralized configuration.

### What is the difference between `@Component` and `@Bean`?

- `@Component` — placed on a **class** you own, detected via component scanning.
- `@Bean` — placed on a **method** in a `@Configuration` class, returns an instance. Use for third-party classes or
  custom creation logic.

### Why is field injection bad?

1. Dependencies are hidden — not visible from the class API.
2. Fields can't be `final` — no immutability guarantee.
3. Hard to test — requires reflection or a Spring context, can't use a simple constructor.
4. Allows accumulating too many dependencies without noticing (no "constructor too large" warning).

### What is the difference between `singleton` and `prototype` scope?

- `singleton` — one instance per ApplicationContext, shared across all injection points. Container manages the full
  lifecycle.
- `prototype` — new instance for each injection or `getBean()` call. Container creates it but does **not** call
  `@PreDestroy` — the caller is responsible for cleanup.

### How does `@Transactional` relate to AOP?

`@Transactional` is implemented as an **Around advice** via `TransactionInterceptor`. The proxy intercepts the method
call, starts a transaction, invokes the real method, and commits or rolls back. This is why self-invocation bypasses
transactional behavior — the call doesn't go through the proxy.

### What is a `BeanPostProcessor` and when would you use one?

A `BeanPostProcessor` intercepts every bean after instantiation and injection. It can inspect, modify, or wrap beans
in proxies. Spring uses it internally for `@Autowired` resolution, `@Transactional` proxy creation, `@Scheduled`
registration, etc. Custom use cases: annotation processing, logging, validation.

### What is the difference between `@PostConstruct` and constructor logic?

The constructor runs **before** dependencies are injected (for setter/field injection). `@PostConstruct` runs **after**
all dependencies are injected, so you can safely use them. For constructor injection, the difference is minimal — but
`@PostConstruct` is still the standard place for initialization logic that goes beyond simple assignment.

### How do profiles work?

Profiles allow conditional bean registration. Beans annotated with `@Profile("dev")` are only loaded when the `dev`
profile is active. This enables environment-specific configurations (different data sources, feature flags, mock
services) without code changes. Multiple profiles can be active simultaneously.
