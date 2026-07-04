# Thread Fundamentals

The foundation of Java concurrency: thread creation, lifecycle, states, interrupt mechanism, and the difference
between `Runnable`, `Callable`, and `Thread`. Every concurrency interview starts here.

## Creating Threads

### 1. Extending `Thread`

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + getName());
    }
}

var t = new MyThread();
t.start(); // new OS thread, calls run()
```

**Rarely used** — wastes the single-inheritance slot. Prefer `Runnable`.

### 2. `Runnable` (No Return Value)

```java
Runnable task = () -> System.out.println("Hello from " + Thread.currentThread().getName());

Thread t = new Thread(task, "worker-1"); // second arg = thread name
t.start();
```

### 3. `Callable<V>` (Returns a Value, Can Throw)

```java
Callable<Integer> task = () -> {
    TimeUnit.SECONDS.sleep(1);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);
Integer result = future.get(); // blocks until done → 42
```

### `Runnable` vs `Callable`

| Feature          | `Runnable`              | `Callable<V>`           |
|------------------|-------------------------|-------------------------|
| Return value     | `void`                  | `V`                     |
| Checked exceptions| Cannot throw            | Can throw `Exception`   |
| Submit to        | `Thread`, `ExecutorService` | `ExecutorService` only |
| Result access    | —                       | Via `Future<V>`         |

**Rule of thumb:** Use `Callable` when you need a result or need to propagate checked exceptions. Use `Runnable` for
fire-and-forget tasks.

## Thread Lifecycle (States)

```
        ┌─────────────────────────────────────────────────┐
        │                                                 │
   NEW ──▶ RUNNABLE ──▶ BLOCKED ──▶ RUNNABLE             │
        │      │            ▲                              │
        │      │            │ (waiting for monitor lock)   │
        │      ▼            │                              │
        │  WAITING ─────────┘                              │
        │  TIMED_WAITING ──────────────────────────────▶ TERMINATED
        │      │                                           ▲
        │      └──── (timeout expires / notified) ────────▶│
        └──────────────────────────────────────────────────┘
```

### Six States (`Thread.State`)

| State             | How to enter                                                    | How to exit                       |
|-------------------|-----------------------------------------------------------------|-----------------------------------|
| `NEW`             | `Thread t = new Thread(task)`                                   | `t.start()`                       |
| `RUNNABLE`        | `t.start()` or returned from wait/block                        | Scheduler preempts, enters wait/block, or `run()` ends |
| `BLOCKED`         | Waiting to acquire a `synchronized` monitor lock                | Lock acquired → `RUNNABLE`        |
| `WAITING`         | `Object.wait()`, `Thread.join()`, `LockSupport.park()`         | `notify()`/`notifyAll()`, joined thread ends, `unpark()` |
| `TIMED_WAITING`   | `Thread.sleep(ms)`, `wait(ms)`, `join(ms)`, `parkNanos(ns)`    | Timeout expires or notified       |
| `TERMINATED`      | `run()` completes (normally or via exception)                   | Final state — cannot restart      |

```java
Thread t = new Thread(() -> { /* ... */ });
System.out.println(t.getState()); // NEW

t.start();
System.out.println(t.getState()); // RUNNABLE (or TIMED_WAITING if sleeping)

t.join(); // wait for t to finish
System.out.println(t.getState()); // TERMINATED
```

**Note:** `RUNNABLE` in Java includes both "ready to run" and "actively running on a CPU". Java doesn't distinguish
these — the OS scheduler decides.

## `wait()` / `notify()` / `notifyAll()`

Low-level inter-thread communication using the object's monitor. **Must be called inside a `synchronized` block** on
the same object.

```java
class SharedQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public SharedQueue(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait(); // release lock, enter WAITING, re-acquire on wakeup
        }
        queue.add(item);
        notifyAll(); // wake up all waiting threads (consumers)
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait(); // release lock, wait for items
        }
        T item = queue.poll();
        notifyAll(); // wake up all waiting threads (producers)
        return item;
    }
}
```

### Critical Rules

1. **Always call `wait()` in a `while` loop** — never `if`. Reason: spurious wakeups (thread wakes without
   `notify`) and stolen conditions (another thread consumed the signal before you re-acquire the lock).

```java
// ❌ WRONG — vulnerable to spurious wakeups
synchronized (lock) {
    if (condition) wait();
    // condition might be false again here!
}

// ✅ CORRECT — re-check after waking
synchronized (lock) {
    while (condition) wait();
    // condition is guaranteed true here
}
```

2. **Prefer `notifyAll()` over `notify()`** — `notify()` wakes ONE arbitrary waiting thread. If multiple threads
   wait for different conditions, the wrong thread may wake up and go back to sleep (lost wakeup). `notifyAll()`
   is safer but slightly less efficient.

3. **Must hold the monitor** — calling `wait()`/`notify()` without `synchronized` throws
   `IllegalMonitorStateException`.

### `wait/notify` vs `Condition`

| Feature                 | `wait/notify`                    | `Condition` (with `Lock`)          |
|-------------------------|----------------------------------|------------------------------------|
| Multiple conditions     | ❌ One wait-set per object       | ✅ Multiple conditions per lock    |
| Interruptible wait      | ✅ `wait()` throws `InterruptedException` | ✅ `await()` + `awaitUninterruptibly()` |
| Timed wait              | ✅ `wait(ms)`                    | ✅ `await(time, unit)`, `awaitUntil(deadline)` |
| Fairness                | ❌ No                            | ✅ (if lock is fair)               |

**Modern code should prefer `Lock` + `Condition`.** `wait/notify` is legacy but still asked in interviews.

## Thread Interruption

The cooperative mechanism for requesting a thread to stop. **Threads are not forcibly killed** — they must check and
respond to the interrupt flag.

### How It Works

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            // do work...
            Thread.sleep(1000); // or wait(), join(), BlockingQueue.take()
        } catch (InterruptedException e) {
            // sleep/wait/join throws this when interrupted
            // The interrupt flag is CLEARED — you must decide what to do:

            Thread.currentThread().interrupt(); // ✅ restore flag (let caller know)
            break; // exit the loop
        }
    }
    System.out.println("Worker stopped gracefully");
});

worker.start();
Thread.sleep(3000);
worker.interrupt(); // set the interrupt flag → worker wakes from sleep
```

### Key Points

| Method                              | Behavior                                                    |
|-------------------------------------|-------------------------------------------------------------|
| `t.interrupt()`                     | Sets the interrupt flag. If `t` is in `sleep/wait/join`, it wakes and throws `InterruptedException` |
| `Thread.interrupted()`              | Returns `true` if flag is set **and clears it** (static, checks current thread) |
| `t.isInterrupted()`                 | Returns `true` if flag is set, **does not clear it** (instance method) |
| `InterruptedException`              | Thrown by blocking methods when interrupted. **Clears the flag** |

### Interrupt Handling Rules

```java
// ❌ NEVER swallow InterruptedException silently
catch (InterruptedException e) {
    // empty — caller never knows about the interrupt
}

// ✅ Option 1: Re-throw (let it propagate)
catch (InterruptedException e) {
    throw e;
}

// ✅ Option 2: Restore the flag and return
catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore flag
    return; // or break
}
```

## Daemon Threads

Background threads that **don't prevent JVM shutdown**. When all non-daemon threads finish, the JVM exits — killing
all daemon threads immediately (no cleanup, no `finally` blocks guaranteed).

```java
Thread t = new Thread(() -> {
    while (true) {
        cleanupOldFiles(); // background housekeeping
    }
});
t.setDaemon(true); // must set BEFORE start()
t.start();
```

**Use for:** Background tasks, monitoring, heartbeats, GC-like cleanup.
**Don't use for:** Tasks that must complete (I/O, transactions) — they'll be killed mid-execution.

All threads created by a daemon thread are also daemon by default.

## Thread Priority

```java
t.setPriority(Thread.MAX_PRIORITY);  // 10
t.setPriority(Thread.NORM_PRIORITY); // 5 (default)
t.setPriority(Thread.MIN_PRIORITY);  // 1
```

**In practice: priorities are unreliable.** The OS scheduler may ignore them. Don't rely on priority for correctness.
Use proper synchronization instead.

## `Thread.join()`

Block the current thread until the target thread finishes:

```java
Thread t1 = new Thread(() -> compute());
Thread t2 = new Thread(() -> compute());
t1.start();
t2.start();

t1.join(); // current thread waits for t1 to finish
t2.join(); // then waits for t2

// Both t1 and t2 are TERMINATED here
System.out.println("Both done");
```

`join(long millis)` — timed variant, returns after timeout even if the thread is still running.

**Happens-before:** Everything done in thread `t` before termination **happens-before** `join()` returns in the
calling thread.

## `Thread.sleep()` vs `Object.wait()` vs `Thread.yield()`

| Method          | Releases lock? | Purpose                                     |
|-----------------|:-:|----------------------------------------------|
| `Thread.sleep(ms)` | ❌ No       | Pause for a duration. Still holds locks      |
| `Object.wait()`    | ✅ Yes      | Release monitor, wait for `notify()`         |
| `Thread.yield()`   | ❌ No       | Hint to scheduler: "I can pause." Unreliable |
| `LockSupport.park()` | ❌ No    | Low-level park. Used by `Lock` internals     |

## `ThreadLocal`

Provides a **per-thread copy** of a variable — each thread has its own value, no synchronization needed.

```java
private static final ThreadLocal<SimpleDateFormat> dateFormat =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// Each thread gets its own SimpleDateFormat instance
String date = dateFormat.get().format(new Date());
```

### When to Use

- **Non-thread-safe objects** used in multi-threaded context (e.g., `SimpleDateFormat`, `Random`).
- **Request-scoped data** in web apps (user context, transaction ID, locale).
- **Avoiding parameter drilling** — pass request context implicitly.

### Pitfalls

```java
// ❌ Memory leak with thread pools — thread is reused, ThreadLocal is never cleaned
pool.submit(() -> {
    userContext.set(currentUser);
    process();
    // MUST clean up — otherwise next task on this thread sees stale data
});

// ✅ Always clean up in finally
pool.submit(() -> {
    try {
        userContext.set(currentUser);
        process();
    } finally {
        userContext.remove(); // prevent memory leak and stale data
    }
});
```

**`ThreadLocal` + thread pools = memory leak** if not cleaned. Each `ThreadLocal` stores a value in a `ThreadLocalMap`
on the `Thread` object. If the thread is reused (pool), old values accumulate.

### `InheritableThreadLocal`

Child threads inherit the parent's value at creation time:

```java
InheritableThreadLocal<String> ctx = new InheritableThreadLocal<>();
ctx.set("parent-value");

new Thread(() -> {
    System.out.println(ctx.get()); // "parent-value" — inherited
}).start();
```

**With virtual threads:** Use `ScopedValue` (Java 21+) instead — lighter, immutable, no cleanup needed. See
[Virtual Threads](./virtual-threads.md).

## `Thread.stop()`, `Thread.suspend()`, `Thread.resume()` — Deprecated

All three are **deprecated since Java 1.2** and removed in later versions:
- `stop()` — kills thread immediately, releases all monitors, leaves shared state inconsistent.
- `suspend()/resume()` — prone to deadlocks (suspended thread may hold a lock).

**Use interruption instead** — it's cooperative and safe.

## Common Interview Questions

1. **Thread states in Java?** — NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED. RUNNABLE includes both
   "ready" and "running" (Java doesn't distinguish).
2. **`Runnable` vs `Callable`?** — `Runnable.run()` returns void and can't throw checked exceptions. `Callable.call()`
   returns a value and can throw. `Callable` is submitted to `ExecutorService` and produces a `Future`.
3. **How does `interrupt()` work?** — Sets the interrupt flag. If the thread is in `sleep/wait/join`, it throws
   `InterruptedException` (and clears the flag). Otherwise, the thread must check `isInterrupted()`. Never swallow
   `InterruptedException` — re-throw or restore the flag.
4. **`wait()` vs `sleep()`?** — `wait()` releases the monitor lock and waits for `notify()`. `sleep()` pauses but
   holds all locks. `wait()` must be called inside `synchronized`.
5. **Why call `wait()` in a `while` loop?** — Spurious wakeups: the thread can wake without `notify()`. Stolen
   conditions: another thread may have consumed the signal. The `while` loop re-checks the condition after waking.
6. **What is `ThreadLocal`? Pitfalls?** — Per-thread variable storage, no synchronization needed. Main pitfall:
   memory leak in thread pools — if not cleaned up with `remove()`, stale values accumulate on reused threads.
7. **Daemon vs non-daemon threads?** — Daemon threads don't prevent JVM shutdown. When all non-daemon threads finish,
   JVM exits and kills all daemons immediately. Set with `setDaemon(true)` before `start()`.
8. **Why are `Thread.stop()` and `Thread.suspend()` deprecated?** — `stop()` leaves shared state inconsistent
   (releases locks mid-operation). `suspend()` can cause deadlocks (holds locks while suspended). Use cooperative
   interruption instead.

## Related

- [synchronized and volatile](./synchronized-and-volatile.md) — monitors, visibility, JMM
- [Executors and Thread Pools](./executors-and-thread-pools.md) — don't create threads manually
- [Virtual Threads](./virtual-threads.md) — modern alternative, ScopedValue vs ThreadLocal
- [Locks and Atomic](./locks-and-atomic.md) — `Condition` as modern replacement for `wait/notify`

## Resources

- Brian Goetz — *Java Concurrency in Practice* (chapters 5-7)
- [JLS §17.2 — Wait Sets and Notification](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.2)
- [Baeldung — Thread Lifecycle](https://www.baeldung.com/java-thread-lifecycle)
