# Virtual Threads (Java 21+)

Project Loom's flagship feature — lightweight threads scheduled by the JVM, not the OS. Asked at Senior interviews;
useful to know for Middle to show awareness of modern Java.

## What Virtual Threads Are

A virtual thread is a `Thread` instance that runs on a **carrier thread** (platform thread) from a dedicated
`ForkJoinPool`. The JVM schedules virtual threads onto carriers and unmounts them when they block on I/O or parking.

```
┌──────────────────────────────────────────────┐
│ Carrier threads (platform, ~nCpu-1)          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ carrier │  │ carrier │  │ carrier │       │
│  └─────────┘  └─────────┘  └─────────┘       │
│       │            │            │            │
│      runs         runs         runs          │
│       ▼            ▼            ▼            │
│  ┌────────┐   ┌────────┐   ┌────────┐        │
│  │virtual │   │virtual │   │virtual │        │
│  │thread 1│   │thread 2│   │thread 3│        │
│  └────────┘   └────────┘   └────────┘        │
│       │                                      │
│      blocks on socket.read()                 │
│       ▼                                      │
│  unmount — carrier picks another virtual     │
└──────────────────────────────────────────────┘
```

You can have **millions** of virtual threads — each is cheap (~few KB vs ~1MB platform thread).

## Platform vs Virtual Threads

| Property      | Platform Thread              | Virtual Thread          |
|---------------|------------------------------|-------------------------|
| Scheduling    | OS                           | JVM (on carrier pool)   |
| Memory        | ~1MB stack                   | ~few KB (growable)      |
| Blocking cost | Carries whole thread         | Unmounts, carrier freed |
| Count         | Hundreds                     | Millions                |
| Creation      | `new Thread()` / `Executors` | `Thread.ofVirtual()`    |
| Daemon        | configurable                 | always daemon           |
| Priority      | configurable                 | fixed (no priority)     |
| Stack size    | configurable                 | not exposed             |

## Creating Virtual Threads

```java
// Direct
Thread vt = Thread.ofVirtual().name("vt-1").start(() -> {
            // task — blocking is fine here
        });

// Single-started virtual thread
Thread vt = Thread.startVirtualThread(() -> {
    handleRequest(socket);
});

// Per-task executor (recommended pattern)
try(
ExecutorService es = Executors.newVirtualThreadPerTaskExecutor()){
        for(
Socket client :clients){
        es.

submit(() ->

handleRequest(client));
        }
        }  // close() waits for all tasks to finish
```

## Why Virtual Threads Help

For I/O-bound workloads (HTTP servers, DB calls, file I/O), you write **straight-line blocking code** at scale:

```java
public UserResponse handle(long id) throws IOException, SQLException {
    User user = userDao.find(id);            // blocks — fine on a virtual thread
    List<Order> orders = orderDao.list(id);  // blocks — fine
    Pdf pdf = pdfRenderer.render(user);      // blocks — fine
    return new UserResponse(user, orders, pdf);
}
```

No `CompletableFuture`, no reactive chains, no callback hell. The virtual thread is unmounted automatically during each
blocking call, so the carrier thread is free to run other virtual threads.

## Pinning — When Virtual Threads Can't Unmount

A virtual thread is **pinned** when it blocks but cannot unmount from the carrier:

1. **`synchronized` block holding a monitor** — JVM cannot move the virtual thread off the carrier while it owns the
   monitor. Other virtual threads scheduled on the same carrier wait.
    - **Fix (Java 24+, JEP 491):** pinning during `synchronized` is removed in newer Java versions. For Java 21, replace
      `synchronized` with `ReentrantLock` in hot paths.
2. **JNI / native frames** — the stack includes native code that the JVM can't relocate.
3. **`Object.wait()`** — historically pinned; addressed in later Java versions.

**Detecting pinning:**

- JFR event `jdk.VirtualThreadPinned`.
- `jcmd <pid> Thread.dump_to_file /tmp/vthreads.txt`.
- `-Djdk.tracePinnedThreads=full` or `-Djdk.tracePinnedThreads=short`.

## Structured Concurrency

`java.util.concurrent.StructuredTaskScope` (preview in Java 21, stabilized in Java 24+) — manages child virtual threads
with a clear lifecycle:

```java
public record UserResponse(User user, List<Order> orders, Pdf pdf) {
}

public UserResponse handle(long id) {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<User> userT = scope.fork(() -> userDao.find(id));
        Subtask<List<Order>> ordersT = scope.fork(() -> orderDao.list(id));
        Subtask<Pdf> pdfT = scope.fork(() -> pdfRenderer.render(id));

        scope.join();              // wait for all
        scope.throwIfFailed();     // propagate exception

        return new UserResponse(userT.get(), ordersT.get(), pdfT.get());
    }
}
```

Two shutdown policies:

- `ShutdownOnFailure` — if any child fails, cancel the rest.
- `ShutdownOnSuccess` — cancel the rest as soon as one succeeds (e.g., for redundant requests).

## Scoped Values (Replacing ThreadLocal)

`ScopedValue` (preview in Java 21, stabilized later) — immutable values scoped to a code block, automatically inherited
by child virtual threads:

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

public void handle(User user) {
    ScopedValue.where(CURRENT_USER, user).run(() -> {
        // CURRENT_USER.get() works here, and in any virtual thread spawned here
        processOrder();
    });
}
```

Why not `ThreadLocal`:

- `ThreadLocal` on millions of virtual threads is wasteful (memory per thread).
- `ThreadLocal` is mutable, leading to action-at-a-distance.
- `ScopedValue` is bound for a scope, immutable, and cheap.

## When to Use Virtual Threads

**Use them when:**

- I/O-bound workloads (HTTP, DB, file, network).
- You want simple blocking code at concurrency levels where thread-per-request would otherwise fail.
- Replacing `CompletableFuture` chains or reactive code where clarity matters more than sub-microsecond dispatch.

**Don't use them when:**

- CPU-bound work — virtual threads don't speed up computation. Use `parallelStream` or explicit CPU-bound pool.
- You need fine-grained control over the carrier pool size (virtual threads use the shared `ForkJoinPool`).

## Migration: Reactive → Virtual Threads

Many Spring WebFlux apps were built for throughput under I/O-heavy loads. With virtual threads, the same throughput is
achievable with **plain `@Transactional` blocking services** — at the cost of slightly higher memory per request.

Spring Boot 3.2+ supports `spring.threads.virtual.enabled=true` — Tomcat, `@Async`, schedulers, etc. switch to virtual
threads automatically.

## Common Pitfalls

1. **`synchronized` + I/O inside** — pins the carrier (Java 21). Replace with `ReentrantLock`.
2. **`ThreadLocal` per virtual thread** — millions of virtual threads × `ThreadLocal` = memory bloat. Use `ScopedValue`
   where possible.
3. **Pool of virtual threads** — anti-pattern; virtual threads are cheap, no need to pool. Use
   `newVirtualThreadPerTaskExecutor`.
4. **Assuming faster execution** — virtual threads improve throughput under I/O, not latency of a single task.
5. **Mixing `parallelStream` and virtual threads** — `parallelStream` uses the common pool too, but for CPU work — can
   interfere.
6. **Pinning in libraries** — third-party code using `synchronized` + I/O can pin. Profile with JFR before adopting at
   scale.

## Code Examples

### HTTP server with one virtual thread per request

```java
try(ServerSocket server = new ServerSocket(8080);
ExecutorService es = Executors.newVirtualThreadPerTaskExecutor()){
        while(!server.

isClosed()){
Socket client = server.accept();
        es.

submit(() ->

handleRequest(client));
        }
        }
```

### Spring Boot configuration

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

After this, Tomcat processes requests on virtual threads; `@Async` methods run on virtual threads; `@Scheduled` tasks
run on virtual threads.

### Structured concurrency shutdown-on-failure

```java
try(var scope = new StructuredTaskScope.ShutdownOnFailure()){
Subtask<User> userT = scope.fork(() -> userDao.find(id));
Subtask<List<Order>> ordersT = scope.fork(() -> orderDao.list(id));
    scope.

join().

throwIfFailed();
    return new

Response(userT.get(),ordersT.

get());
        }
```

### Replacing synchronized with ReentrantLock

```java
// Before — pins virtual thread on the carrier
public synchronized void update() { /* I/O inside */ }

// After — does not pin
private final Lock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try { /* I/O inside */ } finally {
        lock.unlock();
    }
}
```

## Common Interview Questions

1. **What are virtual threads?**
   → Lightweight threads scheduled by the JVM onto carrier (platform) threads. Each is cheap (~few KB), allowing
   millions of concurrent virtual threads.

2. **How do virtual threads differ from platform threads?**
   → Platform: OS-scheduled, ~1MB stack, count in hundreds. Virtual: JVM-scheduled, growable stack, count in millions,
   automatically unmount on blocking calls.

3. **What is pinning?**
   → A virtual thread that can't unmount from its carrier because it holds a `synchronized` monitor or is in a native
   frame. Other virtual threads scheduled on the same carrier wait.

4. **How do you detect pinning?**
   → JFR event `jdk.VirtualThreadPinned`, `jcmd Thread.dump_to_file`, or `-Djdk.tracePinnedThreads=full`.

5. **How do you fix pinning in Java 21?**
   → Replace `synchronized` blocks containing blocking operations with `ReentrantLock`. (Java 24+ removes the limitation
   entirely.)

6. **Should you pool virtual threads?**
   → No — they're cheap to create and destroy. Use `Executors.newVirtualThreadPerTaskExecutor()` instead of a fixed
   pool.

7. **What is structured concurrency?**
   → `StructuredTaskScope` API that runs child virtual threads with a clear lifecycle — when the scope ends, all
   children are done or cancelled. Shutdown policies: `ShutdownOnFailure`, `ShutdownOnSuccess`.

8. **What is `ScopedValue` and why prefer it over `ThreadLocal`?**
   → Immutable values scoped to a code block, inherited by child virtual threads. Avoids `ThreadLocal`'s per-thread
   memory cost (critical at millions of virtual threads) and mutable action-at-a-distance.

9. **When should you NOT use virtual threads?**
   → CPU-bound work (no I/O to unmount for) — use `parallelStream` or an explicit CPU pool. Also when you need
   fine-grained control over the carrier pool size.

10. **How do you enable virtual threads in Spring Boot?**
    → `spring.threads.virtual.enabled=true` (Spring Boot 3.2+). Tomcat, `@Async`, and `@Scheduled` automatically use
    virtual threads.

## Related

- `Java/concurrency/executors-and-thread-pools.md` — platform thread pools, ForkJoinPool
- `Java/concurrency/synchronized-and-volatile.md` — pinning, `synchronized` vs `ReentrantLock`
- `Java/concurrency/completable-future.md` — async alternative for I/O
- `Java/jvm-internals/memory-model-and-gc.md` — JVM scheduling

## Resources

- **JEP 444 (Virtual Threads):** https://openjdk.org/jeps/444
- **JEP 453 (Structured Concurrency):** https://openjdk.org/jeps/453
- **JEP 446 (Scoped Values):** https://openjdk.org/jeps/446
- **Ron Pressler, "Project Loom" talks** — Project Loom lead
- **Spring docs:** https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.threads.virtual
