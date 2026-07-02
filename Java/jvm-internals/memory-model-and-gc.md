# JVM Memory Model and Garbage Collection

Foundational topic for Middle+ Java interviews. Covers memory regions, heap generations, GC algorithms, and choosing the
right collector.

## JVM Memory Areas

```
┌────────────────────────────────────────────────────────┐
│ JVM Process                                            │
│                                                        │
│  ┌─────────────────┐  ┌────────────────────────────┐   │
│  │ Heap            │  │ Metaspace (Java 8+, was    │   │
│  │  - Young gen    │  │ PermGen before)            │   │
│  │  - Old gen      │  │  - Class metadata          │   │
│  │  - (regions in  │  │  - Method area is a        │   │
│  │    G1)          │  │    logical part of this    │   │
│  └─────────────────┘  └────────────────────────────┘   │
│                                                        │
│  Per-thread (not shared, not GC'd):                    │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐    │
│  │ JVM Stack    │  │ Native Stack │  │ PC Register│    │
│  │ (frames,     │  │ (JNI calls)  │  │(per-thread │    │
│  │  locals)     │  │              │  │ instr ptr) │    │
│  └──────────────┘  └──────────────┘  └────────────┘    │
└────────────────────────────────────────────────────────┘
```

### Heap

Shared across all threads. Holds objects and arrays. **Garbage collected.** Default size auto-tuned by ergonomics; set
with `-Xms` (initial) and `-Xmx` (max).

### Metaspace (was PermGen)

Class metadata, method bytecode, constant pool, static fields. Native memory (off-heap), grows as needed. Replaced
`PermGen` in Java 8 (PermGen had a fixed max and caused `OOM: PermGen space`).

### JVM Stack

Per-thread. Holds **stack frames** — one per method call. Each frame has:

- Local variables array
- Operand stack
- Reference to runtime constant pool

Stack size `-Xss` (default 512KB–1MB). `StackOverflowError` on deep recursion.

### Native Method Stack

For JNI calls. Same lifecycle as JVM Stack.

### PC Register

Per-thread, points to the current JVM instruction. Always one per thread.

## Heap Generations

Most JVM GCs use a **generational** design based on the **generational hypothesis**: most objects die young.

```
Heap
├── Young Generation
│   ├── Eden (new allocations)
│   ├── Survivor 0 (S0)
│   └── Survivor 1 (S1)
│       Ratio: Eden:S0:S1 = 8:1:1 (default)
└── Old Generation (Tenured)
    └── Long-lived objects, promoted from Young
```

Plus, **G1** uses a different layout (region-based, see below).

### Object Lifecycle

1. Allocated in **Eden** (fast, bump-pointer allocation).
2. Minor GC: live objects copied to a Survivor space. Eden is cleared.
3. After surviving N minor GCs (default 15, `MaxTenuringThreshold`), promoted to Old.
4. Large objects (humongous in G1) bypass Eden, go directly to Old.
5. Old gen collected by Major GC / Full GC.

## GC Roots

GC starts from **roots** and traces live objects. Anything not reachable is garbage.

GC roots include:

- Local variables in active stack frames
- Active Java threads
- Static fields
- JNI global references
- Synchronized monitors held
- Internal JVM references (system class loader, etc.)

## GC Algorithms

### Mark-Sweep

1. **Mark** all reachable objects (DFS from roots).
2. **Sweep** unmarked objects.

Fragmentation is the issue — leads to allocation failures requiring compaction.

### Mark-Sweep-Compact

Adds a **compact** phase — moves live objects together. No fragmentation, but pause time scales with heap size.

### Copying

Live objects copied from one region to another. No fragmentation, fast allocation. But uses 2x memory and pause time
scales with live object count. Used for Young gen (copy Eden + one Survivor → the other Survivor).

### Generational

Combine: Young gen uses **copying** (most objects die → few live to copy), Old gen uses **mark-sweep-compact**.

## STW Pauses

Most GCs have **Stop-The-World** pauses where application threads are paused. Reducing pause time is the primary GC
engineering goal.

## GC Collectors

| Collector                 | Type                            | Java                     | Use case                               |
|---------------------------|---------------------------------|--------------------------|----------------------------------------|
| **Serial**                | Single-threaded                 | all                      | Small heaps, client apps               |
| **Parallel** (Throughput) | Multi-threaded STW              | Java 8 default           | Batch jobs, throughput-focused         |
| **CMS**                   | Mostly concurrent               | Removed in Java 14       | (deprecated) Low-pause                 |
| **G1**                    | Region-based, mostly concurrent | Java 9+ default          | Large heaps, balanced pause/throughput |
| **ZGC**                   | Concurrent, <10ms pauses        | Java 15 production       | Very large heaps, ultra-low pause      |
| **Shenandoah**            | Concurrent, <10ms pauses        | Java 12+ (Red Hat build) | Similar goals to ZGC                   |

### Serial GC

`-XX:+UseSerialGC`. Single-threaded everything. For <100MB heaps.

### Parallel GC

`-XX:+UseParallelGC`. Java 8 default. Multiple GC threads, all STW. Maximizes throughput at the cost of pause time. Good
for batch processing.

### G1 (Garbage-First)

`-XX:+UseG1GC`. Default since Java 9.

**Layout:** heap divided into equal-sized **regions** (1–32MB). Each region is dynamically Eden, Survivor, Old, or
Humongous.

**How G1 works:**

- **Concurrent mark** identifies live objects across the whole heap (mostly concurrent with the app).
- **Mixed GC** (stop-the-world) collects Young gen + some Old regions with the most garbage (hence "Garbage-First").
- **Full GC** is the fallback when memory is exhausted before mixed GC can keep up — slow, single-threaded in older
  versions, parallel since Java 10.

**Pause target:** `-XX:MaxGCPauseMillis=200` (default 200ms). G1 estimates which regions to collect to fit the target.
Not a hard guarantee.

**When G1 struggles:**

- Heap fragmentation from humongous objects.
- Concurrent mark can't keep up with allocation rate → falls back to Full GC.
- Pause target too aggressive — G1 spends time on marking, not collecting.

### ZGC

`-XX:+UseZGC`. Production-ready since Java 15.

**Goals:** sub-10ms pauses on multi-terabyte heaps. Colored pointers + load barriers enable concurrent object
relocation. No fragmentation (compaction is concurrent).

**When to use:** heaps > 16GB where pause time matters (real-time analytics, low-latency APIs).

### Shenandoah

`-XX:+UseShenandoahGC`. Red Hat's collector, similar goals to ZGC. Uses **brooks pointers** (an extra pointer per
object) for concurrent relocation.

## Common Flags

```
-Xms4g                Initial heap size
-Xmx4g                Max heap size (often set = Xms in prod to avoid resizing)
-Xss512k              Thread stack size
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseG1GC          Use G1
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app.hprof
-XX:+PrintGCDetails  (Java 8)  /  -Xlog:gc* (Java 9+)
```

## Choosing a Collector

| Heap / Goal                     | Recommended       |
|---------------------------------|-------------------|
| < 100MB, single-thread          | Serial            |
| 100MB–2GB, throughput           | Parallel          |
| 2GB–16GB, balanced              | G1 (default)      |
| > 16GB, low pause               | ZGC or Shenandoah |
| Latency-sensitive microservices | ZGC (Java 17+)    |

## Common Pitfalls

1. **`-Xmx` too small** → frequent Full GCs, OOM risk.
2. **`-Xmx` too large** without concurrent GC → long STW pauses.
3. **`System.gc()` calls** → forces Full GC, can pause for seconds. Disable with `-XX:+DisableExplicitGC`.
4. **Direct ByteBuffer leak** → native memory not in heap, looks like RSS grows but heap looks fine.
5. **Finalizers / `Cleaner`** → delays GC of objects, can cause memory pressure. Use `try-with-resources` instead.
6. **Thread locals not cleared** → classloader leak in app servers / hot reloads.
7. **Large caches with strong references** → looks like memory leak. Use `WeakReference` or Caffeine with eviction.
8. **String deduplication** in G1 — strings are common; `-XX:+UseStringDeduplication` saves heap (G1 only).

## Diagnostics

```bash
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram
jcmd <pid> Thread.print
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr
```

- **Visual tools:** JConsole, VisualVM, JProfiler, async-profiler.
- **Heap dump:** `-XX:+HeapDumpOnOutOfMemoryError`, then analyze with MAT (Eclipse Memory Analyzer).

## Code Examples

### Forcing GC (don't)

```java
System.gc();   // suggestion, not a guarantee — and slows down the app
```

### Try-with-resources (preferred over finalizers)

```java
try(Connection c = dataSource.getConnection();
PreparedStatement ps = c.prepareStatement(sql)){
        // resources auto-closed — no need for finalizers
        }
```

### Weak reference cache

```java
Map<String, WeakReference<Bitmap>> cache = new ConcurrentHashMap<>();
```

Or use Caffeine:

```java
Cache<String, Bitmap> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build();
```

## Common Interview Questions

1. **What are the JVM memory areas?**
   → Heap (shared, GC'd), Metaspace (class metadata, off-heap), JVM Stack (per-thread, frames), Native Stack (JNI), PC
   Register (per-thread).

2. **What's the difference between PermGen and Metaspace?**
   → PermGen was a fixed-size part of the heap (pre-Java 8) → `OOM: PermGen space`. Metaspace is native memory (
   off-heap), grows dynamically.

3. **What is the generational hypothesis?**
   → Most objects die young. GC separates Young and Old gens so it can use efficient copying GC for Young (where few
   objects survive) and less-frequent mark-sweep-compact for Old.

4. **What are GC roots?**
   → Starting points for the reachability trace: local variables, active threads, static fields, JNI globals, held
   monitors. Objects not reachable from roots are garbage.

5. **What's the difference between Minor GC, Major GC, and Full GC?**
   → Minor: Young gen only (fast, frequent). Major: Old gen (sometimes used loosely for any old-gen collection). Full:
   entire heap + metaspace (slow, last-resort).

6. **Which GC is the default in Java 17?**
   → G1 (since Java 9). For very large heaps or low-pause needs, choose ZGC.

7. **How does G1 work?**
   → Heap divided into regions (Eden, Survivor, Old, Humongous). Concurrent mark identifies garbage. Mixed GC collects
   Young + the "garbage-first" Old regions to hit the pause target.

8. **What's the difference between ZGC and G1?**
   → G1 has STW pauses for collection (target ~200ms). ZGC is mostly concurrent with sub-10ms pauses via colored
   pointers + load barriers. ZGC for very large / latency-sensitive heaps.

9. **What is STW (Stop-The-World)?**
   → A pause where all application threads stop. Even "concurrent" GCs have brief STW phases for marking start/end and
   relocation.

10. **What happens when `System.gc()` is called?**
    → It's a hint, not a command. JVM may or may not honor it. In production, often disabled with
    `-XX:+DisableExplicitGC` because explicit GCs cause long pauses.

11. **What is `MaxTenuringThreshold`?**
    → Number of minor GCs an object survives before being promoted to Old gen. Default 15 (for parallel/G1).

12. **How do you detect a memory leak?**
    → Heap dump on OOM, MAT to find GC-root paths to suspected leaked objects. Common: caches with strong refs, unclosed
    resources, thread-local leaks.

## Related

- `Java/jvm-internals/class-loading.md` — class loading, metaspace
- `Java/concurrency/synchronized-and-volatile.md` — JMM, memory visibility
- `Java/core/collections-hashmap-internals.md` — `WeakReference` for caches
- `System-Design/HLD/caching-strategies.md` — application-level caching
- `DevOps-Cloud/Monitoring/` — JVM metrics, GC dashboards

## Resources

- **Java docs:** `java.lang.management`, `jcmd`
- **"Java Performance" (Scott Oaks)** — comprehensive JVM tuning
- **Aleksey Shipilëv:** https://shipilev.net/jvm/ — deep dives
- **JEP 377 (ZGC):** https://openjdk.org/jeps/377
- **JEP 379 (Shenandoah):** https://openjdk.org/jeps/379
