# Java 11 LTS (September 2018)

First LTS under the new 6-month release cadence. Stabilized Java 9-10 features (module system, `var`) and added
developer ergonomics: HTTP Client, String convenience methods.

## Theme

Java 11 is the first LTS where the **module system (JPMS)** from Java 9 is production-ready. `var` for local type
inference and the standardized HTTP Client are the headline developer features. Many APIs were pruned (Java EE, JavaFX).

## Key Features

### `var` — Local Variable Type Inference (JEP 286, from Java 10)

Compiler infers the type from the initializer. Local variables only (not fields, params, or return types).

```java
var users = new ArrayList<User>();   // ArrayList<User>
var map = new HashMap<String, Order>();
var path = Path.of("/tmp");           // Path
```

**Where `var` works:**

- Local variables with initializers
- `for`-each loop variables
- `try`-with-resources variables

**Where `var` does NOT work:**

- Fields
- Method parameters
- Return types
- Uninitialized variables (`var x;`)
- Null initializer (`var x = null;` — no type to infer)

**When to use `var`:** when the type is obvious from the right-hand side (`new ...`, factory methods, casts). Avoid when
the type isn't clear — readability over brevity.

### HTTP Client (JEP 321)

Standardized `java.net.http.HttpClient` (was incubating in Java 9-10). Supports HTTP/1.1, HTTP/2, WebSocket. Sync and
async.

```java
HttpClient client = HttpClient.newBuilder()
        .version(Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(10))
        .build();

HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Accept", "application/json")
        .GET()
        .build();

// Sync
HttpResponse<String> resp = client.send(req, BodyHandlers.ofString());
System.out.

println(resp.statusCode());
        System.out.

println(resp.body());

// Async
        client.

sendAsync(req, BodyHandlers.ofString())
        .

thenApply(HttpResponse::body)
        .

thenAccept(System.out::println);
```

Replaces Apache HttpClient / OkHttp for many simple use cases. No third-party dependency.

### String Methods

```java
"  ".isBlank();          // true — only whitespace
"a\nb\nc".

lines();       // Stream<String> of lines
"  hi  ".

strip();        // "hi" — Unicode-aware trim
"ab".

repeat(3);          // "ababab"
"  hi  ".

stripLeading(); .

stripTrailing();
```

`strip()` vs `trim()`: `strip` uses `Character.isWhitespace` (Unicode-aware); `trim` uses ASCII whitespace only.

### Files API

```java
String content = Files.readString(Path.of("file.txt"));
Files.

writeString(Path.of("out.txt"),content);
```

Convenience for reading/writing small text files. Replaces `Files.readAllBytes` + `new String` boilerplate.

### Collection.toArray(IntFunction) (JEP ??)

```java
String[] arr = list.toArray(String[]::new);
```

Concise typed array conversion. Older form: `list.toArray(new String[0])`.

### Nest-Based Access Control (JEP 181)

Nested types (top-level class and its nested classes) can access each other's private members without compiler-generated
bridge methods. Cleaner bytecode, slight performance improvement. Mostly invisible to developers.

### `Optional` Methods (Java 9-11)

Added in Java 9-11:

- `Optional.isEmpty()` (Java 11) — opposite of `isPresent()`
- `Optional.ifPresentOrElse(action, emptyAction)` (Java 9)
- `Optional.or(supplier)` (Java 9)
- `Optional.stream()` (Java 9)

### GC and Tooling

- **Epsilon GC** (JEP 318) — no-op garbage collector for performance testing (allocates, never collects).
- **ZGC** experimental (Java 11) — sub-10ms pauses on large heaps. Production-ready in Java 15.
- **Java Flight Recorder (JFR)** open-sourced (was Oracle commercial). Use `jcmd <pid> JFR.start ...`.
- **Java Mission Control (JMC)** decoupled from JDK.

### `Files.list` and `BufferedReader.lines` (Java 8, stable)

Already in Java 8 but heavily used from 11 onward — return `Stream<String>`, must be closed in try-with-resources.

## What Was Removed

- **JavaFX** decoupled from JDK (now a separate project).
- **Java EE modules** removed from JDK: `javax.xml.bind` (JAXB), `javax.xml.ws` (JWS), `javax.activation`,
  `javax.transaction`, `javax.annotation` (some). They moved to Jakarta EE — must add as Maven/Gradle dependencies.
- **CORBA** modules deprecated (removed in Java 11).
- `sun.misc.Unsafe` paths to internal APIs gated behind `--add-exports`.

## Migration Notes

- **JAXB / JWS on Java 11:** add dependencies:
  ```xml
  <dependency>
      <groupId>jakarta.xml.bind</groupId>
      <artifactId>jakarta.xml.bind-api</artifactId>
  </dependency>
  <dependency>
      <groupId>org.glassfish.jaxb</groupId>
      <artifactId>jaxb-runtime</artifactId>
  </dependency>
  ```
- **Lombok / byte-code manipulators:** may need updates for Java 11+ bytecode level.
- **`var` adoption:** start with local variables where the type is obvious; don't use in fields or signatures.

## Common Interview Questions

1. **What's new in Java 11?**
   → `var` (from 10), HTTP Client, String methods (`isBlank`, `strip`, `lines`, `repeat`),
   `Files.readString/writeString`, Nest-Based Access, Epsilon GC, ZGC experimental, JFR open-sourced.

2. **Where can `var` be used?**
   → Local variables with initializers, `for`-each, `try`-with-resources. NOT in fields, parameters, return types, or
   uninitialized.

3. **Difference between `trim()` and `strip()`?**
   → `trim` removes ASCII whitespace (≤ U+0020). `strip` uses `Character.isWhitespace` — Unicode-aware, removes all
   whitespace code points.

4. **What is the HTTP Client and what protocols does it support?**
   → `java.net.http.HttpClient` — standardized in Java 11. Supports HTTP/1.1 and HTTP/2 (with server push), WebSocket.
   Both sync and async.

5. **What was removed from the JDK in Java 11?**
   → Java EE modules (JAXB, JWS, etc.) — now Jakarta EE. JavaFX decoupled. CORBA deprecated.

6. **Why was Java Flight Recorder open-sourced?**
   → Was a commercial Oracle feature. Open-sourced to make low-overhead profiling universally available. Use via
   `jcmd <pid> JFR.start`.

7. **What is Epsilon GC?**
   → A no-op garbage collector that allocates memory but never reclaims. Used for performance testing — measuring
   allocation throughput, memory-bound workload characterization.

## Related

- `Java/modern-java/java-8-lts.md` — previous LTS
- `Java/modern-java/java-17-lts.md` — next LTS
- `Java/concurrency/completable-future.md` — pairs with the HTTP Client async API

## Resources

- **JEP index:** https://openjdk.org/jeps/0 — Java 9-11 JEPs
- **Oracle Java 11 docs:** https://docs.oracle.com/en/java/javase/11/
- **Java 11 migration guide:** https://docs.oracle.com/en/java/javase/11/migrate/
