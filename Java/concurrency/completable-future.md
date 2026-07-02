# CompletableFuture

Java 8's composable async abstraction. Asked at Middle+ interviews — covers composition, error handling, executor choice, and the common ForkJoinPool pitfall.

## What It Is

`CompletableFuture<T>` is a `Future` that can also be **completed manually** and **chained** with dependent stages. Unlike `Future`, you don't poll with `get()` — you register callbacks.

```java
CompletableFuture<Order> future = CompletableFuture
        .supplyAsync(() -> fetchOrder(id))
        .thenApply(Order::enrich)
        .thenCompose(o -> saveAsync(o));
```

## Creating a CompletableFuture

```java
// Async supplier (default common pool, OR pass executor)
CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture.supplyAsync(() -> fetchUser(id), ioExecutor);

// Async runnable (returns CompletableFuture<Void>)
CompletableFuture.runAsync(() -> sendEmail());

// Already-completed
CompletableFuture.completedFuture(value);
CompletableFuture.failedFuture(new IOException("oops"));   // Java 9+

// Manual completion
var f = new CompletableFuture<String>();
f.complete("done");    // or f.completeExceptionally(ex)
```

## Composition Operators

| Operator | Input | Output | Equivalent |
|----------|-------|--------|------------|
| `thenApply(fn)` | `T` | `R` | `map` |
| `thenCompose(fn)` | `T` | `CompletableFuture<R>` | `flatMap` |
| `thenCombine(other, fn)` | `(T, U)` | `R` | `zip` |
| `thenAccept(fn)` | `T` | `void` | side-effect |
| `thenRun(fn)` | `void` | `void` | side-effect (no input) |
| `handle((t, ex) -> ...)` | `T or ex` | `R` | recover + transform |
| `whenComplete((t, ex) -> ...)` | `T or ex` | same | side-effect, doesn't transform |
| `exceptionally(ex -> ...)` | `ex` | `T` | recover |

### thenApply vs thenCompose

`thenApply` is `map` — the function returns a plain value:
```java
CompletableFuture<Integer> length = future.thenApply(String::length);
```

`thenCompose` is `flatMap` — the function returns a `CompletableFuture`:
```java
CompletableFuture<User> user = future.thenCompose(id -> fetchUserAsync(id));
```

Using `thenApply` with a function that returns a `CompletableFuture` produces `CompletableFuture<CompletableFuture<User>>` — the nested-monad trap.

### thenCombine — Parallel Zip

```java
CompletableFuture<User> userF = fetchUserAsync(id);
CompletableFuture<List<Order>> ordersF = fetchOrdersAsync(id);

CompletableFuture<UserProfile> profile = userF.thenCombine(ordersF, UserProfile::new);
```

Both run in parallel; `profile` completes when both complete.

## Async Variants

Every operator has an `...Async` variant:
- `thenApply(fn)` — runs on the thread that completed the previous stage (or the caller if not yet complete).
- `thenApplyAsync(fn)` — runs on the common ForkJoinPool.
- `thenApplyAsync(fn, executor)` — runs on the given executor.

**Recommendation:** use `Async` variants with an explicit executor for predictable behavior, especially for I/O.

## Combination Operators

```java
// All — wait for all
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
List<User> users = all.thenApply(_ -> List.of(f1.join(), f2.join(), f3.join())).join();

// Any — first to complete
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2, f3);
Object first = any.join();
```

Java 9+ added type-safe alternatives; `anyOf` itself remains untyped — for type safety use a library or manual handling.

## Error Handling

```java
CompletableFuture<User> f = fetchUserAsync(id)
        .exceptionally(ex -> {
            log.error("failed", ex);
            return fallbackUser();
        })
        .handle((user, ex) -> ex != null ? fallbackUser() : user);
```

`handle` covers both success and failure. `exceptionally` only fires on failure. `whenComplete` is side-effect-only (cannot transform the result).

**Important:** without explicit error handling, exceptions are swallowed into the future and only surface on `.join()` / `.get()`. A forgotten `CompletableFuture` (no one calls `join`) silently drops errors.

## Cancellation

```java
future.cancel(true);   // mayInterruptIfRunning flag
```

Cancellation completes the future exceptionally with `CancellationException`. Downstream stages see it. Unlike `Future.cancel`, the underlying task is NOT actually interrupted unless the task author checks the interrupt flag.

## Java 9+ Additions

- `orTimeout(timeout, unit)` — fail with `TimeoutException` after timeout.
- `completeOnTimeout(value, timeout, unit)` — complete with `value` after timeout.
- `copy()` — defensive copy that completes independently.
- `newIncompleteFuture()` — subtype-friendly factory.
- `defaultExecutor()` — override the default executor in subclasses.

```java
User user = fetchUserAsync(id)
        .orTimeout(2, SECONDS)
        .exceptionally(ex -> User.UNKNOWN)
        .join();
```

## The Common ForkJoinPool Pitfall

By default, `supplyAsync` and `*Async` (no executor) run on the **common ForkJoinPool** (size `nCpu - 1`). This pool is shared across:
- All `CompletableFuture` async stages.
- All `parallelStream` operations.
- All other code using the common pool.

**If you block on it (I/O, DB, blocking HTTP), you starve everything.**

Fix: always pass an explicit executor for blocking work:

```java
ExecutorService ioExecutor = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors() * 8);

CompletableFuture.supplyAsync(() -> blockingHttpCall(), ioExecutor);
```

## Common Pitfalls

1. **Blocking on common pool** — see above.
2. **`thenApply` with `CompletableFuture` return** — produces nested `CompletableFuture<CompletableFuture<T>>`. Use `thenCompose`.
3. **Forgotten futures** — never `.join()` → exceptions silently swallowed. Always attach `exceptionally` or `handle`, or `join` somewhere.
4. **Assuming `thenApply` runs async** — by default runs on the completing thread, which may be the caller. Use `thenApplyAsync` for offloading.
5. **Mixing blocking and async** — calling `.join()` inside a `CompletableFuture` stage on the common pool blocks a carrier thread.
6. **`anyOf` returns `Object`** — loses type safety. Cast carefully or wrap.
7. **Not handling `TimeoutException`** — `orTimeout` fails the future; downstream sees it.
8. **Side effects in `whenComplete`** — if the action throws, the original result is replaced with the new exception.

## Code Examples

### Sequential composition

```java
CompletableFuture<Order> f = CompletableFuture
        .supplyAsync(() -> fetchUser(id), ioExecutor)
        .thenComposeAsync(user -> fetchOrders(user), ioExecutor)
        .thenApplyAsync(orders -> new Order(orders), cpuExecutor);
```

### Parallel fetch + combine

```java
CompletableFuture<User> userF = fetchUserAsync(id);
CompletableFuture<Settings> settingsF = fetchSettingsAsync(id);

CompletableFuture<Dashboard> dashboard = userF.thenCombine(settingsF, Dashboard::new)
        .thenApplyAsync(this::enrich, cpuExecutor)
        .exceptionally(ex -> Dashboard.EMPTY);
```

### Wait for all with type-safe list

```java
List<CompletableFuture<User>> futures = ids.stream()
        .map(this::fetchUserAsync)
        .toList();

CompletableFuture<Void> all = CompletableFuture.allOf(
        futures.toArray(CompletableFuture[]::new));

List<User> users = all.thenApply(_ -> futures.stream()
        .map(CompletableFuture::join)
        .toList()).join();
```

### Timeout + fallback

```java
User user = fetchUserAsync(id)
        .orTimeout(500, MILLISECONDS)
        .exceptionally(ex -> UserCache.get(id))
        .join();
```

## Common Interview Questions

1. **What is CompletableFuture?**
   → A `Future` that can be manually completed and chained with dependent stages via callbacks. Eliminates the need to poll `get()`.

2. **Difference between `thenApply` and `thenCompose`?**
   → `thenApply` is `map` — the function returns a plain value. `thenCompose` is `flatMap` — the function returns a `CompletableFuture`, preventing nested `CompletableFuture<CompletableFuture<T>>`.

3. **What's the difference between `Async` and non-Async variants?**
   → Non-Async runs on the thread that completed the previous stage (or the caller). `Async` runs on the common ForkJoinPool (or a provided executor). Use `Async` with explicit executor for predictable behavior.

4. **Why is blocking on the common ForkJoinPool dangerous?**
   → The pool is shared by all `CompletableFuture` async stages and all `parallelStream` calls. Blocking starves unrelated code. Always pass an explicit executor for I/O.

5. **What's `thenCombine`?**
   → Zips two independent futures into one. Both run in parallel; the combined future completes when both complete.

6. **How do you handle errors in a CompletableFuture?**
   → `exceptionally(ex -> fallback)` for recovery-only, `handle((t, ex) -> ...)` for both success and failure, `whenComplete((t, ex) -> ...)` for side-effects.

7. **What's the difference between `allOf` and `anyOf`?**
   → `allOf` waits for all input futures (returns `CompletableFuture<Void>`). `anyOf` completes when the first input completes (returns `CompletableFuture<Object>`).

8. **How do you add a timeout?**
   → `orTimeout(timeout, unit)` (Java 9+) — fails with `TimeoutException` after the duration. `completeOnTimeout(value, ...)` — completes with the given value instead.

9. **What happens if a CompletableFuture is never `join`ed and throws?**
   → The exception is silently swallowed unless `exceptionally` / `handle` is attached. Always handle errors, even if you don't await the result.

10. **What does `cancel(true)` do?**
    → Completes the future exceptionally with `CancellationException`. Does NOT actually interrupt the running task unless the task checks `Thread.interrupted()`.

## Related

- `Java/concurrency/executors-and-thread-pools.md` — executors, ForkJoinPool
- `Java/concurrency/virtual-threads.md` — Java 21 alternative to async chains
- `Java/concurrency/synchronized-and-volatile.md` — JMM, happens-before
- `Java/modern-java/stream-api.md` — parallel streams share the common pool

## Resources

- **Java docs:** `java.util.concurrent.CompletableFuture`
- **"Modern Java in Action"** — chapter on CompletableFuture
- **Spring docs:** async controller methods returning `CompletableFuture`
