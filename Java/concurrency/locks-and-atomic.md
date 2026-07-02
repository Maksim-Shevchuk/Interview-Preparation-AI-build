# Locks and Atomic

Beyond `synchronized`: explicit `Lock` implementations and lock-free atomic variables. Asked at Middle+ interviews,
especially for high-concurrency code.

## The Lock Interface

`java.util.concurrent.locks.Lock` is the explicit-lock counterpart to `synchronized`. Provides:

- `lock()` — block until acquired
- `lockInterruptibly()` — block, but throw `InterruptedException` if interrupted
- `tryLock()` — non-blocking; returns immediately (success or fail)
- `tryLock(timeout, unit)` — block up to timeout
- `unlock()` — release
- `newCondition()` — create a `Condition` (replacement for `wait/notify`)

```java
Lock lock = new ReentrantLock();
lock.lock();
try{
        // critical section
} finally {
        lock.unlock();   // MUST be in finally — locks don't auto-release
}
```

## ReentrantLock

The most-used `Lock` implementation. Reentrant (same thread can re-acquire; counter). Constructor accepts a fairness
flag:

```java
ReentrantLock fair = new ReentrantLock(true);   // fair: longest-waiting thread gets it
ReentrantLock unfair = new ReentrantLock(false); // default: scheduler picks (allows barging)
```

**Fair vs unfair:**

- Fair — strict FIFO; reduces starvation but adds context-switch overhead.
- Unfair (default) — barging allowed; better throughput, possible starvation.

### tryLock Pattern

Non-blocking acquisition — useful for avoiding deadlocks:

```java
if(lock.tryLock()){
    try { /* got it */ }
    finally{ 
        lock.unlock();
    }
} else {
    // didn't get it, do something else
}
```

Timed variant for deadlock avoidance:

```java
if(lock1.tryLock(1,SECONDS) && lock2.tryLock(1,SECONDS)){
        try { /* both held */ }
        finally {
            lock1.unlock(); 
            lock2.unlock();
        }
}
```

## synchronized vs ReentrantLock

| Feature                   | synchronized            | ReentrantLock                   |
|---------------------------|-------------------------|---------------------------------|
| Try-lock                  | no                      | yes (`tryLock`)                 |
| Interruptible             | no                      | yes (`lockInterruptibly`)       |
| Fairness                  | no                      | yes (optional)                  |
| Multiple conditions       | 1 (`wait/notify`)       | yes (multiple `Condition`)      |
| Auto-release on exit      | yes                     | no — must `unlock` in `finally` |
| Read/write separation     | no                      | use `ReentrantReadWriteLock`    |
| Performance (uncontended) | biased/thin locks, fast | similar                         |
| Performance (contended)   | JVM-tuned               | similar                         |

**Rule of thumb:** use `synchronized` for simple cases; use `ReentrantLock` when you need `tryLock`, interruptibility,
fairness, or multiple conditions.

## ReentrantReadWriteLock

Separates read and write locks:

- Multiple readers can hold the read lock simultaneously.
- Only one writer; no readers while writing.

```java
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();   // shared
rw.writeLock().lock();  // exclusive
```

**Caveats:**

- Write lock can downgrade to read lock (acquire write, then read, then release write).
- Read lock CANNOT upgrade to write lock — would deadlock.
- Under heavy read load, writers can starve (use fair mode).

## StampedLock

Java 8+. Adds **optimistic reads** — read without acquiring a lock, validate at the end:

```java
StampedLock lock = new StampedLock();

double distanceFromOrigin() {
    long stamp = lock.tryOptimisticRead();      // optimistic (no lock)
    double x = currentX, y = currentY;          // snapshot
    if (!lock.validate(stamp)) {                 // did a write happen?
        stamp = lock.readLock();                 // fallback to pessimistic
        try {
            x = currentX;
            y = currentY;
        } finally {
            lock.unlockRead(stamp);
        }
    }
    return Math.sqrt(x * x + y * y);
}
```

**Pros:** very fast for read-heavy workloads.
**Cons:** not reentrant; tricky API; can starve writers if misused.

## Condition

Replaces `Object.wait/notify/notifyAll` with multiple wait-sets per lock:

```java
Lock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

public void put(E e) throws InterruptedException {
    lock.lock();
    try {
        while (queue.size() == capacity) notFull.await();
        queue.add(e);
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
}
```

`signal` vs `signalAll` — same as `notify` vs `notifyAll`: signal one or all waiters.

## Atomic Variables

`java.util.concurrent.atomic` — lock-free primitives built on CAS (Compare-And-Swap):

| Class                          | Use                              |
|--------------------------------|----------------------------------|
| `AtomicInteger` / `AtomicLong` | Counters, sequence numbers       |
| `AtomicBoolean`                | One-shot flags                   |
| `AtomicReference<V>`           | Safe publication of mutable refs |
| `AtomicStampedReference<V>`    | ABA protection                   |
| `LongAdder` / `DoubleAdder`    | High-contention counters         |
| `AtomicIntegerArray` etc.      | Per-element atomic arrays        |

```java
AtomicInteger counter = new AtomicInteger();
counter.

incrementAndGet();   // ++counter (atomic)
counter.

compareAndSet(0,1); // if counter == 0, set to 1 (returns success)
```

## CAS — Compare-And-Swap

Hardware instruction (x86: `cmpxchg`, ARM: `LDREX/STREX`). Atomically:

1. Read current value.
2. If equals expected → write new value, return success.
3. Else → return failure (retry or give up).

Java exposes CAS via `Unsafe` (legacy), `VarHandle` (Java 9+), and the `Atomic*` classes.

**ABA problem:** thread reads A, another thread changes A→B→A, CAS succeeds but the intermediate change is missed.
Solutions: versioned references (`AtomicStampedReference`), or accept it if only the final state matters.

## LongAdder — High-Contention Counters

Under heavy contention, `AtomicLong` spends CPU on CAS retries. `LongAdder` maintains a cell array (one per stripe) and
sums them on read:

```java
LongAdder counter = new LongAdder();
counter.increment();    // writes to a cell, not the shared field
counter.sum();          // sum of all cells
```

Trade-off: faster writes, slower reads, higher memory. Perfect for metrics / counters read rarely.

## Common Pitfalls

1. **Forgetting `unlock` in `finally`** — deadlock on exception.
2. **Locking in wrong order across methods** — classic deadlock.
3. **Read-lock upgrade** — `ReentrantReadWriteLock` read → write is a self-deadlock.
4. **Using `synchronized` + `Condition`** — `Condition` requires a `Lock`, not the intrinsic monitor.
5. **`tryLock` without checking the result** — returns false instantly; check before proceeding.
6. **Atomicity confusion** — `AtomicReference` makes the reference atomic, not the object's fields.
7. **`LongAdder` for low-contention** — overhead exceeds benefit; use `AtomicLong`.
8. **`StampedLock` is not reentrant** — recursive acquire deadlocks.

## Code Examples

### Thread-safe counter with ReentrantLock

```java
public class Counter {
    private final Lock lock = new ReentrantLock();
    private int count;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

### Atomic counter

```java
AtomicInteger counter = new AtomicInteger();

public void increment() {
    counter.incrementAndGet();
}
```

### Bounded buffer with Condition

```java
public class BoundedBuffer<T> {
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) notFull.await();
            queue.add(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await();
            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### StampedLock optimistic read

```java
public class Point {
    private final StampedLock lock = new StampedLock();
    private double x, y;

    public void move(double dx, double dy) {
        long stamp = lock.writeLock();
        try {
            x += dx;
            y += dy;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double x = this.x, y = this.y;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                x = this.x;
                y = this.y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(x * x + y * y);
    }
}
```

## Common Interview Questions

1. **What's the difference between synchronized and ReentrantLock?**
   → ReentrantLock adds `tryLock`, `lockInterruptibly`, fairness, multiple `Condition`s. Trade-off: must explicitly
   `unlock` in `finally`.

2. **What is CAS?**
   → Compare-And-Swap: atomic CPU instruction that updates a value only if it still equals the expected value.
   Foundation of lock-free programming.

3. **What is the ABA problem?**
   → A thread reads A, another changes A→B→A; CAS succeeds but the intermediate change is missed. Solution: versioned
   references (`AtomicStampedReference`).

4. **When would you use ReentrantReadWriteLock?**
   → Read-heavy workloads where reads dominate writes. Multiple readers can hold simultaneously; writers are exclusive.

5. **What is StampedLock's optimistic read?**
   → Acquire a stamp without locking; read the data; validate at the end. If a write happened meanwhile, fall back to a
   pessimistic read lock. Very fast for read-mostly workloads.

6. **Why use LongAdder instead of AtomicLong?**
   → Under high contention, AtomicLong burns CPU on CAS retries. LongAdder stripes writes across cells and sums on
   read — faster writes, slower reads.

7. **Can a read lock be upgraded to a write lock?**
   → No — self-deadlock. Write lock can downgrade to read (acquire write, then read, release write), but not vice versa.

8. **What is the Condition interface?**
   → Multiple wait-sets per `Lock`. Replaces `wait/notify`, which only allows one wait-set per object. Use
   `lock.newCondition()`.

9. **What's the difference between fair and unfair ReentrantLock?**
   → Fair: longest-waiting thread acquires next (FIFO). Unfair: barging allowed; better throughput, possible starvation.
   Default is unfair.

10. **Is `AtomicReference` enough for thread-safe mutable state?**
    → No — only the reference is atomic. The object's fields are not. Either make the object immutable, or use
    additional synchronization.

## Related

- `Java/concurrency/synchronized-and-volatile.md` — intrinsic locks, JMM
- `Java/concurrency/executors-and-thread-pools.md` — thread pools
- `Java/concurrency/completable-future.md` — async composition
- `Java/core/collections-hashmap-internals.md` — `ConcurrentHashMap` uses CAS

## Resources

- **Brian Goetz, "Java Concurrency in Practice"** — Lock, Condition, atomic
- **Java docs:** `java.util.concurrent.locks`, `java.util.concurrent.atomic`
- **"The Art of Multiprocessor Programming" (M. Herlihy)** — lock-free theory
- **Aleksey Shipilëv:** https://shipilev.net/ — JMM, CAS
