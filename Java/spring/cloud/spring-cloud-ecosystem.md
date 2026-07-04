# Spring Cloud Ecosystem

Spring Cloud provides ready-to-use implementations for common microservice patterns. This note maps each pattern to its
Spring Cloud component — the practical counterpart to [Microservices Architecture](./microservices-architecture.md).

## Ecosystem Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Spring Cloud                              │
│                                                                  │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │   Gateway     │  │ Service Discovery│  │   Config Server   │  │
│  │ (Spring Cloud │  │ (Eureka / Consul)│  │ (Spring Cloud     │  │
│  │  Gateway)     │  │                  │  │  Config)          │  │
│  └──────────────┘  └──────────────────┘  └───────────────────┘  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │  REST Client  │  │   Resilience     │  │   Observability   │  │
│  │ (OpenFeign)   │  │ (Resilience4j)   │  │ (Micrometer +     │  │
│  │              │  │                  │  │  OpenTelemetry)   │  │
│  └──────────────┘  └──────────────────┘  └───────────────────┘  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │  Messaging    │  │  Load Balancer   │  │   Security        │  │
│  │ (Spring Cloud │  │ (Spring Cloud    │  │ (Spring Security  │  │
│  │  Stream)      │  │  LoadBalancer)   │  │  + OAuth2)        │  │
│  └──────────────┘  └──────────────────┘  └───────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Pattern → Spring Component Mapping

| Pattern                  | Spring Component                      | Replaced (Legacy)          |
|--------------------------|---------------------------------------|----------------------------|
| API Gateway              | Spring Cloud Gateway                  | Zuul                       |
| Service Discovery        | Eureka / Consul / Kubernetes          | —                          |
| Client Load Balancing    | Spring Cloud LoadBalancer             | Ribbon                     |
| Declarative REST Client  | OpenFeign / Spring HTTP Interface     | —                          |
| Circuit Breaker          | Resilience4j                          | Hystrix                    |
| Config Management        | Spring Cloud Config                   | —                          |
| Distributed Tracing      | Micrometer Tracing + OpenTelemetry    | Spring Cloud Sleuth        |
| Event-Driven Messaging   | Spring Cloud Stream (Kafka/RabbitMQ)  | —                          |

## Service Discovery — Eureka

### Server

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false   # server itself doesn't register
    fetch-registry: false
```

### Client (Every Microservice)

```yaml
# application.yml
spring:
  application:
    name: order-service              # service name used for discovery
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
```

After registration, other services can call `order-service` **by name** instead of hardcoded URLs.

## API Gateway — Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service         # lb:// = load-balanced via discovery
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/orders
```

### Custom Global Filter (e.g., JWT validation)

```java
@Component
public class AuthFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // validate token, set security context...
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1; // run before other filters
    }
}
```

## Declarative REST Client — OpenFeign

```java
// Dependency: spring-cloud-starter-openfeign

@FeignClient(
    name = "user-service",                    // resolved via service discovery
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    UserDto getUser(@PathVariable Long id);

    @GetMapping("/api/users")
    List<UserDto> getUsers(@RequestParam("ids") List<Long> ids);

    @PostMapping("/api/users")
    UserDto createUser(@RequestBody CreateUserDto dto);
}
```

### Fallback (Resilience)

```java
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public UserDto getUser(Long id) {
                return UserDto.unknown(id); // graceful degradation
            }

            @Override
            public List<UserDto> getUsers(List<Long> ids) {
                return Collections.emptyList();
            }

            @Override
            public UserDto createUser(CreateUserDto dto) {
                throw new ServiceUnavailableException("User service is down", cause);
            }
        };
    }
}
```

### Spring 6 HTTP Interface (Modern Alternative)

```java
// No Feign dependency needed — built into Spring Framework 6+
@HttpExchange("/api/users")
public interface UserClient {

    @GetExchange("/{id}")
    UserDto getUser(@PathVariable Long id);

    @PostExchange
    UserDto createUser(@RequestBody CreateUserDto dto);
}

// Configuration
@Bean
public UserClient userClient(RestClient.Builder builder) {
    RestClient restClient = builder.baseUrl("http://user-service").build();
    return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(UserClient.class);
}
```

## Resilience4j

### Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        sliding-window-size: 10
        failure-rate-threshold: 50          # open after 50% failures
        wait-duration-in-open-state: 10s    # stay open for 10s
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true

  retry:
    instances:
      userService:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2   # 500ms → 1s → 2s
        retry-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException

  timelimiter:
    instances:
      userService:
        timeout-duration: 3s

  bulkhead:
    instances:
      userService:
        max-concurrent-calls: 20
```

### Usage

```java
@Service
public class OrderService {

    private final UserClient userClient;

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @Retry(name = "userService")
    @TimeLimiter(name = "userService")
    public CompletableFuture<UserDto> getUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(userId));
    }

    private CompletableFuture<UserDto> getUserFallback(Long userId, Throwable t) {
        log.warn("Fallback for user {}: {}", userId, t.getMessage());
        return CompletableFuture.completedFuture(UserDto.unknown(userId));
    }
}
```

**Decoration order (inner → outer):** Bulkhead → TimeLimiter → CircuitBreaker → Retry.
The outermost (Retry) wraps everything — if the circuit is open, retry won't help (circuit breaker fires immediately).

## Config Server — Spring Cloud Config

### Server

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# Config server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
```

### Client

```yaml
# bootstrap.yml or spring.config.import
spring:
  application:
    name: order-service
  config:
    import: configserver:http://localhost:8888

# Refresh config at runtime without restart:
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

```java
@RefreshScope  // bean is re-created when /actuator/refresh is called
@Component
public class FeatureFlags {

    @Value("${feature.new-checkout:false}")
    private boolean newCheckoutEnabled;
}
```

## Spring Cloud Stream — Event-Driven

Abstracts away the message broker (Kafka, RabbitMQ) behind a functional programming model.

### Producer

```java
@Configuration
public class OrderEventProducer {

    @Bean
    public Supplier<OrderCreatedEvent> orderCreated() {
        // Spring Cloud Stream polls this supplier and sends messages
        return () -> new OrderCreatedEvent(orderId, userId);
    }
}

// Or imperative style via StreamBridge
@Service
public class OrderService {

    private final StreamBridge streamBridge;

    public void placeOrder(Order order) {
        orderRepository.save(order);
        streamBridge.send("order-events-out-0",
                new OrderCreatedEvent(order.getId(), order.getUserId()));
    }
}
```

### Consumer

```java
@Configuration
public class PaymentEventConsumer {

    @Bean
    public Consumer<OrderCreatedEvent> processOrder() {
        return event -> {
            log.info("Processing order: {}", event.orderId());
            paymentService.initiatePayment(event);
        };
    }
}
```

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: order-events     # Kafka topic or RabbitMQ exchange
          group: payment-service        # consumer group
      kafka:
        binder:
          brokers: localhost:9092
```

## Observability — Micrometer + OpenTelemetry

### Distributed Tracing

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0            # 100% in dev, lower in prod
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

Spring Boot auto-instruments:
- RestClient / WebClient / Feign calls.
- Spring MVC / WebFlux incoming requests.
- Kafka / RabbitMQ message processing.
- JDBC queries.

Trace ID is automatically propagated across services via HTTP headers (`traceparent`).

### Metrics — Actuator + Prometheus

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

Access metrics at `/actuator/prometheus` — scraped by Prometheus, visualized in Grafana.

## Security — OAuth2 / JWT

```
Client ──▶ API Gateway ──validate JWT──▶ Auth Server (Keycloak / Auth0)
                │
                ▼ (valid)
           Microservice (receives claims from gateway)
```

### Resource Server (Every Microservice)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/actuator/**").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                .build();
    }
}
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/my-realm
```

## Typical Microservice Project Structure

```
my-microservices/
├── config-server/           # Spring Cloud Config
├── discovery-server/        # Eureka
├── api-gateway/             # Spring Cloud Gateway
├── user-service/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── order-service/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── payment-service/
├── notification-service/
├── docker-compose.yml       # local development
└── k8s/                     # Kubernetes manifests
    ├── user-service.yaml
    ├── order-service.yaml
    └── ...
```

## Common Interview Questions

1. **What Spring Cloud components have you used?** — Name the pattern and the component: Gateway for routing,
   Eureka for discovery, Resilience4j for circuit breaking, Config Server for centralized config, Micrometer for
   tracing.
2. **How does service discovery work with Eureka?** — Each service registers with the Eureka server on startup.
   Clients query Eureka to get a list of instances and use Spring Cloud LoadBalancer for client-side load balancing.
3. **What is Spring Cloud Gateway and how does it differ from Zuul?** — Gateway is the modern, non-blocking
   (Reactor/Netty-based) replacement for Zuul. Supports predicates, filters, circuit breakers, rate limiting.
4. **How do you handle inter-service communication?** — Sync: OpenFeign or Spring HTTP Interface (declarative
   clients with load balancing). Async: Spring Cloud Stream with Kafka/RabbitMQ for event-driven communication.
5. **How do you implement resilience?** — Resilience4j: Circuit Breaker (stop calling failing service), Retry
   (with exponential backoff), Timeout, Bulkhead (isolate thread pools). Fallback methods for graceful degradation.
6. **How do you manage configuration across services?** — Spring Cloud Config Server backed by Git. Services fetch
   config on startup. `@RefreshScope` + `/actuator/refresh` for runtime updates without restart.
7. **How do you trace a request across services?** — Micrometer Tracing with OpenTelemetry. A trace ID is auto-
   propagated via HTTP headers. Exported to Zipkin/Jaeger for visualization.

## Related

- [Microservices Architecture](./microservices-architecture.md) — patterns and concepts
- [@Transactional Deep Dive](../data/transactional-annotation.md) — local transactions
- [Virtual Threads](../../concurrency/virtual-threads.md) — scalability for blocking I/O in microservices

## Resources

- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Spring Cloud Gateway Docs](https://docs.spring.io/spring-cloud-gateway/reference/)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Baeldung — Spring Cloud Series](https://www.baeldung.com/spring-cloud-series)
