# synchronized and volatile

Two core concurrency primitives in Java. `synchronized` provides mutual exclusion AND visibility; `volatile` provides
visibility only. Using them correctly requires understanding the Java Memory Model (JMM).

## Java Memory Model (JMM) Primer

The JMM defines how threads interact through memory. Key concepts:

- **Main memory** vs **thread-local caches** (CPU caches, registers). Threads may read stale values from their cache.
- **Visibility** — changes made by one thread become visible to others.
- **Atomicity** — operations appear indivisible.
- **Ordering** — compiler/CPU may reorder instructions for performance.
- **happens-before** — a partial order guaranteeing that memory writes by one action are visible to another.

Without JMM guarantees, the compiler/CPU may reorder reads/writes, cache values, and produce "impossible" results (e.g.,
a thread seeing a partially constructed object).

## synchronized

`synchronized` provides:

1. **Mutual exclusion** — only one thread holds the monitor at a time.
2. **Visibility** — releasing the monitor flushes writes to main memory; acquiring rereads from memory.
3. **Atomicity** of the synchronized block as a whole.

Every Java object has an intrinsic lock (monitor). `synchronized` acquires/releases it.

### Three Forms

```java
public synchronized void method() { ...}           // locks `this`

public static synchronized void method() { ...}    // locks the Class object

public void method() {
    synchronized (lock) { ...}                      // locks `lock`
}
```

### Reentrancy

A thread holding a monitor can re-enter it (counter incremented). Prevents self-deadlock:

```java
public synchronized void outer() {
    inner();   // same thread re-acquires the same monitor — works
}

public synchronized void inner() { ...}
```

### Cost

Pre-Java 6: heavy (OS-level "fat lock"). Modern JVMs use **biased locking**, **thin locks**, **lock inflation** to
optimize uncontended cases. Uncontended `synchronized` is ~nanoseconds. Contended locks are the real cost (context
switches, OS park/unpark).

Java 15+ **disabled biased locking by default** (JEP 374) — contention is rare in modern apps and the code complexity
wasn't worth it.

## volatile

`volatile` provides:

1. **Visibility** — reads/writes go directly to main memory (no caching in registers/threads).
2. **Ordering** — a volatile read/write creates a memory barrier preventing certain reorderings.
3. **Atomicity** for a single read/write of the field (NOT for compound operations like `++`).

```java
private volatile boolean running = true;

public void stop() {
    running = false;
}

public void run() {
    while (running) { /* ... */ }   // without volatile, may never see the update
}
```

### What volatile Does NOT Provide

`volatile int counter; counter++;` is **NOT atomic**. `counter++` is read-modify-write — three operations. Two threads
can both read 5, both write 6 → lost update. Use `AtomicInteger`:

```java
private AtomicInteger counter = new AtomicInteger();
counter.

incrementAndGet();   // atomic
```

### When to Use volatile

- **Status flags** (`boolean running`, `boolean initialized`).
- **Publishing immutable state** — a `volatile` reference to an effectively immutable object published safely.
- **Double-checked locking** (see below) — paired with `synchronized`.

## happens-before Relationship

The JMM defines a partial order. If action A **happens-before** B, then A's effects are visible to B. Established by:

- Program order (within a single thread)
- Monitor lock — `unlock` happens-before subsequent `lock` on the same monitor
- `volatile` field — write happens-before subsequent read
- `Thread.start()` — happens-before any action in the started thread
- `Thread.join()` — actions in the joined thread happen-before `join` returns
- `Thread.interrupt()` (and various `java.util.concurrent` operations)

This is why `volatile boolean running` works in the producer-consumer pattern — the writer's updates happen-before the
reader's reads.

## synchronized vs volatile

| Property                  | synchronized                 | volatile                        |
|---------------------------|------------------------------|---------------------------------|
| Mutual exclusion          | yes                          | no                              |
| Visibility                | yes                          | yes                             |
| Atomicity of compound ops | yes                          | no                              |
| Performance (uncontended) | nanoseconds                  | very fast, no lock              |
| Performance (contended)   | context switches, slower     | no contention, but no exclusion |
| Can block                 | yes                          | no                              |
| Use case                  | Critical sections, mutations | Flags, safe publication         |

## Double-Checked Locking (DCL)

Lazy initialization pattern. Requires `volatile` to prevent seeing a partially constructed object due to instruction
reordering.

```java
public class Singleton {

    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {                    // 1: check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {             // 2: check (under lock)
                    instance = new Singleton();     // 3: assign
                }
            }
        }
        return instance;
    }
}
```

**Why `volatile` is required:** `instance = new Singleton()` is three steps:

1. Allocate memory.
2. Run constructor.
3. Assign reference.

The JIT may reorder steps 2 and 3. A thread at check 1 could see the assigned reference before the constructor runs →
use a half-initialized object.

**Alternative (cleaner):** use a holder class — lazy init is guaranteed by class loading:

```java
public class Singleton {
    private Singleton() {
    }

    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

## Common Pitfalls

1. **`volatile` for `++`** — not atomic; use `AtomicInteger`.
2. **Synchronizing on a non-`final` field** — if the field changes, callers may lock on different objects.
3. **Synchronizing on a String literal / Integer cache** — other code in the JVM may lock on the same instance →
   unexpected contention or deadlock.
4. **Holding a lock during I/O** — long I/O blocks all other threads waiting for the lock. Move I/O outside critical
   sections.
5. **Forgetting `volatile` in DCL** — works "most of the time" but breaks under JIT optimization on multi-core.
6. **Assuming `volatile` makes everything visible** — only the volatile field is directly visible; non-volatile fields
   written before it become visible via happens-before, but only if the ordering is established.

## Code Examples

### Thread-safe counter with synchronized

```java
public class Counter {
    private int count;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }
}
```

### Thread-safe flag with volatile

```java
public class Worker {
    private volatile boolean running = true;

    public void stop() {
        running = false;
    }

    public void run() {
        while (running) {
            doWork();
        }
    }
}
```

### Lock on a private final monitor

```java
public class Cache {
    private final Object lock = new Object();
    private final Map<String, String> map = new HashMap<>();

    public void put(String k, String v) {
        synchronized (lock) {
            map.put(k, v);
        }
    }
}
```

## Common Interview Questions

1. **What's the difference between synchronized and volatile?**
   → synchronized provides mutual exclusion + visibility + atomicity of compound ops. volatile provides visibility +
   ordering only — no mutual exclusion, no atomicity for compound ops like `++`.

2. **What is the Java Memory Model?**
   → Specification of how threads interact through memory: visibility, atomicity, ordering, happens-before. Defines when
   one thread's writes are guaranteed visible to another.

3. **What is happens-before?**
   → A partial order guaranteeing memory effects of action A are visible to action B if A happens-before B. Established
   by monitor unlock→lock, volatile write→read, Thread.start, Thread.join, etc.

4. **Why does `counter++` need synchronization even with volatile?**
   → `++` is read-modify-write (3 steps). volatile makes each individual read/write visible but doesn't make the
   compound operation atomic. Use `AtomicInteger` or `synchronized`.

5. **Why is `volatile` required in double-checked locking?**
   → Without volatile, the JIT may reorder constructor and assignment. A thread checking `instance != null` could see a
   non-null reference to a half-constructed object.

6. **Can synchronized cause deadlock?**
   → Yes — if thread A holds lock1 and waits for lock2, while thread B holds lock2 and waits for lock1. Use a consistent
   lock ordering or `tryLock` with timeout.

7. **What is a monitor (intrinsic lock)?**
   → Every Java object has an associated monitor. synchronized acquires it. Monitors are reentrant — the same thread can
   re-acquire.

8. **Is synchronized reentrant?**
   → Yes. The same thread can re-enter the monitor; the JVM tracks the count. Prevents self-deadlock when synchronized
   methods call each other.

9. **What's the cost of uncontended synchronized?**
   → Nanoseconds — modern JVMs use biased/thin locks. Java 15+ removed biased locking by default; still very fast.

10. **Why not synchronize on String literals or Integer?**
    → They may be interned/cached and shared across unrelated code → accidental contention or deadlock. Always
    synchronize on a private `final Object`.

## Related

- `Java/concurrency/executors-and-thread-pools.md` — thread pools and executors
- `Java/concurrency/locks-and-atomic.md` — `ReentrantLock`, `Atomic*` classes
- `Java/jvm-internals/memory-model-and-gc.md` — JVM memory, GC
- `Java/core/collections-hashmap-internals.md` — `ConcurrentHashMap` uses volatile
- `CS-Fundamentals/Operating-Systems/concurrency-basics.md` — OS-level concepts

## Resources

- **JSR 133 (Java Memory Model):** https://www.cs.umd.edu/~pugh/java/memoryModel/
- **Brian Goetz, "Java Concurrency in Practice"** — the canonical book
- **JEP 374 (biased locking removed):** https://openjdk.org/jeps/374
- **Aleksey Shipilëv, "JMM Pragmatics":** https://shipilev.net/
