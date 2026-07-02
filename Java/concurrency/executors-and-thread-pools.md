# Executors and Thread Pools

Modern Java concurrency infrastructure. Asked at Middle+ interviews — covers pool sizing, rejection policies, and the
pitfalls of `Executors.newFixedThreadPool`.

## Why Not `new Thread()`?

```java
new Thread(task).

start();   // DON'T
```

Problems:

1. **No reuse** — each task creates and destroys a thread (expensive: ~1MB stack, OS overhead).
2. **No bounds** — unbounded thread creation can starve the OS.
3. **No lifecycle management** — hard to shut down gracefully.
4. **No queueing** — can't buffer tasks when all threads are busy.

`ExecutorService` solves all of these.

## The Executor Framework

Hierarchy:

```
Executor                  (void execute(Runnable))
└── ExecutorService       (Future submit, shutdown, invokeAll)
    └── ScheduledExecutorService  (schedule, scheduleAtFixedRate)
```

`Executors` is a **factory class** — provides static methods to create common pools. Don't use these in production (see
below); use `ThreadPoolExecutor` directly.

## Executors Factory Methods

| Method                      | Behavior                                                     | Use with caution                   |
|-----------------------------|--------------------------------------------------------------|------------------------------------|
| `newFixedThreadPool(n)`     | Fixed n threads, unbounded `LinkedBlockingQueue`             | Tasks pile up in queue → OOM       |
| `newCachedThreadPool()`     | 0..Integer.MAX threads, 60s idle timeout, `SynchronousQueue` | Unbounded threads → OOM under load |
| `newSingleThreadExecutor()` | Single thread, unbounded queue                               | Tasks pile up → OOM                |
| `newScheduledThreadPool(n)` | For delayed/periodic tasks                                   | Same as fixed pool                 |

**Why avoid `Executors` in production:** unbounded queues (`newFixedThreadPool`) and unbounded threads (
`newCachedThreadPool`) hide load until OOM. Sonar / SpotBugs / IntelliJ inspections flag these.

## ThreadPoolExecutor Internals

The constructor exposes all the knobs:

```java
public ThreadPoolExecutor(
        int corePoolSize,                    // min threads kept alive
        int maximumPoolSize,                 // max threads
        long keepAliveTime, TimeUnit unit,   // idle timeout for non-core threads
        BlockingQueue<Runnable> workQueue,   // task queue
        ThreadFactory threadFactory,         // how to create threads (name, daemon)
        RejectedExecutionHandler handler     // what to do when full
)
```

### Task Submission Flow

1. If running threads < `corePoolSize` → create new thread for the task.
2. Else try to enqueue in `workQueue`.
3. If queue is full → create new thread up to `maximumPoolSize`.
4. If pool is maxed out and queue is full → invoke `RejectedExecutionHandler`.

**Subtle behavior:** with an unbounded queue, `maximumPoolSize` is **never reached** (queue never fills). This is why
`newFixedThreadPool` ignores the max concept — it sets core = max = n and uses an unbounded queue.

### Rejection Policies

| Policy                  | Behavior                                                    |
|-------------------------|-------------------------------------------------------------|
| `AbortPolicy` (default) | Throws `RejectedExecutionException`                         |
| `CallerRunsPolicy`      | Runs the task on the caller's thread — applies backpressure |
| `DiscardPolicy`         | Silently drops the new task                                 |
| `DiscardOldestPolicy`   | Drops the oldest queued task, then retries the new one      |

**In production:** `CallerRunsPolicy` is often the right choice — it slows the producer when the consumer can't keep up,
instead of failing or dropping.

## Thread Pool Sizing

### CPU-bound

```
N_threads = N_CPU + 1
```

The +1 accounts for occasional page faults / I/O. More threads just compete for CPU.

```java
int cpuBound = Runtime.getRuntime().availableProcessors() + 1;
```

### I/O-bound (DB, HTTP, file)

```
N_threads = N_CPU * U * (1 + W/C)
```

Where:

- `U` — target CPU utilization (e.g., 0.8 for 80%)
- `W/C` — ratio of wait time to compute time (e.g., 10 for mostly-waiting tasks)

Brian Goetz's formula. For I/O-bound workloads, this can be tens or hundreds. Be sure to cap at your resource limits (DB
connection pool, downstream service throughput).

```java
int ioBound = (int) (nCpu * 0.8 * (1 + 10.0));
```

### Reality Check

Most Spring apps use a single shared pool for HTTP requests (Tomcat's worker pool, typically 200 threads). For async
work, use a separate bounded pool sized to the bottleneck (e.g., DB pool size + 10).

## ThreadFactory

Default threads are non-daemon, named `pool-N-thread-M`. For production, always set:

- A meaningful name (for thread dumps, logs).
- Daemon flag (so JVM can exit).
- UncaughtExceptionHandler (logging).

```java
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setName("order-processor-" + t.threadId());
    t.setDaemon(true);
    t.setUncaughtExceptionHandler((thread, e) -> log.error("Uncaught in {}", thread.getName(), e));
    return t;
};
```

Spring Boot's `TaskExecutor` builders wrap this — prefer them over hand-rolled factories.

## ForkJoinPool

Special pool implementing **work-stealing**: idle threads steal tasks from other threads' queues. Used by:

- `parallelStream()` (uses the common pool — `ForkJoinPool.commonPool()`, size = `nCpu - 1`).
- `CompletableFuture` (also common pool by default).

```java
int sum = list.parallelStream().mapToInt(x -> x).sum();
```

**Caution:** the common pool is shared — blocking operations in `parallelStream` or `CompletableFuture.supplyAsync` can
starve unrelated code. Always pass an explicit executor for blocking work.

## CompletableFuture Basics

```java
CompletableFuture<Order> future = CompletableFuture
        .supplyAsync(() -> fetchOrder(id), orderExecutor)   // explicit executor
        .thenApply(Order::enrich)
        .thenCompose(o -> saveAsync(o, dbExecutor));
```

**Pitfalls:**

- Without an explicit executor, all stages run on the **common ForkJoinPool** — blocking there is catastrophic.
- `thenApply` runs on the completing thread (or the caller if not yet complete) — can cause unexpected thread switches.
  Use `thenApplyAsync` for predictable execution.
- Exceptions don't propagate to the caller unless you call `.join()` / `.get()` or attach `exceptionally` / `handle`.

## Graceful Shutdown

```java
executor.shutdown();              // stop accepting new tasks
if(!executor.

awaitTermination(60,SECONDS)){
List<Runnable> pending = executor.shutdownNow();  // interrupt running tasks
// log pending count
}
```

In Spring, `ExecutorService` beans should be destroyed in `@PreDestroy`:

```java

@PreDestroy
public void shutdown() throws InterruptedException {
    executor.shutdown();
    if (!executor.awaitTermination(30, SECONDS)) {
        executor.shutdownNow();
    }
}
```

Spring Boot's `ThreadPoolTaskExecutor` handles this automatically when configured as a bean.

## Common Pitfalls

1. **`Executors.newFixedThreadPool` with unbounded queue** → memory leak; tasks accumulate forever.
2. **`Executors.newCachedThreadPool`** → unbounded thread creation under burst → `OutOfMemoryError`.
3. **Wrong pool size** → too few: underutilization; too many: context-switch overhead.
4. **Blocking I/O on common ForkJoinPool** → starvation of `parallelStream` everywhere.
5. **Not naming threads** → thread dumps are useless.
6. **Not shutting down** → JVM hangs at exit (non-daemon threads).
7. **Returning `Future` and forgetting `get`** → exceptions swallowed, errors invisible.
8. **`submit(Runnable)` vs `execute(Runnable)`** — `submit` swallows uncaught exceptions into the `Future`. Use
   `execute` if you want the default uncaught handler to fire.

## Code Examples

### Production-ready pool

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4, 16,                                 // core, max
        60L, TimeUnit.SECONDS,                 // idle timeout
        new LinkedBlockingQueue<>(100),        // bounded queue
        new CustomThreadFactory("order"),      // named threads
        new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure
);
```

### Scheduled task

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.

scheduleAtFixedRate(this::cleanup, 0,10,TimeUnit.MINUTES);
```

### Spring Boot configuration

```java

@Configuration
public class PoolConfig {

    @Bean
    public ThreadPoolTaskExecutor orderExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(16);
        exec.setQueueCapacity(100);
        exec.setThreadNamePrefix("order-");
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setWaitForTasksToCompleteOnShutdown(true);
        exec.setAwaitTerminationSeconds(30);
        exec.initialize();
        return exec;
    }
}
```

### CompletableFuture with explicit executor

```java
CompletableFuture
        .supplyAsync(() ->

fetchUser(id),ioExecutor)
        .

thenApplyAsync(User::enrich, cpuExecutor)
    .

exceptionally(ex ->{log.

error("failed",ex); return null;});
```

## Common Interview Questions

1. **Why not use `new Thread()`?**
   → No reuse (expensive to create/destroy), no bounds (OOM risk), no queueing, no lifecycle. Use `ExecutorService`.

2. **What's wrong with `Executors.newFixedThreadPool`?**
   → Uses an unbounded `LinkedBlockingQueue`. Tasks pile up under load → `OutOfMemoryError`. Use `ThreadPoolExecutor`
   with a bounded queue.

3. **How does `ThreadPoolExecutor` decide to create a new thread?**
   → Core: if running < `corePoolSize`, create. Else: enqueue. If queue full: create up to `maximumPoolSize`. If maxed
   and queue full: reject.

4. **What's the difference between `corePoolSize` and `maximumPoolSize`?**
   → Core is the minimum kept alive (always running after warmup). Max is the cap; intermediate threads are created only
   when the queue is full, and idle ones are reaped after `keepAliveTime`.

5. **What are rejection policies?**
   → Behavior when pool + queue are full. `Abort` (throw), `CallerRuns` (backpressure), `Discard`, `DiscardOldest`. In
   production, `CallerRunsPolicy` is often best.

6. **How do you size a thread pool?**
   → CPU-bound: `N_CPU + 1`. I/O-bound: `N_CPU × U × (1 + W/C)` (Goetz formula). Cap by the bottleneck resource (DB
   pool, downstream throughput).

7. **What's wrong with blocking on the common `ForkJoinPool`?**
   → Shared by all `parallelStream` and default-executor `CompletableFuture` calls. Blocking starates unrelated code.
   Always pass an explicit executor for blocking work.

8. **What's the difference between `submit` and `execute`?**
   → `submit` returns a `Future` and swallows exceptions into it. `execute` propagates to the uncaught exception
   handler.

9. **How do you shut down an executor gracefully?**
   → `shutdown()` (stop accepting), then `awaitTermination(timeout)`. If still running, `shutdownNow()` (interrupt). In
   Spring, `ThreadPoolTaskExecutor` handles this via `@PreDestroy`.

10. **What is `CallerRunsPolicy` and when would you use it?**
    → Runs the rejected task on the calling thread. Provides natural backpressure — slows the producer. Use when you
    can't drop tasks and can afford the producer to block.

## Related

- `Java/concurrency/synchronized-and-volatile.md` — primitives
- `Java/concurrency/locks-and-atomic.md` — `ReentrantLock`, `Atomic*`
- `Java/concurrency/completable-future.md` — async composition
- `Java/concurrency/virtual-threads.md` — Java 21+ alternative
- `System-Design/HLD/caching-strategies.md` — backpressure patterns

## Resources

- **Brian Goetz, "Java Concurrency in Practice"** — pool sizing, executors
- **JEP 425 (Virtual Threads)** — Java 21+
- **Java docs:** `ThreadPoolExecutor` (read the class javadoc carefully)
- **Spring docs:** `ThreadPoolTaskExecutor` configuration
