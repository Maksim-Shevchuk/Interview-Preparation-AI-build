# Java 17 LTS (September 2021)

The "modern Java" baseline. Spring Boot 3 requires Java 17+. Most projects that were on Java 8/11 migrated here.
Records, sealed classes, and pattern matching for `instanceof` make Java genuinely data-oriented.

## Theme

Java 17 brings **data-oriented programming**: records for data carriers, sealed types for closed hierarchies, pattern
matching for `instanceof`. Text blocks reduce string-literal noise. Strong encapsulation of JDK internals by default.

## Key Features

### Records (JEP 395, finalized in 16; stable in 17)

Transparent data carriers. The compiler generates the constructor, field accessors, `equals`, `hashCode`, `toString`.

```java
public record User(Long id, String email, Role role) {
}
```

Equivalent to a class with:

- `private final` fields
- Canonical constructor (all fields)
- Accessor methods: `id()`, `email()`, `role()` (NOT `getId()`)
- `equals` / `hashCode` based on all fields
- `toString` like `User[id=1, email=a@b.com, role=ADMIN]`

**Compact constructors** for validation:

```java
public record Email(String value) {
    public Email {
        if (!value.contains("@")) throw new IllegalArgumentException();
    }
}
```

**Records are immutable and final.** Cannot extend other classes; can implement interfaces.

### Sealed Classes (JEP 409)

Closed hierarchies — the author explicitly lists permitted subtypes:

```java
public sealed interface Shape
        permits Circle, Square, Triangle {
}

public record Circle(double radius) implements Shape {
}

public record Square(double side) implements Shape {
}

public final class Triangle implements Shape { ...
}
```

Subtypes must be `final`, `sealed`, or `non-sealed`. Enables exhaustiveness checking in switch (Java 21+).

### Pattern Matching for `instanceof` (JEP 394)

No more cast after `instanceof`:

```java
// Old
if(obj instanceof String){
String s = (String) obj;
    System.out.

println(s.length());
        }

// New
        if(obj instanceof
String s){
        System.out.

println(s.length());
        }
```

The binding `s` is in scope where the pattern definitely matches (flow-sensitive).

### Switch Expressions (JEP 361, from Java 14)

Switch as an expression with `->` and no fall-through:

```java
DayType type = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> DayType.BORING;
    case TUESDAY -> DayType.OK;
    case THURSDAY, SATURDAY -> DayType.GOOD;
    case WEDNESDAY -> DayType.HUMP;
};
```

`yield` for blocks:

```java
case WEDNESDAY ->{

log("hump day");

yield DayType.HUMP;
}
```

### Text Blocks (JEP 378, from Java 15)

Multi-line string literals with `"""`:

```java
String json = """
        {
          "id": 1,
          "name": "Alice"
        }
        """;
```

- Minimal incidental whitespace (determined by the closing `"""`).
- Escape sequences still work (`\n`, `\"`).
- `\s` for explicit trailing space, `\` at line end to suppress newline.

### Strong Encapsulation by Default (JEP 396, from Java 16)

`--illegal-access=deny` is the default. Code using `sun.misc.Unsafe` or reflection on JDK internals fails unless
explicitly opened via `--add-opens`. Library authors must migrate away from internal APIs.

### macOS AArch64 Port (JEP 389)

Native Apple Silicon support. Big performance win for Mac dev machines.

## Minor but Useful

- **`Stream.toList()`** (Java 16) — `stream.collect(Collectors.toList())` → `stream.toList()`. Returns **unmodifiable**
  list (subtle change from `Collectors.toList()`).
- **`Stream.mapMulti`** (Java 16) — replacement for `flatMap` where a lambda is overkill.
- **`HttpClient`** supports HTTP/2 over TCP and WebSockets (carried forward from 11).
- **`Optional.isEmpty()`** (Java 11, stable in 17).

## What Was Removed / Deprecated

- **Security Manager** deprecated for removal (JEP 411). Code that calls `System.setSecurityManager` warns; removal
  scheduled.
- **RMI Activation** removed.
- **Applet API** deprecated for removal.
- **Nested-based access control** carried forward (Java 11).
- **Foreign Function & Memory API** incubating (not yet stable).

## Migration Notes

- **Spring Boot 3 requires Java 17+.** Plan the upgrade if on Boot 2.x.
- **Records replace Lombok `@Value`** for simple data carriers — use records for new code.
- **Sealed interfaces** shine with pattern matching in Java 21+ — design closed hierarchies now.
- **Reflection on JDK internals** breaks. Use `--add-opens` only as a temporary bridge; migrate to public APIs.

## Common Interview Questions

1. **What is a record?**
   → An immutable data carrier. Compiler generates canonical constructor, accessors (`field()` not `getField()`),
   `equals`, `hashCode`, `toString`. Records are final and cannot extend other classes.

2. **What are sealed classes?**
   → Closed hierarchies where the author explicitly lists permitted subtypes via `permits`. Subtypes must be `final`,
   `sealed`, or `non-sealed`. Enables exhaustiveness checking in pattern matching.

3. **What is pattern matching for `instanceof`?**
   → Combines the type check and binding: `if (obj instanceof String s)`. The bound variable `s` is in scope only where
   the match is flow-sensitive-guaranteed.

4. **What are switch expressions?**
   → Switch as an expression returning a value, with `->` syntax and no fall-through. Use `yield` for block cases.
   Exhaustiveness enforced for `enum` and sealed types.

5. **What are text blocks?**
   → Multi-line string literals with `"""`. Incidental whitespace stripped based on the closing delimiter. Supports
   `\s` (trailing space) and `\` (line continuation).

6. **What is strong encapsulation by default?**
   → `--illegal-access=deny` is the default since Java 17. Reflective access to JDK internals fails unless `--add-opens`
   is specified. Forces migration to public APIs.

7. **What's the difference between `Stream.toList()` (Java 16+) and `Collectors.toList()`?**
   → `Stream.toList()` is concise and returns an **unmodifiable** list. `Collectors.toList()` returns a mutable list (
   mutable, but unspecified type).

8. **What was deprecated in Java 17?**
   → Security Manager (for removal), RMI Activation (removed), Applet API (for removal).

## Related

- `Java/modern-java/java-11-lts.md` — previous LTS
- `Java/modern-java/java-21-lts.md` — next LTS, pattern matching for switch
- `Java/core/records-and-sealed.md` — deep dive (planned)
- `Engineering-Practices/OOP/` — data-oriented programming vs classic OOP

## Resources

- **JEP index:** https://openjdk.org/jeps/0 — Java 12-17 JEPs
- **Oracle Java 17 docs:** https://docs.oracle.com/en/java/javase/17/
- **Spring Boot 3 migration:** https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide
