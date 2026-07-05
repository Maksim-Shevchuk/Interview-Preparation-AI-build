# Spring Boot

Spring Boot is an opinionated framework built on top of Spring that eliminates boilerplate configuration. It provides
auto-configuration, embedded servers, production-ready features, and a convention-over-configuration approach.
Interviews focus on how auto-configuration works, configuration properties, starters, Actuator, and testing.

---

## What Spring Boot Adds Over Spring

| Concern                    | Plain Spring                                | Spring Boot                             |
|----------------------------|---------------------------------------------|-----------------------------------------|
| Configuration              | Manual `@Bean`, XML, or Java config         | Auto-configuration based on classpath   |
| Server                     | Deploy WAR to external Tomcat/Jetty         | Embedded server, run as JAR             |
| Dependency management      | Manual version alignment                    | Starter POMs with curated versions      |
| Production readiness       | Build yourself                              | Actuator (health, metrics, info)        |
| Properties                 | `PropertySourcesPlaceholderConfigurer`      | `@ConfigurationProperties`, YAML support|
| Entry point                | Configure `DispatcherServlet`, listeners    | `@SpringBootApplication` + `main()`    |

---

## `@SpringBootApplication`

A meta-annotation that combines three annotations:

```java
@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

| Included annotation         | What it does                                              |
|------------------------------|-----------------------------------------------------------|
| `@Configuration`             | Marks the class as a source of `@Bean` definitions        |
| `@EnableAutoConfiguration`   | Activates Spring Boot auto-configuration                  |
| `@ComponentScan`             | Scans the package (and sub-packages) for `@Component` beans |

### What `SpringApplication.run()` Does

1. Creates the appropriate `ApplicationContext` (servlet, reactive, or none).
2. Loads `application.properties` / `application.yml`.
3. Runs all `ApplicationContextInitializer` instances.
4. Performs component scanning and registers bean definitions.
5. Triggers auto-configuration.
6. Instantiates all singleton beans.
7. Starts the embedded web server (if web application).
8. Publishes `ApplicationReadyEvent`.

---

## Auto-Configuration

### How It Works

Spring Boot auto-configuration is powered by `@Conditional` annotations and the `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file (prior to 3.0: `spring.factories`).

**Flow:**

```
Classpath scanned at startup
        │
        ▼
AutoConfiguration classes loaded from META-INF imports file
        │
        ▼
Each class evaluated: @ConditionalOnClass, @ConditionalOnProperty, @ConditionalOnMissingBean, etc.
        │
        ▼
Matching configurations register their @Bean definitions
```

**Example — `DataSourceAutoConfiguration`:**

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)                      // HikariCP on classpath?
@ConditionalOnMissingBean(DataSource.class)                // user hasn't defined their own?
@EnableConfigurationProperties(DataSourceProperties.class) // bind spring.datasource.* properties
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

### Key `@Conditional` Annotations

| Annotation                    | Condition                                            |
|-------------------------------|------------------------------------------------------|
| `@ConditionalOnClass`         | Class is on the classpath                            |
| `@ConditionalOnMissingClass`  | Class is NOT on the classpath                        |
| `@ConditionalOnBean`          | Bean of this type exists                             |
| `@ConditionalOnMissingBean`   | Bean of this type does NOT exist                     |
| `@ConditionalOnProperty`      | Property has a specific value                        |
| `@ConditionalOnResource`      | Resource exists on the classpath                     |
| `@ConditionalOnWebApplication`| Application is a web application                     |
| `@ConditionalOnExpression`    | SpEL expression evaluates to true                    |

### Overriding Auto-Configuration

Auto-configuration backs off when you define your own beans — `@ConditionalOnMissingBean` ensures your explicit
configuration always takes priority:

```java
@Configuration
public class MyDataSourceConfig {

    @Bean   // auto-configured DataSource backs off because this bean exists
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:postgresql://custom-host/db")
                .build();
    }
}
```

### Debugging Auto-Configuration

```bash
# show auto-configuration report
java -jar myapp.jar --debug

# or in application.properties
debug=true
```

This prints a **CONDITIONS EVALUATION REPORT** — which auto-configurations matched and which didn't, and why.

---

## Starters

Starters are curated dependency sets — a single dependency that pulls in everything you need for a feature:

| Starter                               | Brings in                                            |
|---------------------------------------|------------------------------------------------------|
| `spring-boot-starter-web`             | Tomcat, Spring MVC, Jackson                          |
| `spring-boot-starter-webflux`         | Netty, Spring WebFlux, Reactor                       |
| `spring-boot-starter-data-jpa`        | Hibernate, Spring Data JPA, HikariCP                 |
| `spring-boot-starter-data-redis`      | Lettuce, Spring Data Redis                           |
| `spring-boot-starter-security`        | Spring Security, auto-configured filter chain        |
| `spring-boot-starter-actuator`        | Actuator endpoints, Micrometer                       |
| `spring-boot-starter-validation`      | Hibernate Validator, Jakarta Validation              |
| `spring-boot-starter-test`            | JUnit 5, Mockito, AssertJ, Spring Test, Testcontainers |
| `spring-boot-starter-cache`           | Spring Cache abstraction                             |
| `spring-boot-starter-mail`            | Jakarta Mail                                         |
| `spring-boot-starter-amqp`           | RabbitMQ, Spring AMQP                                 |

All starters inherit from `spring-boot-starter` (core: Spring Boot, auto-configuration, logging, YAML).

Version management is handled by the `spring-boot-dependencies` BOM — you declare starters without specifying versions.

---

## Configuration Properties

### Property Sources (Priority, highest first)

1. Command-line arguments (`--server.port=9090`).
2. `SPRING_APPLICATION_JSON` (inline JSON).
3. Servlet config parameters.
4. OS environment variables (`SERVER_PORT=9090`).
5. Profile-specific files (`application-{profile}.properties`).
6. `application.properties` / `application.yml` (outside JAR, then inside JAR).
7. `@PropertySource` on `@Configuration` classes.
8. Default properties (`SpringApplication.setDefaultProperties()`).

### `application.properties` vs `application.yml`

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost/mydb
spring.datasource.username=postgres
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost/mydb
    username: postgres
```

Functionally equivalent. YAML supports hierarchical structure and multi-document files (`---` separator).

### Profile-Specific Configuration

```
application.properties           # default (always loaded)
application-dev.properties       # loaded when "dev" profile is active
application-prod.properties      # loaded when "prod" profile is active
```

Profile-specific properties **override** default properties. Activate with:
- `spring.profiles.active=dev` in `application.properties`.
- `--spring.profiles.active=dev` on command line.
- `SPRING_PROFILES_ACTIVE=dev` as environment variable.

### `@ConfigurationProperties`

Type-safe binding of properties to a POJO:

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
        String host,
        int port,
        boolean ssl,
        Duration timeout,
        List<String> recipients
) { }
```

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    ssl: true
    timeout: 5s
    recipients:
      - admin@example.com
      - ops@example.com
```

Register with `@EnableConfigurationProperties(MailProperties.class)` or annotate the class with `@Component`.

**Advantages over `@Value`:**
- Type-safe — validated at startup, IDE auto-completion with `spring-boot-configuration-processor`.
- Structured — nested objects, lists, maps.
- Immutable — use records or `@ConstructorBinding`.
- Testable — instantiate without Spring context.

### Relaxed Binding

Spring Boot matches properties regardless of case or separator style:

| Property in config                  | Matches Java field      |
|-------------------------------------|-------------------------|
| `app.mail-host`                     | `mailHost`              |
| `app.mailHost`                      | `mailHost`              |
| `app.mail_host`                     | `mailHost`              |
| `APP_MAILHOST` (env var)            | `mailHost`              |

### Property Validation

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {

    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port;

    @DurationMin(seconds = 1)
    private Duration timeout;
}
```

Validation errors cause a startup failure with a clear error message.

---

## Embedded Web Server

Spring Boot packages an embedded server inside the JAR — no external server required:

| Server   | Starter                        | Default? | Protocol     |
|----------|--------------------------------|----------|--------------|
| Tomcat   | `spring-boot-starter-web`      | Yes      | Servlet      |
| Jetty    | Exclude Tomcat, add Jetty      | No       | Servlet      |
| Undertow | Exclude Tomcat, add Undertow   | No       | Servlet      |
| Netty    | `spring-boot-starter-webflux`  | Yes      | Reactive     |

### Server Configuration

```yaml
server:
  port: 8080                          # 0 = random port
  address: 0.0.0.0
  servlet:
    context-path: /api
  tomcat:
    threads:
      max: 200
      min-spare: 10
    max-connections: 8192
    accept-count: 100
    connection-timeout: 20s
  shutdown: graceful                   # wait for in-flight requests on shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s    # max wait time for graceful shutdown
```

### Graceful Shutdown

With `server.shutdown=graceful`:
1. Stop accepting new requests.
2. Wait for in-flight requests to complete (up to `timeout-per-shutdown-phase`).
3. Shut down the server.

---

## Actuator

Production-ready monitoring and management endpoints:

### Key Endpoints

| Endpoint          | Purpose                                            | Default |
|-------------------|----------------------------------------------------|---------|
| `/actuator/health`  | Application health (DB, disk, custom checks)     | Enabled |
| `/actuator/info`    | Application info (build, git, custom)            | Enabled |
| `/actuator/metrics` | Micrometer metrics (JVM, HTTP, custom)           | Enabled |
| `/actuator/env`     | Environment properties                            | Disabled|
| `/actuator/beans`   | All beans in the ApplicationContext               | Disabled|
| `/actuator/mappings` | All `@RequestMapping` endpoints                 | Disabled|
| `/actuator/loggers` | View and change log levels at runtime            | Disabled|
| `/actuator/threaddump` | JVM thread dump                               | Disabled|
| `/actuator/heapdump`  | Heap dump (download)                            | Disabled|
| `/actuator/startup`   | Startup steps timeline                          | Disabled|
| `/actuator/prometheus` | Metrics in Prometheus format                   | Disabled|

### Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus    # which endpoints to expose over HTTP
  endpoint:
    health:
      show-details: when_authorized                # never | when_authorized | always
      show-components: always
  health:
    diskspace:
      enabled: true
    db:
      enabled: true
```

### Custom Health Indicator

```java
@Component
public class CacheHealthIndicator implements HealthIndicator {

    private final CacheService cacheService;

    public CacheHealthIndicator(CacheService cacheService) {
        this.cacheService = cacheService;
    }

    @Override
    public Health health() {
        if (cacheService.isAvailable()) {
            return Health.up()
                    .withDetail("hitRate", cacheService.getHitRate())
                    .build();
        }
        return Health.down()
                .withDetail("error", "Cache unavailable")
                .build();
    }
}
```

### Custom Metrics

```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = registry.counter("orders.created", "type", "online");
        this.orderTimer = registry.timer("orders.processing.time");
    }

    public Order createOrder(OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

## Logging

Spring Boot uses **SLF4J + Logback** by default:

```yaml
logging:
  level:
    root: WARN
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/app.log
  logback:
    rollingpolicy:
      max-file-size: 10MB
      max-history: 30
```

Switch to Log4j2: exclude `spring-boot-starter-logging`, add `spring-boot-starter-log4j2`.

---

## Error Handling

### Default Error Handling

Spring Boot provides a default `BasicErrorController` at `/error` that renders:
- JSON response for REST clients (`Accept: application/json`).
- HTML Whitelabel error page for browsers.

### Custom Error Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(EntityNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        problem.setTitle("Resource Not Found");
        problem.setDetail(ex.getMessage());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(
                        FieldError::getField,
                        f -> f.getDefaultMessage() != null ? f.getDefaultMessage() : "invalid"
                ));
        problem.setProperty("errors", errors);
        return problem;
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ProblemDetail handleGeneric(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setTitle("Internal Server Error");
        return problem;
    }
}
```

`ProblemDetail` (RFC 7807) is the standard error response format since Spring Boot 3.0.

Enable it globally:

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

---

## Testing

### Test Slices

Spring Boot provides test slices that load only the relevant part of the context:

| Annotation           | What it loads                                  | Use case                        |
|----------------------|------------------------------------------------|---------------------------------|
| `@SpringBootTest`    | Full application context                       | Integration tests               |
| `@WebMvcTest`        | Controllers, filters, converters (no service/repo beans) | Controller unit tests |
| `@DataJpaTest`       | JPA repositories, EntityManager, embedded DB   | Repository tests                |
| `@DataRedisTest`     | Redis repositories and template                | Redis tests                     |
| `@WebFluxTest`       | WebFlux controllers and filters                | Reactive controller tests       |
| `@JsonTest`          | Jackson ObjectMapper, JsonComponent            | JSON serialization tests        |
| `@RestClientTest`    | RestClient, MockRestServiceServer              | REST client tests               |

### Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderControllerIT {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        OrderRequest request = new OrderRequest("item-1", 2);

        ResponseEntity<Order> response = restTemplate.postForEntity(
                "/api/orders", request, Order.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getItemId()).isEqualTo("item-1");
    }
}
```

### Controller Unit Test with `@WebMvcTest`

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        when(orderService.findById(1L))
                .thenThrow(new EntityNotFoundException("Order not found"));

        mockMvc.perform(get("/api/orders/1"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.title").value("Resource Not Found"));
    }
}
```

### Repository Test with `@DataJpaTest`

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager em;

    @Test
    void shouldFindByStatus() {
        em.persist(new Order("item-1", OrderStatus.PENDING));
        em.persist(new Order("item-2", OrderStatus.COMPLETED));
        em.flush();

        List<Order> pending = orderRepository.findByStatus(OrderStatus.PENDING);

        assertThat(pending).hasSize(1);
        assertThat(pending.get(0).getItemId()).isEqualTo("item-1");
    }
}
```

### Testcontainers

For testing against real databases/services instead of embedded/in-memory substitutes:

```java
@SpringBootTest
@Testcontainers
class OrderServiceIT {

    @Container
    @ServiceConnection   // auto-configures spring.datasource.* from the container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderService orderService;

    @Test
    void shouldPersistOrder() {
        Order order = orderService.createOrder(new OrderRequest("item-1", 2));
        assertThat(order.getId()).isNotNull();
    }
}
```

`@ServiceConnection` (Spring Boot 3.1+) replaces manual `@DynamicPropertySource` configuration.

---

## DevTools

`spring-boot-devtools` provides development-time features:

| Feature              | What it does                                               |
|----------------------|------------------------------------------------------------|
| Automatic restart    | Restarts the app when classpath files change (fast, uses two classloaders) |
| LiveReload           | Triggers browser refresh on resource changes               |
| Property defaults    | Sets development-friendly defaults (e.g., `spring.thymeleaf.cache=false`) |
| H2 console           | Auto-enables H2 web console at `/h2-console`              |

DevTools is **automatically disabled** when running as a packaged JAR (`java -jar`) — it only activates when launched
from an IDE or with `mvn spring-boot:run`.

---

## Building and Packaging

### Fat JAR (Uber JAR)

Spring Boot packages the application + all dependencies + embedded server into a single executable JAR:

```bash
mvn clean package                  # produces target/myapp-1.0.0.jar
java -jar target/myapp-1.0.0.jar   # runs with embedded server
```

Structure of a Spring Boot JAR:

```
myapp.jar
├── BOOT-INF/
│   ├── classes/          # your compiled classes
│   └── lib/              # dependency JARs
├── META-INF/
│   └── MANIFEST.MF       # Main-Class: org.springframework.boot.loader.launch.JarLauncher
└── org/springframework/boot/loader/   # Spring Boot loader
```

### Layered JARs (Docker Optimization)

Spring Boot 2.3+ supports layered JARs for better Docker image caching:

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /app
COPY target/myapp.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Layers (from least to most frequently changing):
1. `dependencies` — third-party JARs (rarely change).
2. `spring-boot-loader` — Spring Boot loader classes.
3. `snapshot-dependencies` — snapshot JARs.
4. `application` — your code (changes most often).

### GraalVM Native Image

Spring Boot 3.x supports ahead-of-time (AOT) compilation to native executables:

```bash
mvn -Pnative native:compile   # produces a native binary
./target/myapp                  # starts in ~50ms, low memory footprint
```

Tradeoffs: faster startup and lower memory, but longer build time, no runtime reflection (requires AOT hints),
and limited library compatibility.

---

## Common Interview Questions

### How does Spring Boot auto-configuration work?

Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` for
auto-configuration classes. Each class is annotated with `@Conditional*` annotations that check the classpath,
existing beans, and properties. If conditions match, the class registers its `@Bean` definitions. User-defined beans
take precedence due to `@ConditionalOnMissingBean`.

### What is the difference between `@SpringBootApplication` and `@EnableAutoConfiguration`?

`@SpringBootApplication` is a convenience annotation that combines `@Configuration`, `@EnableAutoConfiguration`, and
`@ComponentScan`. `@EnableAutoConfiguration` is just the auto-configuration trigger — it doesn't enable component
scanning or mark the class as a configuration source.

### How do you disable a specific auto-configuration?

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

Or in properties: `spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration`.

### What is the difference between `@ConfigurationProperties` and `@Value`?

- `@ConfigurationProperties` — type-safe, structured binding of a property prefix to a POJO. Supports validation,
  relaxed binding, lists, maps, nested objects. Ideal for groups of related properties.
- `@Value` — injects a single property or SpEL expression. Simpler but no type safety, no IDE support, and
  harder to test.

### What is the difference between `@SpringBootTest` and `@WebMvcTest`?

- `@SpringBootTest` — loads the **full** application context (all beans). Use for integration tests.
- `@WebMvcTest` — loads only the **web layer** (controllers, filters, converters). Services and repositories must be
  mocked with `@MockitoBean`. Use for controller unit tests. Much faster because it doesn't load the full context.

### How does Spring Boot handle externalized configuration?

Spring Boot loads properties from multiple sources in a defined order (command line > env vars > profile-specific
files > application.properties). Properties from higher-priority sources override lower-priority ones. This allows
the same JAR to run in different environments without rebuilding.

### What is graceful shutdown and why is it important?

With `server.shutdown=graceful`, Spring Boot stops accepting new requests but allows in-flight requests to complete
within a configurable timeout. This prevents request failures during deployments in Kubernetes or other orchestrators
that send SIGTERM before routing traffic away.

### How do you create a custom starter?

1. Create an `autoconfigure` module with `@AutoConfiguration` classes and `@Conditional*` annotations.
2. Register the class in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
3. Create a `starter` module that depends on `autoconfigure` + required libraries.
4. Provide `@ConfigurationProperties` for user customization.
5. Use `@ConditionalOnMissingBean` so users can override default beans.
