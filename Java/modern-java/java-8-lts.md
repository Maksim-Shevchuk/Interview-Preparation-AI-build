# Java 8 LTS (March 2014)

The biggest Java release of its era. Brought functional programming, modern collections, and a new date/time API. The
foundation for "modern Java" — every later LTS builds on it.

## Theme

Java 8 introduced lambdas, method references, and Stream API, fundamentally changing how Java code is written.
`Optional` and `java.time` addressed two long-standing pain points (nulls and `java.util.Date`).

## Key Features

### Lambdas (JEP 126)

Syntax: `(params) -> expression` or `(params) -> { statements; }`.

```java
Runnable r = () -> System.out.println("hi");
Comparator<String> c = (a, b) -> Integer.compare(a.length(), b.length());
```

The type of a lambda is a **functional interface** — an interface with exactly one abstract method.
`@FunctionalInterface` is optional but documents intent.

Built-in functional interfaces in `java.util.function`:

- `Function<T, R>` — `R apply(T)`
- `Predicate<T>` — `boolean test(T)`
- `Consumer<T>` — `void accept(T)`
- `Supplier<T>` — `T get()`
- `BiFunction<T, U, R>`, `BiPredicate`, `BiConsumer`
- Primitive variants: `IntFunction`, `ToIntFunction`, `IntPredicate`, etc.

### Method References (JEP 126)

Shorthand for lambdas that just call an existing method:

```java
// Static
Function<String, Integer> parser = Integer::parseInt;
// Instance (of a particular object)
Supplier<Integer> size = myList::size;
// Instance (of the argument)
Function<String, String> upper = String::toUpperCase;
// Constructor
Supplier<List<String>> factory = ArrayList::new;
```

### Stream API (JEP 107)

Declarative processing of collections. See `Java/core/stream-api.md` for the full deep dive.

```java
List<String> top = orders.stream()
        .filter(o -> o.status() == SHIPPED)
        .map(Order::customer)
        .distinct()
        .sorted()
        .toList();
```

### Optional (JEP ??)

Container for a possibly-absent value. Designed as a return type, not a field type.

```java
Optional<User> user = repo.findById(id);
user.

map(User::name).

orElse("anonymous");
user.

ifPresent(this::sendEmail);
```

**Anti-patterns:** `Optional.get()` without `isPresent()` (throws `NoSuchElementException`); using `Optional` as a field
or parameter (serialization and API design issues). See `Java/core/optional.md` (planned).

### Default Methods (JEP 126)

Interfaces can have method bodies via `default`:

```java
interface Vehicle {
    default void start() {
        System.out.println("starting");
    }
}
```

Enables interface evolution (adding methods to `Collection` for Stream API without breaking implementations). Resolution
of diamond conflicts: the implementing class must override, or the compiler errors with "ambiguous".

### Static Methods on Interfaces

```java
interface Comparator<T> {
    static <T> Comparator<T> reverseOrder() { ...}
}
```

### java.time (JEP 150 — ThreeTen)

Replaces `java.util.Date` and `Calendar`. Immutable, thread-safe, well-modeled:

- `LocalDate`, `LocalTime`, `LocalDateTime` — no timezone
- `ZonedDateTime` — with timezone
- `Instant` — UTC timestamp (machine time)
- `Duration` — time-based (hours, minutes, seconds)
- `Period` — date-based (years, months, days)
- `DateTimeFormatter` — thread-safe replacement for `SimpleDateFormat`

```java
LocalDate today = LocalDate.now();
LocalDate nextWeek = today.plusWeeks(1);
Period age = Period.between(birthday, today);
```

### CompletableFuture (JEP ??)

Async composition. See `Java/concurrency/completable-future.md`.

### Type Annotations (JEP 104)

Annotations can be applied to any type use:

```java
List<@NonNull String> names;
(@NonNull String)obj;
```

Useful for static analysis (Checker Framework, IntelliJ inspections).

### Nashorn

Replaced Rhino as the JavaScript engine. Removed in Java 15. Mostly historical now.

## What Was Removed / Changed

- **Permanent Generation (PermGen)** → replaced by **Metaspace** (native memory, grows dynamically). Fixes the classic
  `OOM: PermGen space`.
- `java.util.Date` and `Calendar` left in place but deprecated for new code.

## Migration Notes

- Code written for Java 7 streams in loops can be incrementally refactored to Stream API.
- `Optional` adoption: start by using it for return types of repository methods; don't retrofit into every field.
- `java.util.Date` → `java.time` migration: use `Date.toInstant()` to bridge.

## Common Interview Questions

1. **What are the major features of Java 8?**
   → Lambdas, method references, Stream API, `Optional`, default/static interface methods, `java.time`,
   `CompletableFuture`, type annotations.

2. **What is a functional interface?**
   → An interface with exactly one abstract method. `@FunctionalInterface` annotation enforces it at compile time.
   Lambdas are typed as functional interfaces.

3. **What's the difference between `Stream.of` and `Collection.stream`?**
   → `Stream.of` creates a stream from explicit values; `Collection.stream` creates one from a collection's elements.

4. **Why was `Optional` introduced?**
   → To make absence explicit in the type system and avoid `NullPointerException` from unguarded null returns. Designed
   as a return type, not a field.

5. **What are default methods and why were they added?**
   → Interface methods with a body. Added to enable interface evolution — `Collection` could gain `stream()` without
   breaking existing implementations.

6. **What replaces `java.util.Date`?**
   → `java.time` package: `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `Instant`. All immutable and thread-safe.

7. **What is PermGen and why was it removed?**
   → PermGen was a fixed-size heap region for class metadata. Replaced by Metaspace (native memory) in Java 8,
   eliminating `OOM: PermGen space`.

## Related

- `Java/core/stream-api.md` — full Stream API deep dive
- `Java/core/lambda-expressions.md` — lambdas and method references (planned)
- `Java/core/optional.md` — `Optional` patterns (planned)
- `Java/modern-java/java-11-lts.md` — next LTS
- `Java/concurrency/completable-future.md` — introduced in Java 8

## Resources

- **JEP index:** https://openjdk.org/jeps/0 — filter by Java 8
- **"Modern Java in Action" (Urma, Fusco, Mycroft)** — definitive Java 8 reference
- **Oracle Java 8 docs:** https://docs.oracle.com/javase/8/docs/
