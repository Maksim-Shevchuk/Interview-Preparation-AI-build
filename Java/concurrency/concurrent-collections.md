# Concurrent Collections

Thread-safe collections from `java.util.concurrent` — the building blocks of real concurrent systems. Interviewers
expect you to know **ConcurrentHashMap internals**, **BlockingQueue** variants, and when to use which.

## Why Not `Collections.synchronized*`?

```java
// ❌ Synchronized wrapper — entire map locked on every operation
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());

// ❌ Still NOT thread-safe for compound operations (check-then-act)
synchronized (map) {
    if (!map.containsKey(key)) {
        map.put(key, computeValue()); // must lock externally
    }
}
```

`synchronizedMap` wraps every method with `synchronized(mutex)` — one global lock, poor concurrency. No atomic
compound operations.

## ConcurrentHashMap

The most important concurrent collection. Lock-free reads, fine-grained locking for writes.

### Internal Structure (Java 8+)

```
ConcurrentHashMap
├── Node<K,V>[] table           // bucket array (power of 2)
├── Segmented locking            // lock per bucket (CAS + synchronized on node)
├── Treeification                // like HashMap: list → red-black tree at threshold 8
└── Size tracking                // LongAdder-style distributed counters
```

**Java 7 and earlier:** Array of `Segment` objects, each with its own `ReentrantLock`. Default 16 segments = 16
concurrent writers max.

**Java 8+:** Segments removed. Uses **CAS on the first node** of each bucket for insertion, **synchronized on the
node** for updates within a bucket. Much finer granularity.

### Thread-Safety Guarantees

- **Individual operations** (`get`, `put`, `remove`) are atomic.
- **Iteration is weakly consistent** — reflects the state at some point during or after the iterator's creation. No
  `ConcurrentModificationException`. May or may not reflect concurrent modifications.
- **`size()` is approximate** — uses distributed counters (like `LongAdder`). Exact count requires traversal. Don't
  use `size()` for synchronization logic.

### Atomic Compound Operations

The killer feature — **check-then-act atomically**, no external locking needed:

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// putIfAbsent — atomic check + insert
map.putIfAbsent("key", 1); // inserts only if "key" is absent

// computeIfAbsent — lazy computation, atomic
map.computeIfAbsent("key", k -> expensiveCompute(k));

// computeIfPresent — update only if exists
map.computeIfPresent("key", (k, v) -> v + 1);

// compute — update or insert
map.compute("key", (k, v) -> v == null ? 1 : v + 1);

// merge — combine old and new value
map.merge("key", 1, Integer::sum); // increment counter, atomic

// replaceAll — transform all values
map.replaceAll((k, v) -> v * 2);
```

### Bulk Operations (Java 8+)

Parallel stream-like operations that exploit internal parallelism:

```java
// forEach with parallelism threshold
// (if map has > 10 elements, execute in parallel using ForkJoinPool)
map.forEach(10, (key, value) -> process(key, value));

// search — find first match, short-circuits
String found = map.search(10, (key, value) -> value > 100 ? key : null);

// reduce
int sum = map.reduce(10, (key, value) -> value, Integer::sum);
```

The first argument is the **parallelism threshold**: if the map size exceeds this, the operation runs in parallel.
Use `1` to always parallelize, `Long.MAX_VALUE` to never.

### Common Pitfalls

```java
// ❌ NOT atomic — check and put are separate operations
if (!map.containsKey(key)) {
    map.put(key, value); // another thread may put between check and put
}

// ✅ Atomic
map.putIfAbsent(key, value);

// ❌ null keys and values are NOT allowed (unlike HashMap)
map.put("key", null); // NullPointerException

// ❌ Iterating and modifying — weakly consistent, may miss updates
for (Map.Entry<String, Integer> e : map.entrySet()) {
    if (e.getValue() == 0) map.remove(e.getKey()); // works, but may miss concurrent additions
}
```

## BlockingQueue

A queue that **blocks** when trying to take from an empty queue or put into a full queue. The backbone of
producer-consumer patterns.

### API — Four Sets of Methods

| Operation | Throws exception | Returns special value | Blocks           | Times out                  |
|-----------|------------------|-----------------------|------------------|----------------------------|
| Insert    | `add(e)`         | `offer(e)` → `false`  | `put(e)`         | `offer(e, time, unit)`     |
| Remove    | `remove()`       | `poll()` → `null`     | `take()`         | `poll(time, unit)`         |
| Examine   | `element()`      | `peek()` → `null`     | N/A              | N/A                        |

**`put()`/`take()`** are the most commonly used — they block until space/element is available.

### Implementations

#### `ArrayBlockingQueue`

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100); // bounded, capacity = 100

// Producer
queue.put(task); // blocks if queue is full

// Consumer
Task task = queue.take(); // blocks if queue is empty
```

- **Bounded** (fixed capacity, specified at creation).
- **Backed by array** — pre-allocated, no GC pressure.
- **Single lock** for both put and take (fair or unfair).
- **Best for:** Known capacity, FIFO ordering, general producer-consumer.

#### `LinkedBlockingQueue`

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000); // bounded
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();     // unbounded (Integer.MAX_VALUE)
```

- **Optionally bounded** (default: `Integer.MAX_VALUE` = effectively unbounded → OOM risk).
- **Two locks** — separate for head (take) and tail (put) → higher throughput under contention.
- **Node-based** — allocates a node per element (more GC).
- **Best for:** Higher throughput than `ArrayBlockingQueue` when producers and consumers are balanced.

#### `PriorityBlockingQueue`

```java
BlockingQueue<Task> queue = new PriorityBlockingQueue<>(11, Comparator.comparing(Task::priority));
```

- **Unbounded** (grows dynamically).
- **Sorted by priority** (natural order or `Comparator`).
- **`take()` blocks** if empty, but `put()` never blocks (unbounded).
- **Best for:** Task scheduling by priority.

#### `SynchronousQueue`

```java
BlockingQueue<Task> queue = new SynchronousQueue<>();
```

- **Zero capacity** — no internal storage. Every `put()` blocks until another thread calls `take()` (and vice versa).
- **Direct handoff** — producer hands directly to consumer.
- **Used by:** `Executors.newCachedThreadPool()` — every task gets a new thread if none is available.

#### `DelayQueue`

```java
class DelayedTask implements Delayed {
    private final long executeAt;

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }
    // ...
}

DelayQueue<DelayedTask> queue = new DelayQueue<>();
```

- **Unbounded** — elements implement `Delayed`.
- **`take()` blocks** until the element's delay has expired.
- **Best for:** Scheduling, retry with backoff, TTL-based expiration.

### Comparison

| Implementation           | Bounded | Ordering   | Locks     | Best for                     |
|--------------------------|---------|------------|-----------|------------------------------|
| `ArrayBlockingQueue`     | ✅ Yes  | FIFO       | 1 lock    | General producer-consumer    |
| `LinkedBlockingQueue`    | Optional| FIFO       | 2 locks   | High-throughput P-C          |
| `PriorityBlockingQueue`  | ❌ No   | Priority   | 1 lock    | Priority scheduling          |
| `SynchronousQueue`       | ✅ (0)  | N/A        | Lock-free | Direct handoff (CachedPool) |
| `DelayQueue`             | ❌ No   | By delay   | 1 lock    | Delayed/scheduled tasks      |

## CopyOnWriteArrayList / CopyOnWriteArraySet

Every **write** (add, set, remove) creates a **new copy** of the underlying array. Reads are never blocked and always
consistent.

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("a"); // copies entire array + adds element
list.get(0);   // reads from current snapshot, no lock

// Safe iteration — snapshot at iterator creation time
for (String s : list) {
    list.add("b"); // modifies a DIFFERENT copy — iterator is not affected, no CME
}
```

### Trade-offs

| Aspect     | `CopyOnWriteArrayList`              | `synchronizedList`               |
|------------|-------------------------------------|----------------------------------|
| Reads      | ✅ Lock-free, very fast             | ❌ Synchronized                  |
| Writes     | ❌ Expensive (full array copy)      | ✅ In-place, fast                |
| Iteration  | ✅ Snapshot, no CME                 | ❌ Must lock externally          |
| Memory     | ❌ O(N) per write                   | ✅ In-place                      |

**Use when:** Reads far outnumber writes. Small lists. Event listeners/observers (rarely modified, frequently iterated).

**Don't use when:** Frequent writes, large lists — O(N) copy per write.

## ConcurrentLinkedQueue / ConcurrentLinkedDeque

Lock-free, unbounded, non-blocking queue based on CAS:

```java
ConcurrentLinkedQueue<Task> queue = new ConcurrentLinkedQueue<>();
queue.offer(task);    // never blocks, always succeeds
Task t = queue.poll(); // returns null if empty, never blocks
```

- **No blocking operations** — unlike `BlockingQueue`, `poll()` returns `null` instead of blocking.
- **Lock-free** — uses CAS internally (Michael & Scott algorithm).
- **`size()` is O(n)** — traverses the list (don't use in loops).
- **Best for:** Non-blocking producer-consumer, work-stealing queues.

## ConcurrentSkipListMap / ConcurrentSkipListSet

Concurrent sorted map/set (like `TreeMap` but thread-safe):

```java
ConcurrentSkipListMap<String, Integer> map = new ConcurrentSkipListMap<>();
map.put("banana", 2);
map.put("apple", 3);
map.firstKey(); // "apple" — sorted
```

- **O(log n)** for get/put/remove.
- **Lock-free** — CAS-based skip list.
- **Sorted iteration** — unlike `ConcurrentHashMap`.
- **Best for:** Concurrent sorted access, range queries.

## Choosing the Right Collection

| Need                                  | Collection                              |
|---------------------------------------|-----------------------------------------|
| Thread-safe key-value, high throughput| `ConcurrentHashMap`                     |
| Thread-safe sorted map                | `ConcurrentSkipListMap`                 |
| Producer-consumer (blocking)          | `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| Read-heavy, rarely written list       | `CopyOnWriteArrayList`                  |
| Non-blocking queue                    | `ConcurrentLinkedQueue`                 |
| Priority-based task queue             | `PriorityBlockingQueue`                 |
| Direct handoff (zero-buffer)          | `SynchronousQueue`                      |
| Delayed/scheduled tasks               | `DelayQueue`                            |

## Common Interview Questions

1. **How does `ConcurrentHashMap` work internally (Java 8+)?** — Bucket array with CAS on first node for insert,
   `synchronized` on the node for updates within a bucket. No global lock. Reads are lock-free.
   Treeification at threshold 8 (like `HashMap`).
2. **`ConcurrentHashMap` vs `synchronizedMap`?** — `synchronizedMap` uses one global lock (poor concurrency).
   `ConcurrentHashMap` uses per-bucket locking (high concurrency) and provides atomic compound operations
   (`computeIfAbsent`, `merge`).
3. **Can `ConcurrentHashMap` have `null` keys or values?** — No. Both throw `NullPointerException`. This is by
   design — `null` is ambiguous (absent vs. present with null value) and unsafe in concurrent context.
4. **What is `BlockingQueue`? Name implementations.** — A queue that blocks on `take()` if empty and `put()` if full.
   Implementations: `ArrayBlockingQueue` (bounded, single lock), `LinkedBlockingQueue` (optionally bounded, two
   locks), `PriorityBlockingQueue` (unbounded, sorted), `SynchronousQueue` (zero capacity, direct handoff).
5. **`CopyOnWriteArrayList` — when to use?** — When reads vastly outnumber writes and the list is small. Every write
   copies the entire array. Iteration is snapshot-based — no `ConcurrentModificationException`.
6. **`computeIfAbsent` vs `putIfAbsent`?** — `putIfAbsent(key, value)` always evaluates the value (even if key
   exists). `computeIfAbsent(key, fn)` only calls `fn` if the key is absent — lazy computation, more efficient.

## Related

- [HashMap Internals](../core/collections-hashmap-internals.md) — non-concurrent HashMap deep dive
- [Executors and Thread Pools](./executors-and-thread-pools.md) — BlockingQueue as task queue in ThreadPoolExecutor
- [Locks and Atomic](./locks-and-atomic.md) — CAS mechanism used in concurrent collections

## Resources

- Brian Goetz — *Java Concurrency in Practice* (chapter 5: Building Blocks)
- [JDK Docs — java.util.concurrent](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Baeldung — ConcurrentHashMap](https://www.baeldung.com/java-concurrent-map)
