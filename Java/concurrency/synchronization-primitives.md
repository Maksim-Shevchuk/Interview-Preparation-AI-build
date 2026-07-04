# Synchronization Primitives

`CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`, and `Exchanger` — higher-level coordination tools from
`java.util.concurrent`. Essential for senior-level interviews and real-world concurrent system design.

## Overview

| Primitive          | Purpose                              | Reusable? | Parties   |
|--------------------|--------------------------------------|-----------|-----------|
| `CountDownLatch`   | Wait for N events to happen          | ❌ No     | Fixed     |
| `CyclicBarrier`    | N threads wait for each other        | ✅ Yes    | Fixed     |
| `Semaphore`        | Limit concurrent access to N         | ✅ Yes    | Dynamic   |
| `Phaser`           | Multi-phase barrier, dynamic parties | ✅ Yes    | Dynamic   |
| `Exchanger`        | Two threads swap values              | ✅ Yes    | Exactly 2 |

## CountDownLatch

A one-shot latch: one or more threads wait until a **count reaches zero**. Cannot be reset.

```java
CountDownLatch latch = new CountDownLatch(3); // 3 events to wait for

// Workers count down
for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        try {
            doWork();
        } finally {
            latch.countDown(); // decrement count (thread-safe, non-blocking)
        }
    });
}

// Main thread waits
latch.await(); // blocks until count reaches 0
System.out.println("All 3 workers done");

// With timeout
boolean completed = latch.await(5, TimeUnit.SECONDS);
```

### Use Cases

```java
// 1. Wait for services to initialize before accepting traffic
CountDownLatch ready = new CountDownLatch(3);

startDatabasePool(ready::countDown);
startCacheWarmer(ready::countDown);
startHealthCheck(ready::countDown);

ready.await(); // blocks until all 3 services report ready
startAcceptingRequests();

// 2. Coordinate test: ensure N threads start simultaneously
CountDownLatch startGun = new CountDownLatch(1);

for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        startGun.await(); // all threads block here
        // all released simultaneously
        runLoadTest();
    }).start();
}

Thread.sleep(100); // let all threads reach await()
startGun.countDown(); // fire! all 10 threads start at once
```

**Key point:** `countDown()` can be called by **any thread**, not just the waiting ones. One thread can count down
multiple times. The latch doesn't track WHO counted down.

## CyclicBarrier

N threads wait for **each other** at a common barrier point. When all arrive, they are released simultaneously.
**Can be reused** (cyclic).

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    // Optional barrier action — runs ONCE when all threads arrive
    System.out.println("All 3 threads reached the barrier");
});

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        computePartialResult();
        barrier.await(); // blocks until all 3 threads call await()

        // All threads resume here simultaneously
        mergeResults();
        barrier.await(); // can reuse for next phase!
    });
}
```

### CountDownLatch vs CyclicBarrier

| Feature                | `CountDownLatch`                  | `CyclicBarrier`                    |
|------------------------|-----------------------------------|------------------------------------|
| Who waits?             | Thread(s) calling `await()`       | All participating threads          |
| Who counts?            | Any thread calls `countDown()`    | Each thread calls `await()`        |
| Reusable?              | ❌ No                             | ✅ Yes (auto-resets)               |
| Barrier action?        | ❌ No                             | ✅ Optional `Runnable`             |
| On failure?            | Other waiters unaffected          | `BrokenBarrierException` for all   |
| Typical use            | "Wait for N tasks to complete"    | "N threads synchronize at a point" |

### `BrokenBarrierException`

If any thread is interrupted or times out while waiting, the barrier **breaks** — all other waiting threads receive
`BrokenBarrierException`. The barrier must be `reset()` to be reused after breaking.

## Semaphore

Controls access to a resource by maintaining a set of **permits**. Threads acquire permits before accessing the
resource and release them when done.

```java
Semaphore semaphore = new Semaphore(5); // 5 permits = max 5 concurrent accesses

void accessResource() throws InterruptedException {
    semaphore.acquire();  // blocks if no permits available (or acquire(n) for multiple)
    try {
        useSharedResource();
    } finally {
        semaphore.release(); // return permit
    }
}
```

### Variants

```java
// Try without blocking
if (semaphore.tryAcquire()) {
    try { useResource(); }
    finally { semaphore.release(); }
} else {
    handleRejection(); // resource full
}

// Try with timeout
if (semaphore.tryAcquire(2, TimeUnit.SECONDS)) { ... }

// Fair semaphore — FIFO ordering for waiting threads
Semaphore fair = new Semaphore(5, true);

// Binary semaphore (mutex)
Semaphore mutex = new Semaphore(1);
```

### Use Cases

```java
// 1. Connection pool — limit concurrent DB connections
Semaphore connectionLimit = new Semaphore(10);

Connection getConnection() throws InterruptedException {
    connectionLimit.acquire();
    return pool.borrowConnection();
}

void returnConnection(Connection conn) {
    pool.returnConnection(conn);
    connectionLimit.release();
}

// 2. Rate limiter — max N requests per window
Semaphore rateLimiter = new Semaphore(100); // 100 concurrent requests

// 3. Producer-consumer with bounded buffer (alternative to BlockingQueue)
```

### Semaphore vs Lock

| Feature          | `Semaphore`                       | `ReentrantLock`                  |
|------------------|-----------------------------------|----------------------------------|
| Permits          | 1 to N                            | 1 (mutual exclusion)             |
| Owner            | No ownership — any thread releases| Owner thread must unlock         |
| Reentrant        | ❌ No                             | ✅ Yes                           |
| Use case         | Limit concurrency to N            | Mutual exclusion of a section    |

## Phaser

The most flexible barrier — supports **dynamic registration** of parties and **multiple phases**. Generalizes both
`CountDownLatch` and `CyclicBarrier`.

```java
Phaser phaser = new Phaser(1); // 1 = main thread is a party

for (int i = 0; i < 3; i++) {
    phaser.register(); // dynamically add a party
    executor.submit(() -> {
        // Phase 0: initialize
        initialize();
        phaser.arriveAndAwaitAdvance(); // barrier

        // Phase 1: process
        process();
        phaser.arriveAndAwaitAdvance(); // barrier

        // Phase 2: cleanup
        cleanup();
        phaser.arriveAndDeregister(); // done, leave the phaser
    });
}

phaser.arriveAndDeregister(); // main thread deregisters
```

### When to Use Phaser

- **Dynamic party count** — threads can register/deregister at any time.
- **Multi-phase computation** — like CyclicBarrier but with phase tracking.
- **Conditional participation** — `onAdvance()` can be overridden to control termination.

```java
// Self-terminating phaser — stops after 5 phases
Phaser phaser = new Phaser(3) {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        return phase >= 4 || registeredParties == 0; // terminate after phase 4
    }
};
```

## Exchanger

A synchronization point where **exactly two threads** swap values.

```java
Exchanger<List<String>> exchanger = new Exchanger<>();

// Thread 1: producer fills a buffer, exchanges it for an empty one
executor.submit(() -> {
    List<String> buffer = new ArrayList<>();
    while (true) {
        buffer.add(produce());
        if (buffer.size() >= BATCH_SIZE) {
            buffer = exchanger.exchange(buffer); // give full buffer, get empty one back
            buffer.clear();
        }
    }
});

// Thread 2: consumer gets a full buffer, exchanges it for an empty one
executor.submit(() -> {
    List<String> buffer = new ArrayList<>();
    while (true) {
        buffer = exchanger.exchange(buffer); // give empty buffer, get full one back
        process(buffer);
    }
});
```

**Use case:** Double-buffering — one thread fills a buffer while the other processes the previous one. Rare in practice.

## Classic Concurrency Problems

### Deadlock

Four conditions (all must hold simultaneously):

1. **Mutual exclusion** — resource is held exclusively.
2. **Hold and wait** — thread holds one resource while waiting for another.
3. **No preemption** — resources can't be forcibly taken away.
4. **Circular wait** — A waits for B, B waits for A.

```java
// ❌ Deadlock — inconsistent lock ordering
void transferA(Account from, Account to, int amount) {
    synchronized (from) {          // Thread 1 locks A
        synchronized (to) {        // Thread 1 waits for B
            from.debit(amount);
            to.credit(amount);
        }
    }
}
// Thread 2 calls transfer(B, A) — locks B, waits for A → DEADLOCK

// ✅ Fix: consistent lock ordering
void transfer(Account from, Account to, int amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Prevention strategies:**
- **Lock ordering** — always acquire locks in the same global order.
- **`tryLock` with timeout** — back off and retry.
- **Single lock** — use one lock for related resources.
- **Avoid nesting** — restructure code to use fewer locks.

### Livelock

Threads are not blocked but keep **retrying without making progress**. Like two people stepping aside for each other
in a hallway, then stepping to the same side.

```java
// ❌ Livelock — both threads keep backing off and retrying
void transfer(Lock lockA, Lock lockB) {
    while (true) {
        if (lockA.tryLock()) {
            try {
                if (lockB.tryLock()) {
                    try {
                        doTransfer();
                        return;
                    } finally { lockB.unlock(); }
                }
            } finally { lockA.unlock(); }
        }
        // Both threads reach here simultaneously, retry, fail again...
    }
}

// ✅ Fix: random backoff
Random random = ThreadLocalRandom.current();
Thread.sleep(random.nextInt(10)); // randomized delay breaks the livelock cycle
```

### Starvation

A thread **never gets access** to the resource because other threads keep taking it.

**Causes:** Unfair locks (new arrivals bypass waiting threads), low-priority threads never scheduled, thread holds
a lock for too long.

**Fix:** Fair locks (`new ReentrantLock(true)`), `Semaphore(n, true)`, prioritize starving threads.

### Priority Inversion

A high-priority thread waits for a low-priority thread (which holds a lock), while a medium-priority thread runs
instead of the low-priority one — effectively inverting priorities.

**Fix:** Priority inheritance (OS-level), avoid holding locks during long operations.

## Common Interview Questions

1. **`CountDownLatch` vs `CyclicBarrier`?** — `CountDownLatch`: one-shot, any thread counts down, waiting threads
   are separate from counting threads. `CyclicBarrier`: reusable, all participating threads call `await()` and are
   released together.
2. **What is a `Semaphore`?** — Controls concurrent access by maintaining permits. `acquire()` blocks if no permits.
   `release()` returns a permit. Unlike locks, permits are not bound to threads — any thread can release.
3. **What are the four conditions for deadlock?** — Mutual exclusion, hold and wait, no preemption, circular wait.
   Break any one to prevent deadlock. Most practical: enforce consistent lock ordering.
4. **Deadlock vs livelock vs starvation?** — Deadlock: threads blocked forever. Livelock: threads running but making
   no progress (keep retrying). Starvation: a thread never gets access because others monopolize the resource.
5. **When to use `Phaser` over `CyclicBarrier`?** — When the number of parties changes dynamically
   (register/deregister), or when you need multi-phase coordination with per-phase control via `onAdvance()`.
6. **How to prevent deadlock in a bank transfer?** — Lock ordering: always lock accounts in the same order (e.g.,
   by account ID). Or use `tryLock` with timeout and retry with random backoff.

## Related

- [Thread Fundamentals](./thread-fundamentals.md) — thread states, wait/notify
- [Locks and Atomic](./locks-and-atomic.md) — ReentrantLock, tryLock, Condition
- [Executors and Thread Pools](./executors-and-thread-pools.md) — BlockingQueue as executor task queue
- [Concurrent Collections](./concurrent-collections.md) — BlockingQueue implementations

## Resources

- Brian Goetz — *Java Concurrency in Practice* (chapters 5, 8, 14)
- [JDK Docs — java.util.concurrent](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Baeldung — CountDownLatch](https://www.baeldung.com/java-countdown-latch)
- [Baeldung — CyclicBarrier](https://www.baeldung.com/java-cyclic-barrier)
