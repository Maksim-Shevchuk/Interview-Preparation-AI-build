# Java 25 LTS (September 2025)

The latest LTS as of this writing. Stabilizes the Loom-related APIs and removes the biggest pain point of Java 21 —
virtual thread pinning on `synchronized`.

## Theme

Java 25 finalizes **structured concurrency** and **scoped values**, and crucially removes the pinning limitation that
forced `ReentrantLock` workarounds in Java 21. New memory model features (compact object headers, generational ZGC)
improve footprint and pause times.

## Key Features

### Synchronized Virtual Threads Without Pinning (JEP 491)

The biggest practical change. In Java 21, a virtual thread holding a `synchronized` monitor could not unmount from its
carrier while blocking — pinning the carrier. Java 25 lifts this limitation.

**Before (Java 21 workaround):**

```java
private final Lock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try { /* blocking I/O — pinning-safe */ } finally {
        lock.unlock();
    }
}
```

**After (Java 25):**

```java
public synchronized void update() {
    /* blocking I/O — virtual thread unmounts normally */
}
```

Migration: code that switched to `ReentrantLock` purely to avoid pinning can revert to `synchronized` for readability.
Don't rush — benchmark first; `ReentrantLock` is still preferable when you need `tryLock`, interruptibility, or multiple
`Condition`s.

### Scoped Values — Final (JEP 506)

Replaces `ThreadLocal` for virtual threads. Immutable, scoped to a code block, inherited by child virtual threads.

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

public void handle(User user) {
    ScopedValue.where(CURRENT_USER, user).run(() -> process());
}

void process() {
    User u = CURRENT_USER.get();
    // ...
}
```

Why not `ThreadLocal`:

- Per-thread storage — millions of virtual threads × `ThreadLocal` = memory bloat.
- Mutable — action-at-a-distance bugs.
- `ScopedValue` is bound for a scope, immutable, cheap.

### Structured Concurrency — Final (JEP 505)

`StructuredTaskScope` API finalized. Manage child virtual threads with a clear lifecycle:

```java
public record UserResponse(User user, List<Order> orders) {
}

public UserResponse handle(long id) {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<User> userT = scope.fork(() -> userDao.find(id));
        Subtask<List<Order>> ordersT = scope.fork(() -> orderDao.list(id));
        scope.join().throwIfFailed();
        return new UserResponse(userT.get(), ordersT.get());
    }
}
```

Two built-in shutdown policies:

- `ShutdownOnFailure` — if any child fails, cancel the rest. Most common.
- `ShutdownOnSuccess` — cancel the rest as soon as one succeeds (e.g., redundant lookups for latency).

Custom policies can be built by subclassing `StructuredTaskScope`.

### Stream Gatherers — Final (JEP 485)

Extends Stream API with **intermediate operations** that are stateful and can use arbitrary logic. Lets you build
operations that `Collectors` can't express mid-pipeline.

```java
Stream.of(1,2,3,4,5)
        .

gather(Gatherers.windowFixed(2))   // [[1, 2], [3, 4], [5]]
        .

toList();
```

Built-in gatherers: `windowFixed`, `windowSliding`, `fold`, `scan`, `mapConcurrent`.

### Module Import Declarations — Final (JEP 511 — varies by exact JEP number)

Import a whole module's `exports` in one line:

```java
import module java.base;
```

Replaces boilerplate of many individual `import` statements for common packages. Mainly for scripting and small
programs.

### Compact Source Files / Instance Main Methods (Final or preview — verify)

Single-file programs without `class` or `public static void main`:

```java
void main() {
    println("hello");
}
```

Aimed at beginners and scripting. The full `class` + `public static void main(String[])` form remains for real
applications.

### Class-File API — Final (JEP 484)

Standard API for reading/writing/transforming class files. Replaces ASM/Javassist dependencies for many framework use
cases (Spring, Hibernate, Mockito all had their own copies of ASM).

### Ahead-of-Time Class Loading & Linking (JEP 483)

At build time, the linker can pre-process class files so the runtime loads them faster. Improves startup of short-lived
services (CLI tools, Lambda functions).

### Compact Object Headers (JEP 473 — experimental, stabilized in 25)

Reduces the HotSpot object header from 12-16 bytes to **8 bytes** on 64-bit JVMs. Significant heap savings for apps with
many small objects.

Enabled by default in later releases; check with `java -XX:+PrintFlagsFinal | grep Header`.

### Generational ZGC (JEP 490 — final)

ZGC now has generations — young objects collected more frequently than old. Better throughput and pause distribution for
heaps where allocation rate is high.

### Security Manager Permanently Disabled (JEP 486)

The legacy Security Manager (deprecated in Java 17) is now non-functional. Code relying on `System.setSecurityManager`
no longer works. Modern alternatives: OS-level sandboxing, container isolation, signed modules.

## What Was Removed

- **32-bit x86 port** (JEP 479) — only 64-bit x86 and ARM are maintained.
- **Security Manager** (JEP 486) — non-functional; planned for full removal.
- **Some legacy JDK internal tools** superseded by `jcmd`.

## Migration Notes

- **Java 21 → Java 25:** low-risk for most apps. Re-enable `synchronized` patterns that were rewritten to
  `ReentrantLock` to avoid pinning — but benchmark first.
- **`ThreadLocal` → `ScopedValue`:** migrate per use case. Start with request-scoped values in web apps. `ThreadLocal`
  still works but is wasteful under many virtual threads.
- **`CompletableFuture` chains → structured concurrency:** not a drop-in replacement. Use structured concurrency for new
  I/O-bound code; leave existing `CompletableFuture` pipelines as-is unless you're rearchitecting.
- **Spring Boot 3.4+** supports Java 25 features (check version matrix).
- **ASM / CGLIB / ByteBuddy:** frameworks migrate to the Class-File API gradually. Third-party byte-code tools may still
  be needed for advanced cases.

## Common Interview Questions

1. **What's new in Java 25?**
   → Synchronized virtual threads without pinning (JEP 491), scoped values (final), structured concurrency (final),
   stream gatherers, compact object headers, generational ZGC, AOT class loading, Security Manager disabled.

2. **Why is Java 25's synchronized-without-pinning important?**
   → In Java 21, a virtual thread holding a `synchronized` monitor couldn't unmount while blocking — pinning the
   carrier. Java 25 removes this limitation. You can use plain `synchronized` with blocking I/O on virtual threads
   again.

3. **What are scoped values and why replace ThreadLocal?**
   → Immutable, scope-bound values inherited by child virtual threads. `ThreadLocal` has per-thread memory cost (
   problematic at millions of virtual threads) and mutable action-at-a-distance. `ScopedValue` is cheap and clear.

4. **What is structured concurrency?**
   → `StructuredTaskScope` API for spawning child virtual threads with a clear lifecycle. When the scope exits, children
   are guaranteed done or cancelled. Policies: `ShutdownOnFailure`, `ShutdownOnSuccess`.

5. **What are stream gatherers?**
   → Stateful intermediate stream operations expressible in user code. Built-ins: `windowFixed`, `windowSliding`,
   `fold`, `scan`. Fills the gap left by `Collectors` (which are terminal).

6. **What is the Class-File API?**
   → Standard JDK API for reading/writing/transforming class files. Reduces framework dependency on ASM / ByteBuddy.

7. **What happened to the Security Manager?**
   → Permanently disabled in Java 25 (deprecated in 17). Modern apps use container isolation, OS sandboxing, and signed
   modules instead.

8. **What are compact object headers?**
   → Reduces the HotSpot object header from 12-16 bytes to 8 bytes. Saves heap for apps with many small objects. Enabled
   by default in later Java 25 builds.

9. **What is AOT class loading?**
   → At build time, class files can be pre-processed by `jlink` / a linker for faster runtime loading. Improves startup
   for short-lived services.

## Related

- `Java/modern-java/java-21-lts.md` — previous LTS
- `Java/concurrency/virtual-threads.md` — pinning, structured concurrency deep dive
- `Java/concurrency/synchronized-and-volatile.md` — `synchronized` semantics
- `Java/core/stream-api.md` — gatherers extend the Stream API

## Resources

- **JEP index:** https://openjdk.org/jeps/0 — Java 22-25 JEPs
- **Oracle Java 25 docs:** https://docs.oracle.com/en/java/javase/25/
- **Inside Java podcasts and talks on Project Loom finalization**
