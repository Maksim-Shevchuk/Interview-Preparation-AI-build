# The happens-before Relationship

The **happens-before** relationship is the central concept of the **Java Memory Model (JMM)**, defined in
**JLS §17.4.5**. It is a *partial order* over actions (reads, writes, lock/unlock, thread start/join, etc.) that
**guarantees memory effects of one action are visible to another**. Every correct concurrent Java program is
correct *because* of happens-before edges — even when the code never mentions them explicitly.

A recurring senior-level interview question — expect to enumerate the rules, explain why double-checked locking
needs `volatile`, why `final` fields are safe, and why `AtomicInteger.get()` is enough after a `set()`.

---

## Quick Reference — The Rules

| #  | Rule                       | Statement                                                                   |
|----|----------------------------|-----------------------------------------------------------------------------|
| 1  | **Single thread** (program order) | Actions in a thread happen-before every later action in the **same** thread |
| 2  | **Monitor lock**           | An `unlock` on a monitor happens-before every subsequent `lock` on the **same** monitor |
| 3  | **Volatile field**         | A write to a `volatile` field happens-before every subsequent read of that field |
| 4  | **Thread start**           | `Thread.start()` happens-before any action in the started thread            |
| 5  | **Thread termination**     | Actions in a thread happen-before another thread successfully returns from `join()` or `isAlive() == false` |
| 6  | **Interruption**           | A thread calling `interrupt()` happens-before the interrupted thread detects the interrupt (exception or `isInterrupted()`) |
| 7  | **Object finalization**    | The end of a constructor happens-before the start of a finalizer            |
| 8  | **Transitivity**           | If A happens-before B and B happens-before C, then A happens-before C       |
| 9  | **`java.util.concurrent`** | Each synchronizer class (locks, atomics, executors, queues, etc.) documents its own happens-before guarantees |

---

## 1. Why happens-before Exists

### The myth of "the program is executed in order"

Java source code is not what executes. The path from your `int x = 1; y = 2;` to CPU instructions includes:

1. **The compiler** (javac + JIT) reorders instructions for performance.
2. **The CPU** executes out of order (superscalar pipelines, branch prediction).
3. **CPU caches** are per-core and not coherent with main memory by default.
4. **Store buffers** and **invalidate queues** defer visibility to other cores.

Each layer is allowed to reorder *as long as a single-threaded program can't tell the difference* (**as-if-serial**
semantics). The problem: that license silently breaks **multi-threaded** code. Without explicit guarantees:

- A thread may see `flag == true` while the data it was supposed to guard is still **stale**.
- A thread may see a non-null reference to an object whose **constructor hasn't finished**.
- A `HashMap` written by one thread may appear infinitely looped to another.

### The JMM fix

The JMM defines:

- **Visibility** — when does a write by thread A become visible to thread B?
- **Ordering** — what reorderings are *forbidden* between threads?
- **Atomicity** — which operations are indivisible?

`happens-before` is the *single* primitive that ties them together. If **A happens-before B**, then:

1. A's effects are **visible** to B.
2. A and B are **ordered** — no compiler or CPU may reorder them in a way observable to B.

If there is **no** happens-before edge between A and B, the JMM provides **no** guarantee — they are in a **data
race**, and the program has undefined behaviour (in the JMM sense: arbitrary values, including "impossible" ones).

---

## 2. The Definition

The JLS defines two relations over actions:

- **Synchronizes-with** (`sw`): pairs of *synchronization actions* — e.g., a `volatile` write `sw` a subsequent
  `volatile` read of the same variable; a `unlock` `sw` a subsequent `lock`.
- **Program order** (`po`): the order of actions within a single thread.

The **happens-before** relation (`hb`) is then:

```
   hb = (po ∪ sw)⁺        — the transitive closure of (program order ∪ synchronizes-with)
```

In plain English:

> If there is a path from A to B that walks program-order edges (inside a thread) and synchronizes-with edges
> (across threads), then A **happens-before** B.

```
   Thread A                       Thread B
   ────────                       ────────
   write x = 1
   write v = true                 ──── hb ───▶  read v == true   (volatile write sw volatile read)
                                                  read x            → guaranteed to see 1
```

The `volatile` write on A is in program order after `x = 1`; it synchronizes-with the volatile read on B; the read
of `x` is in program order after the volatile read. The transitive chain delivers the guarantee.

---

## 3. The Rules in Detail

### 3.1 Single-thread rule (program order)

> Each action in a thread happens-before every action later in **program order** within the same thread.

This is just the as-if-serial guarantee. Within a thread, code runs as written *as far as that thread can observe*.
It does **not** stop reordering — it just guarantees the thread itself can't observe the reordering.

### 3.2 Monitor lock rule

> An `unlock` on a monitor happens-before every subsequent `lock` of the **same** monitor.

```java
class Holder {
    int a = 0, b = 0;            // non-volatile, no final
    final Object lock = new Object();

    void writer() {
        synchronized (lock) {    // lock
            a = 1; b = 2;        // writes happen INSIDE the lock
        }                        // unlock — hb every subsequent lock
    }

    void reader() {
        synchronized (lock) {    // lock — sees everything from the unlock above
            int x = a;           // guaranteed 1 if reader ran after writer released
            int y = b;           // guaranteed 2
        }
    }
}
```

The unlock **flushes** the writes to main memory; the next lock **invalidates** the reading thread's cache. Both
orderings are also forbidden from being reordered across the boundary.

> **Same monitor** is critical. Two different lock objects give **no** happens-before edge.

### 3.3 Volatile field rule

> A write to a `volatile` field happens-before every subsequent read of that field.

`volatile` is the **lightest** way to establish a cross-thread happens-before edge — no mutual exclusion, just
visibility + ordering for the single field.

```java
class Publisher {
    private int data;                 // non-volatile
    private volatile boolean ready;   // the "publication flag"

    void write() {
        data = 42;            // (1) program order before
        ready = true;         // (2) volatile write
    }

    void read() {
        if (ready) {                  // (3) volatile read
            int x = data;             // (4) guaranteed to see 42
        }
    }
}
```

Why does (4) see 42, even though `data` is not volatile? Because of transitivity:
- (1) po-hb (2)
- (2) sw-hb (3) [volatile write sw volatile read]
- (3) po-hb (4)
- ⇒ (1) hb (4)

A `volatile` write/read pair acts as a **memory fence**: it publishes everything the writer did before, and orders
everything the reader does after.

> **What volatile does NOT do:** it does **not** make `++` atomic. `i++` is read-modify-write — three separate
> operations, each atomic alone, but the whole sequence races. Use `AtomicInteger` or `synchronized`.

### 3.4 Thread start rule

> Actions in a thread before it calls `Thread.start()` happen-before any action in the started thread.

```java
class Config {
    Map<String, String> values;       // not synchronized
}

Config cfg = new Config();
cfg.values = loadFromDisk();          // (1) before start
Thread t = new Thread(() -> {
    String v = cfg.values.get("k");   // (2) guaranteed to see the populated map
});
t.start();                            // establishes hb (1) → (2)
```

`start()` is itself a synchronization action. You don't need extra locking to publish data **to** a freshly started
thread.

### 3.5 Thread termination rule

> Actions in a thread happen-before another thread successfully returns from `join()` on it (or observes
> `isAlive() == false`).

```java
class Worker extends Thread {
    long result;
    public void run() {
        result = expensiveComputation();   // (1) inside worker
    }
}

Worker w = new Worker(); w.start();
w.join();                                  // (2)
long r = w.result;                         // (3) guaranteed to see the value written in run()
```

`join()` is the symmetric twin of `start()`: it lets you retrieve results from a finished thread without extra
locking.

### 3.6 Interruption rule

> A thread calling `interrupt()` on another thread happens-before the interrupted thread observes the interrupt
> (via `InterruptedException` or `Thread.isInterrupted()`).

Useful when one thread signals a worker to stop and expects the worker to see the **state set before** the
interrupt call.

### 3.7 Object finalization rule

> The end of a constructor happens-before the start of the finalizer for that object.

Rarely practically important, but completes the picture — even the GC respects happens-before.

### 3.8 Transitivity (the multiplier)

> If A happens-before B and B happens-before C, then A happens-before C.

This is the rule that makes the model compositional. Complex publication chains reduce to a graph of program-order
and synchronizes-with edges.

```
   Thread A        Thread B          Thread C
   write x=1       read v1           read v2
   write v1        write v2          read x   ← sees 1
```

Each volatile write/read hands the chain forward.

### 3.9 Synchronizers in `java.util.concurrent`

Every `java.util.concurrent` class documents its happens-before guarantees. Examples:

| Class                       | Happens-before edge                                    |
|-----------------------------|--------------------------------------------------------|
| `Lock.unlock()`             | → subsequent `Lock.lock()` on the same lock            |
| `Atomic*` set/inc/etc.      | → subsequent `Atomic*` get on the same variable        |
| `Semaphore.release()`       | → subsequent `acquire()` on the same semaphore         |
| `CountDownLatch.countDown()`| → `await()` returning                                  |
| `CyclicBarrier`             | barrier arrival → subsequent barrier actions           |
| `Phaser.arriveAndAwaitAdvance()` | → next phase actions                              |
| `Exchanger.exchange()`      | actions before exchange → actions after                |
| `BlockingQueue.put()`       | → corresponding `take()`/`poll()`                      |
| `Executor.execute()`        | → start of the submitted task                          |
| `ExecutorService.submit()`  | → start of the task; task completion → `Future.get()`   |
| `ConcurrentHashMap`         | operations have hb edges consistent with the map being a thread-safe wrapper |

This is why you rarely write raw `volatile` or `synchronized` in modern Java — `java.util.concurrent` does it for
you, and documents the consequences.

---

## 4. Memory Barriers (Under the Hood)

The JVM emits CPU **memory barriers** ("fences") to implement happens-before. There are four canonical types:

| Barrier        | Prevents                                              |
|----------------|-------------------------------------------------------|
| **LoadLoad**   | A load before the fence finishing before a load after |
| **StoreStore** | A store before finishing before a store after         |
| **LoadStore**  | A load before finishing before a store after          |
| **StoreLoad**  | A store before finishing before a load after          |

Mapping to Java constructs:

```
   volatile write   ≈   StoreStore + StoreLoad   (the expensive one — full fence)
   volatile read    ≈   LoadLoad  + LoadStore
   final field write (end of constructor) ≈ StoreStore
   lock acquire      ≈   LoadLoad + LoadStore on entry; StoreStore + StoreLoad on release
```

`StoreLoad` is the most expensive barrier — it forces the CPU to drain its **store buffer** before issuing any
subsequent loads. This is why `volatile` writes are pricier than `volatile` reads. JIT is allowed to *elide* fences
when it can prove no other thread can observe the difference.

---

## 5. Practical Consequences

### 5.1 Safe publication

To **safely publish** an object to another thread (publish = make a reference visible), at least one of:

- Initialise it from a static initialiser (`static final`).
- Store the reference into a `volatile` field or `AtomicReference`.
- Store it into a field guarded by a lock that the reader also acquires.
- Put it into a `BlockingQueue`, `ConcurrentHashMap`, or hand it to an `Executor`.

Without safe publication, the reader may see a **half-constructed** object: non-null reference, but with default
field values (0, null) instead of the constructor's writes.

### 5.2 Final fields and initialization safety

> `final` fields get a special guarantee: once the constructor completes, every thread that sees a reference to the
> object will see the **correctly constructed** values of its `final` fields — even without synchronization.

```java
class ImmutablePoint {
    private final int x, y;            // final → safely publishable

    ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public int x() { return x; }
    public int y() { return y; }
}

ImmutablePoint p = new ImmutablePoint(1, 2);
sharedRef = p;                          // even unsynchronized, any reader seeing p
                                        // will see x=1, y=2
```

This is the foundation of **immutability = thread safety** in Java. Rules:
- All fields must be `final`.
- The object must not **escape** the constructor (no `this`-publication).
- For mutable sub-objects (arrays, collections) immutability is more subtle — `final` only protects the reference.

### 5.3 Double-checked locking — finally correct

The classic lazy singleton, broken before Java 5:

```java
class Singleton {
    private static Singleton instance;              // BAD — without volatile
    public static Singleton getInstance() {
        if (instance == null) {                     // (1) outer check — no lock
            synchronized (Singleton.class) {
                if (instance == null) {             // (2) inner check
                    instance = new Singleton();     // (3) construction + assignment
                }
            }
        }
        return instance;
    }
}
```

Why broken? Step (3) is **two** operations: construct the object, then assign the reference. Without happens-before,
the compiler/CPU may reorder them so the **reference is assigned first**, then the constructor runs. Another thread
passing check (1) sees a non-null reference but reads uninitialised fields.

Fix — make the field `volatile`:

```java
private static volatile Singleton instance;
```

The `volatile` write at (3) establishes a StoreStore barrier preventing the construction from being reordered
after the assignment, and every reader's `volatile` read forms the happens-before edge. (Before JLS §17 revision
in JSR-133 / Java 5, even `volatile` did not reliably fix DCL.)

### 5.4 `String` and immutable value types

`String`, `Integer`, `LocalDate` and friends are safely publishable **without synchronization** precisely because
their fields are `final` — the JMM gives them initialization-safety for free.

### 5.5 `ConcurrentHashMap` and safe iteration

`ConcurrentHashMap`'s iterator is **weakly consistent**:
- It reflects some state of the map at or since the iterator's creation.
- It never throws `ConcurrentModificationException`.
- Operations on the map establish happens-before edges with the iterator's traversal.

You still need to use the API's atomic methods (`putIfAbsent`, `compute`) for compound actions — happens-before
gives you visibility, not atomicity.

---

## 6. Common Pitfalls

### 6.1 "It works on my machine"

The absence of happens-before edges means **anything** can happen — and "anything" includes "looks fine locally".
You may never see the bug until production runs a different CPU, JIT optimisation, or timing. Reproducing JMM bugs
is notoriously hard. Use tools like **JCStress** (`org.openjdk.jcstress`) to stress-test your assumptions.

### 6.2 Forgetting the same-monitor / same-field qualifier

```java
synchronized (lockA) { x = 1; }       // releases lockA
synchronized (lockB) { int y = x; }   // locks lockB — NO hb with the above
```

Different locks, different variables — no guarantee.

### 6.3 Assuming atomicity from visibility

```java
volatile int counter = 0;
// counter++ is NOT atomic — read, add, write — three steps
```

`volatile` gives visibility + ordering but **not** compound atomicity. Use `AtomicInteger` or `synchronized`.

### 6.4 `this`-escape from a constructor

```java
class Bad {
    Bad(EventBus bus) {
        bus.register(this);           // publishes `this` before constructor finishes
    }
}
```

Other threads may observe `this` via the bus before its constructor returns — and the final-field guarantee no
longer applies. Always register/launch **after** construction.

### 6.5 Infinite loops without volatile

```java
boolean running = true;
while (running) { ... }   // JIT may hoist `running` to a register → never sees the update
```

`running` must be `volatile` (or guarded by a lock) for another thread's `running = false` to be observed.

### 6.6 Believing `System.out.println` is the bug

`PrintStream` is internally `synchronized`. Adding `println` "fixes" a race by accident — the lock establishes
happens-before. Remove it, and the bug returns. Don't reason from "added a print statement and it works".

---

## 7. Decision Matrix — Which Primitive Establishes happens-before?

| You need                                | Use                                              |
|-----------------------------------------|--------------------------------------------------|
| One flag, only visibility               | `volatile boolean`                               |
| Atomic counter / accumulator            | `AtomicInteger`, `LongAdder`                     |
| Single published reference              | `volatile`, `AtomicReference`                    |
| Mutual exclusion + visibility           | `synchronized`, `ReentrantLock`                  |
| Cross-thread handshake / wait           | `CountDownLatch`, `Phaser`, `CyclicBarrier`      |
| Producer/consumer                       | `BlockingQueue` (`ArrayBlockingQueue`, etc.)     |
| Async result                            | `Future`, `CompletableFuture`                    |
| Thread-safe collection                  | `ConcurrentHashMap`, `CopyOnWriteArrayList`      |
| Immutable data                          | `final` fields (initialization safety)           |

> Modern Java code rarely needs raw `volatile` or `synchronized` — the higher-level `java.util.concurrent` APIs
> both establish happens-before and avoid common mistakes.

---

## 8. Worked Example — Putting It Together

```java
class WorkerPool {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>();
    private volatile boolean shutdown = false;

    private final ExecutorService workers = Executors.newFixedThreadPool(4);

    WorkerPool() {
        for (int i = 0; i < 4; i++) {
            workers.submit(() -> {
                while (!shutdown) {                       // volatile read
                    try {
                        Task t = queue.poll(100, MILLISECONDS);
                        if (t != null) t.run();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
    }

    void submit(Task t) {
        if (shutdown) throw new IllegalStateException();
        queue.put(t);                                      // hb: put → take by worker
    }

    void shutdown() {
        shutdown = true;                                   // volatile write
        workers.shutdown();                                // also creates hb with worker tasks
    }
}
```

Chain of happens-before edges for a submitted task:
- `submit()` `queue.put(t)` — worker's `queue.poll()` returns `t` (BlockingQueue hb rule).
- `shutdown()` writes `shutdown = true` — workers' `while (!shutdown)` sees it (volatile rule).
- `workers.shutdown()` — workers' tasks observe it (ExecutorService rule).

No raw `synchronized`, no explicit `volatile` except the flag — `j.u.c.` does the rest.

---

## 9. Common Interview Questions

1. **What is the happens-before relationship?**
   A partial order over actions defined by the JMM. If A happens-before B, then A's memory effects are visible to
   B and the two are ordered (no compiler/CPU may reorder them in a way B can observe).

2. **List the rules that establish happens-before.**
   Single-thread program order; monitor lock unlock → subsequent lock on the same monitor; volatile write →
   subsequent volatile read of the same field; `Thread.start()` → actions in started thread; actions in thread →
   `join()` returning in another thread; `interrupt()` → detection; constructor end → finalizer start; and
   transitivity.

3. **Why does double-checked locking need `volatile`?**
   Without it, the assignment `instance = new Singleton()` can be reordered so that the reference is published
   *before* the constructor runs. Another thread sees a non-null reference and reads uninitialised fields. The
   `volatile` field's StoreStore barrier prevents the reordering and the volatile read on the other side establishes
   the happens-before edge.

4. **What does `volatile` guarantee and not guarantee?**
   Guarantees: visibility of writes, ordering relative to other volatile operations, atomicity of a single
   read/write. Does **not** guarantee atomicity of compound operations (`++`, check-then-act) and does **not**
   provide mutual exclusion.

5. **Why are `final` fields special in the JMM?**
   They have initialization safety: once the constructor completes, any thread that obtains a reference to the
   object sees the correctly constructed values of the `final` fields — without any synchronization. This is the
   basis of immutable = thread-safe.

6. **What is a data race?**
   When two threads access the same non-final field, at least one of them writes, and there is no happens-before
   edge between the accesses. The JMM provides no guarantee — any value (including "impossible" ones) may be
   observed.

7. **`synchronized` vs `volatile` — what's the difference?**
   `synchronized` provides **mutual exclusion AND visibility** (atomicity for the block as a whole). `volatile`
   provides **only visibility + ordering** for a single field — no mutual exclusion, no compound atomicity.
   `synchronized` is heavier; `volatile` is lighter but narrower.

8. **Why is `ConcurrentHashMap`'s iterator "weakly consistent"?**
   It does not throw `ConcurrentModificationException`, reflects the map's state at or since creation, and sees
   each element at most once. Operations on the map establish happens-before edges with the iterator, but the
   iterator does **not** provide a consistent snapshot.

9. **What is "safe publication"?**
   Making an object reference visible to other threads in a way that guarantees they also see the object's
   constructed state. Mechanisms: `static final`, `volatile`/`AtomicReference`, locks, `BlockingQueue`, executors,
   or `final` fields.

10. **Why does `BlockingQueue.put()` happen-before `take()`?**
    Because `BlockingQueue` is a `j.u.c.` synchronizer and its documentation promises this happens-before edge. It
    uses internal locks/`volatile`/CAS operations to establish the relationship. The library does the work so you
    don't have to.

11. **What is JCStress and when do you use it?**
    The Java Concurrency Stress tests — an OpenJDK harness for verifying concurrency invariants under heavy load.
    Use it to test custom lock-free algorithms and confirm happens-before assumptions that are hard to reproduce
    in normal unit tests.

12. **Can `a = 1; b = 2;` in one thread be reordered?**
    Yes — by the compiler or CPU — **if nothing else observes them**. The as-if-serial rule allows any
    reordering that doesn't change single-threaded semantics. happens-before rules constrain cross-thread
    observability; within a single thread, you can't tell.

13. **Why is `StoreLoad` the most expensive memory barrier?**
    It requires draining the store buffer before any subsequent load can be issued, which can cost tens of
    nanoseconds and stall the pipeline. That's why `volatile` writes (which emit a full fence including StoreLoad)
    are pricier than `volatile` reads.

14. **If `Thread A` starts `Thread B`, do `A`'s writes become visible to `B`?**
    Yes. The `Thread.start()` rule: every action before `start()` happens-before any action in the started thread.
    No extra synchronisation is needed to pass data to a freshly started thread.

---

## 10. Mental Cheat-Sheet

> **happens-before = visibility + ordering, via transitivity over program order and synchronizes-with.**
>
> To answer any "is this thread-safe?" question:
> 1. Find the writes that must be visible.
> 2. Trace a happens-before path to the reader (program order ➜ sync action ➜ program order).
> 3. If no path exists → there's a data race → add a synchronizer.

```
   PROGRAM ORDER           SYNCHRONIZES-WITH           PROGRAM ORDER
   (inside a thread)   ┌─────────────────────────┐    (inside a thread)
                       │                         │
   writer thread ─po─▶ │ volatile write          │ ─po─▶ reader action
                       │  ────────────sw─────────│
                       │ volatile read           │
                       │  ────────────sw─────────│
                       │ monitor unlock / lock   │
                       │  ────────────sw─────────│
                       │ Thread.start / run      │
                       │  ────────────sw─────────│
                       │ Thread.run / join       │
                       │  ────────────sw─────────│
                       │ j.u.c. pairs            │
                       └─────────────────────────┘

   hb = transitive closure of (po ∪ sw)
```

### Related Files

- `Java/concurrency/synchronized-and-volatile.md` — JMM primer, the two primitives in depth
- `Java/concurrency/locks-and-atomic.md` — `Lock`, `Atomic*`, `Condition`
- `Java/concurrency/concurrent-collections.md` — `ConcurrentHashMap`, `CopyOnWrite*`, `BlockingQueue`
- `Java/concurrency/synchronization-primitives.md` — `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`
- `Java/concurrency/thread-fundamentals.md` — thread lifecycle, `start()`/`join()` semantics
