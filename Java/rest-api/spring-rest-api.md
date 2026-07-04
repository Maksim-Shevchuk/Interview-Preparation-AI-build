# Spring REST API

Building REST APIs with Spring MVC — controllers, exception handling, validation, content negotiation, and
documentation. The practical implementation of [REST Principles](../../System-Design/API-Design/rest-principles.md).

## Controller Basics

### `@RestController` vs `@Controller`

```java
@RestController // = @Controller + @ResponseBody on every method
@RequestMapping("/api/users")
public class UserController {
    // return values are serialized to JSON by default
}

@Controller // for server-side rendering (Thymeleaf, etc.)
public class PageController {
    @GetMapping("/home")
    public String home(Model model) {
        return "home"; // returns view name, not JSON
    }
}
```

`@RestController` applies `@ResponseBody` to all methods — return values are serialized via `HttpMessageConverter`
(Jackson for JSON by default).

### CRUD Endpoints

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    // GET /api/users
    @GetMapping
    public List<UserDto> getAll() {
        return userService.findAll();
    }

    // GET /api/users/42
    @GetMapping("/{id}")
    public UserDto getById(@PathVariable Long id) {
        return userService.findById(id);
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // 201
    public UserDto create(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    // PUT /api/users/42
    @PutMapping("/{id}")
    public UserDto replace(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest request) {
        return userService.replace(id, request);
    }

    // PATCH /api/users/42
    @PatchMapping("/{id}")
    public UserDto update(@PathVariable Long id, @Valid @RequestBody PatchUserRequest request) {
        return userService.update(id, request);
    }

    // DELETE /api/users/42
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)  // 204
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## Request Mapping Annotations

| Annotation       | HTTP Method | Shorthand for                          |
|------------------|-------------|----------------------------------------|
| `@GetMapping`    | GET         | `@RequestMapping(method = GET)`        |
| `@PostMapping`   | POST        | `@RequestMapping(method = POST)`       |
| `@PutMapping`    | PUT         | `@RequestMapping(method = PUT)`        |
| `@PatchMapping`  | PATCH       | `@RequestMapping(method = PATCH)`      |
| `@DeleteMapping` | DELETE      | `@RequestMapping(method = DELETE)`     |

## Extracting Request Data

```java
// Path variable: /api/users/42
@GetMapping("/{id}")
public UserDto getById(@PathVariable Long id) { ... }

// Query parameters: /api/users?role=admin&status=active
@GetMapping
public List<UserDto> search(
        @RequestParam(defaultValue = "user") String role,
        @RequestParam(required = false) String status) { ... }

// Request body (JSON → object via Jackson)
@PostMapping
public UserDto create(@RequestBody CreateUserRequest request) { ... }

// Headers
@GetMapping
public String info(@RequestHeader("X-Request-Id") String requestId) { ... }

// Multiple path variables: /api/users/42/orders/7
@GetMapping("/{userId}/orders/{orderId}")
public OrderDto getOrder(@PathVariable Long userId, @PathVariable Long orderId) { ... }
```

## `ResponseEntity` — Full Control

```java
@PostMapping
public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserRequest request) {
    UserDto created = userService.create(request);

    URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();

    return ResponseEntity
            .created(location)           // 201 + Location header
            .body(created);
}

@GetMapping("/{id}")
public ResponseEntity<UserDto> getById(@PathVariable Long id) {
    return userService.findById(id)
            .map(ResponseEntity::ok)                                    // 200
            .orElseGet(() -> ResponseEntity.notFound().build());        // 404
}
```

`ResponseEntity<T>` gives control over status code, headers, and body.

## Validation

### Bean Validation (Jakarta Validation)

```java
public record CreateUserRequest(
        @NotBlank(message = "Name is required")
        @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
        String name,

        @NotBlank
        @Email(message = "Invalid email format")
        String email,

        @NotNull
        @Min(18)
        @Max(150)
        Integer age
) {}
```

Trigger validation with `@Valid` or `@Validated` on the controller parameter:

```java
@PostMapping
public UserDto create(@Valid @RequestBody CreateUserRequest request) {
    // if validation fails → 400 Bad Request (with MethodArgumentNotValidException)
}
```

### Custom Validator

```java
@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {
    String message() default "Email already in use";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email != null && !userRepository.existsByEmail(email);
    }
}
```

## Exception Handling — `@ControllerAdvice`

Centralized error handling for all controllers:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(400);
        problem.setTitle("Validation Error");
        problem.setDetail("One or more fields are invalid");

        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage()));
        problem.setProperty("errors", errors);

        return problem;
    }

    // Handle "not found"
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(404);
        problem.setTitle("Not Found");
        problem.setDetail(ex.getMessage());
        return problem;
    }

    // Handle business rule violations
    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ProblemDetail handleConflict(BusinessException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(409);
        problem.setTitle("Business Rule Violation");
        problem.setDetail(ex.getMessage());
        return problem;
    }

    // Catch-all
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ProblemDetail handleGeneral(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatus(500);
        problem.setTitle("Internal Server Error");
        problem.setDetail("An unexpected error occurred");
        // Don't expose ex.getMessage() in production — log it instead
        log.error("Unhandled exception", ex);
        return problem;
    }
}
```

### `ProblemDetail` (RFC 9457 — Spring 6+)

Spring 6+ has built-in support for RFC 9457 `application/problem+json`:

```json
{
    "type": "about:blank",
    "title": "Validation Error",
    "status": 400,
    "detail": "One or more fields are invalid",
    "instance": "/api/users",
    "errors": {
        "email": "Invalid email format",
        "name": "must not be blank"
    }
}
```

Enable globally:

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

## DTO Pattern — Don't Expose Entities

```java
// ❌ Exposing JPA entity — couples API to DB schema, leaks internal fields
@GetMapping("/{id}")
public User getById(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}

// ✅ Return a DTO — decouple API shape from persistence
@GetMapping("/{id}")
public UserDto getById(@PathVariable Long id) {
    User user = userService.findById(id);
    return UserDto.from(user);
}

public record UserDto(Long id, String name, String email) {
    public static UserDto from(User entity) {
        return new UserDto(entity.getId(), entity.getName(), entity.getEmail());
    }
}
```

**Why DTOs:**
- Control what fields are exposed (don't leak `passwordHash`, `internalStatus`).
- API shape evolves independently from DB schema.
- Different DTOs for different operations (`CreateUserRequest`, `UserResponse`, `UserListItem`).

For complex mappings: **MapStruct** (compile-time code generation, zero runtime overhead).

## Pagination in Spring

```java
@GetMapping
public Page<UserDto> getAll(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "name,asc") String[] sort) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(
            Arrays.stream(sort)
                    .map(s -> s.split(","))
                    .map(arr -> arr.length > 1 && arr[1].equalsIgnoreCase("desc")
                            ? Sort.Order.desc(arr[0])
                            : Sort.Order.asc(arr[0]))
                    .toList()
    ));

    return userService.findAll(pageable).map(UserDto::from);
}
```

Or use Spring's built-in `Pageable` resolution:

```java
@GetMapping
public Page<UserDto> getAll(Pageable pageable) {
    // Spring auto-resolves ?page=0&size=20&sort=name,asc from query params
    return userService.findAll(pageable).map(UserDto::from);
}
```

`Page<T>` response includes `content`, `totalElements`, `totalPages`, `number`, `size`.

## Content Negotiation

```java
// Produce JSON (default) or XML based on Accept header
@GetMapping(value = "/{id}", produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
public UserDto getById(@PathVariable Long id) {
    return userService.findById(id);
}

// Consume JSON only
@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
public UserDto create(@RequestBody CreateUserRequest request) { ... }
```

For XML support, add `jackson-dataformat-xml` dependency.

## CORS Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.example.com")
                .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}

// Or per-controller
@CrossOrigin(origins = "https://frontend.example.com")
@RestController
public class UserController { ... }
```

## API Documentation — OpenAPI / Swagger

```xml
<!-- springdoc-openapi -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.x</version>
</dependency>
```

```java
@Operation(summary = "Get user by ID")
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "User found"),
        @ApiResponse(responseCode = "404", description = "User not found")
})
@GetMapping("/{id}")
public UserDto getById(@PathVariable Long id) { ... }
```

Swagger UI available at `/swagger-ui.html`. OpenAPI spec at `/v3/api-docs`.

## Idempotency in Practice

```java
// POST is not idempotent — use an idempotency key to prevent duplicates
@PostMapping
public ResponseEntity<OrderDto> createOrder(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody CreateOrderRequest request) {

    // Check if this key was already processed
    return orderService.findByIdempotencyKey(idempotencyKey)
            .map(existing -> ResponseEntity.ok(existing))           // return cached result
            .orElseGet(() -> {
                OrderDto created = orderService.create(request, idempotencyKey);
                return ResponseEntity.status(201).body(created);
            });
}
```

## WebFlux (Reactive) — Brief

For non-blocking, reactive REST APIs:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserRepository userRepository; // ReactiveCrudRepository

    @GetMapping("/{id}")
    public Mono<UserDto> getById(@PathVariable Long id) {
        return userRepository.findById(id).map(UserDto::from);
    }

    @GetMapping
    public Flux<UserDto> getAll() {
        return userRepository.findAll().map(UserDto::from);
    }
}
```

**When WebFlux:** High concurrency with many I/O-bound requests (proxies, gateways, streaming). **When MVC:** Most
CRUD apps, simpler mental model, blocking I/O is fine with virtual threads (Java 21+).

## Common Interview Questions

1. **`@RestController` vs `@Controller`?** — `@RestController` adds `@ResponseBody` to every method — return values
   are serialized to JSON. `@Controller` returns view names for server-side rendering.
2. **How to handle exceptions globally?** — `@RestControllerAdvice` with `@ExceptionHandler` methods. Map
   exceptions to status codes and error response bodies (use `ProblemDetail` for RFC 9457).
3. **Why use DTOs instead of entities?** — Decouple API shape from DB schema, control exposed fields, prevent
   leaking internal data. Different DTOs for create/read/update operations.
4. **How does validation work?** — Jakarta Bean Validation annotations (`@NotBlank`, `@Email`, etc.) on DTO fields.
   Trigger with `@Valid` on the controller parameter. Validation failures throw
   `MethodArgumentNotValidException` → handle in `@ControllerAdvice`.
5. **`@PathVariable` vs `@RequestParam`?** — `@PathVariable` extracts from the URI path (`/users/{id}`).
   `@RequestParam` extracts from query string (`/users?role=admin`). Path for resource identification, query for
   filtering/pagination.
6. **How to implement pagination?** — Accept `Pageable` parameter (Spring auto-resolves from query params). Return
   `Page<T>` with content, total elements, total pages. Use `PageRequest.of(page, size, sort)`.
7. **Spring MVC vs WebFlux?** — MVC: blocking, thread-per-request, simpler. WebFlux: non-blocking, reactive
   streams, better for high-concurrency I/O-bound workloads. With Java 21 virtual threads, MVC handles high
   concurrency well too.

## Related

- [REST Principles](../../System-Design/API-Design/rest-principles.md) — theory, HTTP methods, status codes
- [@Transactional Deep Dive](../spring/data/transactional-annotation.md) — transaction management in services
- [Microservices Architecture](../spring/cloud/microservices-architecture.md) — inter-service REST communication

## Resources

- [Spring Docs — Web MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Spring Docs — Error Responses (RFC 9457)](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
- [Baeldung — Building REST APIs with Spring](https://www.baeldung.com/rest-with-spring-series)
- [springdoc-openapi](https://springdoc.org/)
