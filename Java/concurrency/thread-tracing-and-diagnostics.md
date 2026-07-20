# Thread Tracing & Diagnostics

Java gives you an unusually rich set of tools to inspect what its threads are doing — from a single static snapshot
(`jstack`) to continuous profiling (JFR, async-profiler) and cross-service tracing (OpenTelemetry). Knowing which
tool to reach for is the difference between fixing a production hang in two minutes and chasing it for two days.

A frequent senior-interview topic, especially in Java/Spring shops running microservices. Expect to discuss: how to
capture a thread dump inside a container, what `BLOCKED` vs `WAITING` means in a stack trace, how to detect
deadlocks programmatically, why `CompletableFuture.supplyAsync` drops your MDC, and how to spot a pinned virtual
thread.

---

## Quick Reference — Symptom ➜ Tool ➜ Fix

| Symptom                                | Likely cause                          | First tool                          | First fix                              |
|----------------------------------------|---------------------------------------|-------------------------------------|----------------------------------------|
| CPU pinned at 100% on one core         | Hot thread, busy loop, GC             | `top -H`, `async-profiler` (CPU)    | Look at on-CPU flame graph             |
| CPU pinned at 100% on all cores        | GC storm, Futex, hashing, JIT         | JFR, `jcmd GC.heap_info`            | Check GC log, heap pressure            |
| Latency spike, no CPU usage            | Lock wait, I/O, DB                    | `jstack` × 3, async-profiler (wall) | Find the `BLOCKED`/`WAITING` threads   |
| Request hangs forever                  | Deadlock or pool exhaustion           | `jstack` (auto-detects deadlock)    | Break the cycle or grow the pool       |
| Thread pool saturated, queued tasks    | Slow downstream, missing timeout      | `jstack`, `jcmd Thread.print`       | Add timeouts, isolate pools            |
| `OutOfOfMemoryError: unable to create  | Thread leak (platform threads)        | `jcmd Thread.print`, count threads  | Find the leaking `new Thread()` caller |
|   native thread`                       |                                       |                                     |                                        |
| Latency regression after Loom adoption | Virtual-thread pinning                | `-Djdk.tracePinnedThreads=full`,    | Replace `synchronized` with `ReentrantLock` |
|                                        |                                       | JFR `jdk.VirtualThreadPinned`       |                                        |
| Lost user id / trace in async logs     | `ThreadLocal` / MDC not propagated    | Log instrumentation check           | Custom `Executor` wrapper, `ScopedValue` |
| Random slow requests in microservices  | Cross-service causal gap              | OpenTelemetry distributed trace     | Instrument HTTP/Kafka/executor hops    |

---

## 1. The Three Kinds of "Thread Trace"

The phrase "thread trace" is overloaded. Three very different techniques share the name:

| Kind             | What it captures                              | Cost            | Best for                          |
|------------------|-----------------------------------------------|-----------------|-----------------------------------|
| **Snapshot**     | All threads' stacks **at one instant**        | ~0 (one freeze) | Deadlocks, deadlocks, hung threads |
| **Sampling profile** | Statistical samples at fixed interval    | 1–3% overhead   | Hot paths, on/off-CPU time        |
| **Distributed trace** | Causal path of one request across services | ~µs per span    | Microservice latency, async hops  |

> **Mental model:** A snapshot is a **photograph** of all threads; a sampling profile is a **long-exposure
> video** of one thread's CPU time; a distributed trace is a **story** of one request as it crosses threads,
> services, and processes.

---

## 2. Tooling

### 2.1 JDK built-ins (always available, no extra install)

| Tool                      | What it does                                                      |
|---------------------------|-------------------------------------------------------------------|
| `jstack <pid>`            | Prints thread dump of a JVM process to stdout                     |
| `jcmd <pid> Thread.print` | Same as `jstack`, but uses the JVM's own attach API               |
| `kill -3 <pid>`           | Sends `SIGQUIT` → JVM prints the dump to its `stderr`             |
| `jcmd <pid> Thread.print -l` | With **lock info** (`-l`) — shows ownable synchronizers        |
| `jcmd <pid> JFR.start duration=60s filename=/tmp/r.jfr` | Start a 60s JFR recording |
| `jcmd <pid> VM.version`   | Sanity check; verifies you can attach                             |
| `jhsdb jstack --pid <pid>` | Modern replacement for `jstack` (JDK 9+)                         |
| `jconsole`, `jvisualvm`   | GUI live view of threads, MBeans, heap                            |
| `JDK Mission Control`     | GUI for analysing JFR files                                       |

> **Inside containers:** the JDK tools attach via UNIX domain sockets in `/tmp/.java_pidNNNN`. The easiest way is
> to `exec` into the container: `docker exec <id> jcmd 1 Thread.print`. If you cannot, run the JVM with
> `-XX:+StartAttachListener` and use `jhsdb` from outside with the same PID namespace.

### 2.2 External profilers

| Tool              | Type          | Strengths                                              |
|-------------------|---------------|--------------------------------------------------------|
| **async-profiler**| Sampling      | Low overhead, flame graphs, no JVMTI agent startup cost, ASGCT |
| **JProfiler / YourKit** | Instrumenting | Live UI, deep object/lock introspection                |
| **honest-profiler**| Sampling     | Lightweight, log-based                                  |
| **py-spy / perf** | OS-level      | When Java tools can't attach (e.g., JVM hang)          |

### 2.3 Programmatic (in-process)

```java
import java.lang.management.*;

// Current thread's stack
StackTraceElement[] trace = Thread.currentThread().getStackTrace();

// All live threads + their states
Map<Thread, StackTraceElement[]> all = Thread.getAllStackTraces();

// Platform MXBean — much richer
ThreadMXBean tb = ManagementFactory.getThreadMXBean();

long[] deadlocks = tb.findDeadlockedThreads();        // monitor + ReentrantLock
long[] monitorDeadlocks = tb.findMonitorDeadlockedThreads();  // monitor only

ThreadInfo[] infos = tb.dumpAllThreads(true, true);   // with locked monitors + synchronizers
for (ThreadInfo info : infos) {
    System.out.printf("%s [%s]%n", info.getThreadName(), info.getThreadState());
    for (StackTraceElement e : info.getStackTrace()) {
        System.out.println("    at " + e);
    }
}
```

This is how APM agents (Datadog, New Relic, AppDynamics) build "live thread" views and how Spring Boot Actuator's
`/threaddump` endpoint works.

---

## 3. Reading a Thread Dump

### 3.1 Thread states — what each means

```
   NEW              — created but not started (rare in dumps)
   RUNNABLE         — running OR ready to run (OS-runnable)
                      → on-CPU OR in OS run-queue
   BLOCKED          — waiting for a monitor lock (synchronized)
                      → contention on `synchronized`
   WAITING          — indefinite wait: Object.wait(), LockSupport.park(), Thread.join() without timeout
                      → blocked on a CountDownLatch, Future.get(), BlockingQueue.take()
   TIMED_WAITING    — same as WAITING but with a timeout
                      → Thread.sleep, wait(ms), poll(timeout)
   TERMINATED       — run() finished
```

> **Critical:** `RUNNABLE` does **not** mean "consuming CPU right now". It means the JVM thinks this thread *could*
> run. The OS may have it in its run-queue, or — crucially — it may be blocked inside a native syscall
> (`socketRead`, `epollWait`) that the JVM doesn't model as blocking. **CPU-bound hot threads show up as `RUNNABLE`
> and stay in the same stack across multiple dumps;** I/O-blocked threads also show `RUNNABLE` but their stack
> changes and the top frame is a native `read`/`poll`.

### 3.2 An annotated thread entry

```
"http-nio-8080-exec-3" #45 daemon prio=5 os_prio=0 cpu=12.50ms elapsed=600.20s tid=0x... nid=0x4e3b runnable
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)         ← native — actually waiting on socket
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:171)
        at org.apache.catalina.connector.CoyoteAdapter.service(...)
        at com.example.MyController.handle(MyController.java:42)         ← your code
        ...

   Locked ownable synchronizers:                                       ← appears with -l flag
        - <0x000000076b4a1f30> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```

| Field | Meaning                                                                  |
|-------|--------------------------------------------------------------------------|
| `tid` | JVM-internal thread id (hex) — internal correlation                      |
| `nid` | **Native OS thread id** (hex) — use to match `top -H -p <pid>` output    |
| `cpu` | Approximate CPU time consumed so far                                     |
| `elapsed` | Wall-clock since thread started                                      |
| `Locked ownable synchronizers` | `ReentrantLock`/`Semaphore`/`CountDownLatch` held by this thread |

**Tip:** convert `nid` from hex to decimal and feed to `top -H -p <pid>` to find which thread burns CPU.

### 3.3 Common patterns

#### Pattern A — pool starvation (busy executors)

```
"http-nio-8080-exec-1" … TIMED_WAITING
    at jdk.internal.misc.Unsafe.park(...)
    at java.util.concurrent.locks.LockSupport.parkNanos(...)
    at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1062)
```

This is a **healthy idle worker** — parked on `queue.poll(timeout)`. If **all** workers look like this and your
app is unresponsive, then either nothing is arriving or callers are blocked elsewhere. If **none** look like this
and new requests are queueing, your pool is too small or one task is stuck.

#### Pattern B — slow downstream (I/O wait)

```
"http-nio-8080-exec-7" … RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    …
    at org.apache.http.impl.client.CloseableHttpClient.execute(...)
    at com.example.PaymentClient.charge(PaymentClient.java:88)         ← your code
```

`RUNNABLE` but stuck in `socketRead0`. The JVM doesn't know the socket is blocked. This is the classic "looks
busy, isn't" pattern. Fix with **timeouts on every outbound call**.

#### Pattern C — lock contention

```
"order-processor-3" … BLOCKED
    - waiting to lock <0x000000076abc1234> (a java.lang.Object)
    - locked <0x000000076abc9876> (a java.util.concurrent.ConcurrentHashMap)
    at com.example.InventoryService.decrement(InventoryService.java:55)
    …
"order-processor-7" … locked <0x000000076abc1234>                      ← holds the lock
    at com.example.InventoryService.recalc(InventoryService.java:120)  ← slow holder
```

One thread holds the monitor; others are `BLOCKED` waiting for it. Take three dumps 10s apart — if the same
thread holds the same lock in all three, that's the bottleneck.

#### Pattern D — deadlock (auto-detected)

```
Found one Java-level deadlock:
=============================
"thread-A":
  waiting to lock monitor 0x00007f... (object 0x000000076abc1234, a java.lang.Object),
  which is held by "thread-B"
"thread-B":
  waiting to lock monitor 0x00007f... (object 0x000000076abc9876, a java.lang.Object),
  which is held by "thread-A"
```

`jstack` prints this automatically when it detects a monitor cycle. For `ReentrantLock` cycles use
`jcmd Thread.print -l` (with `-l`) or `ThreadMXBean.findDeadlockedThreads()`.

#### Pattern E — GC threads dominating

If most threads are named `G1 Main Marker`, `GC Thread#n`, `VM Thread`, `C2 CompilerThread`, you are looking at
**JVM internal threads**, not application threads. A dump where these dominate usually means GC pressure or a JIT
storm — cross-check with the GC log.

### 3.4 The "three dumps, 10 seconds apart" rule

A single snapshot is often noise. Take three:

```bash
jcmd 1 Thread.print > /tmp/d1.txt; sleep 10
jcmd 1 Thread.print > /tmp/d2.txt; sleep 10
jcmd 1 Thread.print > /tmp/d3.txt
```

A thread **actually executing** shows **different** stack frames across the dumps. A thread **stuck** shows the
**same** frames. Tools like `jstack.review`, `fastthread.io`, or `IBM Thread and Monitor Dump Analyzer` visualise
the diff.

---

## 4. Deadlock Detection

### 4.1 Monitor deadlock (`synchronized`)

```java
final Object a = new Object(), b = new Object();

new Thread(() -> { synchronized (a) { sleep(); synchronized (b) {} } }, "T1").start();
new Thread(() -> { synchronized (b) { sleep(); synchronized (a) {} } }, "T2").start();
```

Detection:

```bash
jstack <pid>           # prints "Found 1 deadlock" at the end
jcmd <pid> Thread.print
```

Programmatic:

```java
ThreadMXBean tb = ManagementFactory.getThreadMXBean();
long[] ids = tb.findMonitorDeadlockedThreads();   // monitor-only cycle
if (ids != null) {
    ThreadInfo[] info = tb.getThreadInfo(ids, true, true);
    // log, alert, kill the JVM, etc.
}
```

### 4.2 `ReentrantLock` / `CountDownLatch` deadlock

`findMonitorDeadlockedThreads` only sees `synchronized`. For `Lock`/`Semaphore`/`CountDownLatch` cycles use
`findDeadlockedThreads()` (which includes monitor cycles too):

```java
long[] ids = tb.findDeadlockedThreads();   // monitor + ownable-synchronizer cycle
```

> Spring Boot Actuator exposes both via the `/threaddump` and (with the right actuator) programmatic endpoints.
> APM agents periodically call `findDeadlockedThreads()` and alert.

---

## 5. Lock Contention & Off-CPU Analysis

A `BLOCKED` thread is **off-CPU** — it consumes no cycles, but it slows the request. To measure contention:

### 5.1 JFR lock events

```bash
jcmd 1 JFR.start duration=60s filename=/tmp/r.jfr settings=profile
# After 60s, open in JMC; look at:
#   jdk.JavaMonitorWait    — wait() on monitor
#   jdk.MonitorEnter       — entering a synchronized block (contention time)
#   jdk.ThreadPark         — LockSupport.park() (most BlockingQueue ops)
```

Each event records duration, the monitor/class involved, the holder thread, and the stack of the waiter. Sort by
total time to find the lock that's hurting most.

### 5.2 async-profiler lock mode

```bash
./asprof -d 60 -e lock -f /tmp/locks.html <pid>   # show lock contention flame graph
./asprof -d 60 -e wall -f /tmp/wall.html <pid>    # off-CPU (wall-clock) flame graph
```

A **wall-clock profile** catches both CPU-bound and off-CPU time (waiting on locks, I/O). It's the right tool
when "CPU is idle but latency is high".

---

## 6. CPU vs Latency Profiling — On-CPU vs Off-CPU

The single biggest distinction in profiling:

```
   ┌─────────────────────────────────────────────────────────────────┐
   │                  Wall-clock time of one request                 │
   │                                                                 │
   │   ├─────────┬─────────────────────────┬───────────┬───────────┤ │
   │   │ on-CPU │        off-CPU          │  on-CPU   │  off-CPU  │ │
   │   │ compute│ wait on DB, lock, sleep │ compute   │ wait ...  │ │
   │   └─────────┴─────────────────────────┴───────────┴───────────┘ │
   │                                                                 │
   │   A "CPU profile" samples only the on-CPU parts.                │
   │   A "wall profile" samples everything — CPU + wait.             │
   └─────────────────────────────────────────────────────────────────┘
```

- **CPU profile** (`-e cpu`, default) → tells you **why CPU is hot**. Use when CPU is the bottleneck.
- **Wall profile** (`-e wall`) → tells you **where the request spent time**, including waiting. Use for latency
  investigations, especially when CPU usage is low.

### Reading a flame graph

A flame graph is a horizontal stack of stack-frames, where:
- **Width** = number of samples (proportional to time spent).
- **Vertical stacking** = call depth.
- **Colour** is arbitrary (warm hues by convention).

A **wide bar** near the top is a single hot leaf (a busy loop). A **wide tower** is a deep call chain that's
collectively expensive. Compare CPU and wall flame graphs side by side: a bar that's wide in wall but absent in
CPU = **waiting** (lock, I/O, sleep). A bar that's wide in CPU = **real work**.

### JFR execution samples

```
jdk.ExecutionSample        — sampled stack at ~10–100ms interval when thread is on-CPU
jdk.ExecutionStat          — aggregated statistics
```

JFR is sampling-only by default. For wall-clock in JFR, set `thread-allocation-enabled` or use async-profiler.

---

## 7. Distributed & Async Context Tracing

In a single JVM, a thread's stack is its identity — and logs / metrics naturally attach to it. In modern apps this
breaks twice:

1. **Async boundaries** — `Executor.submit`, `CompletableFuture.supplyAsync`, reactive schedulers — the **stack is
   severed**. The continuation runs on a different thread, with a different stack.
2. **Service boundaries** — HTTP, Kafka, gRPC — different processes, no shared stack at all.

### 7.1 The MDC problem

```java
MDC.put("userId", "u123");                                  // (1) on request thread
CompletableFuture.supplyAsync(() -> {
    log.info("done");                                        // (2) on pool thread
    // MDC is empty here — userId is lost!
    return null;
});
```

`MDC` (SLF4J / Logback) is backed by a `ThreadLocal`. `ThreadLocal` does **not** propagate across thread
boundaries. So the log at (2) has no `userId`.

### 7.2 Solutions

| Approach                                | When                                                            |
|-----------------------------------------|-----------------------------------------------------------------|
| **Custom `Executor` wrapper**           | Manually copy MDC map into the task, restore after             |
| **`CompletableFuture` copy**            | Wrap `supplyAsync` to snapshot MDC                              |
| **Reactor `Hooks.enableAutomaticContextPropagation()`** | Reactive apps with `io.micrometer.context`     |
| **OpenTelemetry `Context`/`Scope`**     | OTel-instrumented apps — its own propagation                    |
| **`ScopedValue` (Java 21+, preview)**   | Immutable, bounded, virtual-thread-friendly ThreadLocal replacement |
| **Micrometer Context Propagation**      | Library that abstracts the above for servlet/reactive          |

### 7.3 Custom Executor that propagates MDC

```java
public static Executor contextPropagating(Executor delegate) {
    return command -> {
        Map<String, String> snapshot = MDC.getCopyOfContextMap();
        delegate.execute(() -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            if (snapshot != null) MDC.setContextMap(snapshot); else MDC.clear();
            try {
                command.run();
            } finally {
                if (previous != null) MDC.setContextMap(previous); else MDC.clear();
            }
        });
    };
}

CompletableFuture.supplyAsync(supplier, contextPropagating(ForkJoinPool.commonPool()));
```

### 7.4 OpenTelemetry distributed trace

```
   service A                service B                service C
   ┌────────────┐           ┌────────────┐           ┌────────────┐
   │  HTTP span │──trace──▶│  HTTP span │──trace──▶│  DB span   │
   │ (parent)   │  context │            │  context │            │
   └────────────┘  via W3C └────────────┘           └────────────┘
                   traceparent
                   header
```

- Each **span** has a `traceId` (root) and a `spanId` (this hop).
- A **trace** is the directed acyclic graph of all spans sharing a `traceId`.
- The context is **propagated** via HTTP headers (`traceparent`), Kafka headers, gRPC metadata.
- Inside a JVM, the current span is stored in a thread-local-like `Context`, and **OTel instrumentation**
  propagates it across executor boundaries for you (if you use the auto-agent or the instrumented libraries).

This is the modern answer to "which thread, on which service, did what for this request?" — the **distributed
trace** is the cross-service equivalent of a stack.

---

## 8. Virtual-Thread Specifics

Virtual threads are dumped differently from platform threads:

```bash
# New in JDK 21: dump virtual threads (not just platform)
jcmd <pid> Thread.print                        # default — includes vthreads in JDK 21+
jcmd <pid> Thread.dump_to_file /tmp/d.txt      # plain-text dump (no `print` size limit)
jcmd <pid> Thread.dump_to_file -format=json /tmp/d.json
```

Virtual threads are **not** scheduled 1:1 on OS threads, so a thread dump shows the carrier threads at the top,
then the virtual threads (often unmounted, shown with no Java stack).

### Detecting pinning

A virtual thread is **pinned** when it cannot unmount from its carrier — typically because it's inside a
`synchronized` block or a native frame. The carrier is stuck for the duration.

```bash
# Run the JVM with (or set as -D at startup):
-Djdk.tracePinnedThreads=full      # full stack whenever a vthread pins
-Djdk.tracePinnedThreads=short     # one-line summary

# Or capture via JFR:
jcmd <pid> JFR.start duration=60s filename=/tmp/r.jfr settings=profile
# Look at:  jdk.VirtualThreadPinned  events
#           jdk.VirtualThreadStart   / jdk.VirtualThreadEnd   (lifecycle)
#           jdk.VirtualThreadSubmit  (scheduler queue)
```

Each `jdk.VirtualThreadPinned` event records the **duration** and the **stack** of the pinned vthread. Sort by
duration descending to find the worst offenders.

> See `Java/concurrency/virtual-threads.md` for the full discussion of when pinning matters and how to fix it.

---

## 9. Production Playbook

When something goes wrong in production, follow this order:

1. **Confirm the symptom.** Is CPU hot? Latency high? Memory growing? Requests hanging? Different symptom ➜
   different tool.
2. **Capture first, analyse later.** Production evidence is short-lived; restart loses it.
   - Thread dump × 3 (10s apart): `jcmd 1 Thread.print > /tmp/dN.txt`
   - JFR 60s profile: `jcmd 1 JFR.start duration=60s filename=/tmp/r.jfr settings=profile`
   - Heap dump only on `OutOfMemoryError` (`-XX:+HeapDumpOnOutOfMemoryError`).
3. **Look for the obvious patterns first.**
   - Deadlock (auto-printed by `jstack`).
   - Pool starvation (all workers stuck in same stack).
   - Slow downstream (`socketRead0` in stack).
   - Hot CPU (same RUNNABLE stack across all three dumps).
4. **Cross-correlate.** Convert `nid=0x4e3b` to decimal 20027, then `top -H -p <pid>` to find which Java thread
   burns CPU.
5. **Profile if needed.** Sampling (JFR, async-profiler) for hot paths; wall mode for latency.
6. **Match the metric.** Once you have a hypothesis, find the metric in your APM that confirms it.

### Overhead ranking — safe to run in prod

| Tool                       | Overhead                  | Safe to run in prod                |
|----------------------------|---------------------------|------------------------------------|
| `jstack` / `jcmd Thread.print` | Negligible (~10ms STW) | Yes — single dump                  |
| `kill -3`                  | Same as jstack            | Yes — but log noise                |
| JFR default (`default` profile) | < 1%                 | Yes                                |
| JFR `profile` settings     | 1–3%                      | Yes, for short windows             |
| async-profiler CPU         | 1–3%                      | Yes                                |
| async-profiler wall        | 1–3%                      | Yes                                |
| `ThreadMXBean.dumpAllThreads` in a loop | Heavy if called often | No, sparingly       |
| Bytecode-instrumenting profilers (JProfiler, YourKit in tracing mode) | 10%+ | Usually no — sample mode only |

> **Rule of thumb:** start with a thread dump (free), escalate to JFR (cheap), escalate to async-profiler (cheap),
> only then reach for a full instrumenting profiler.

---

## 10. Common Interview Questions

1. **What's the difference between `BLOCKED` and `WAITING` in a thread dump?**
   `BLOCKED` is waiting to acquire a **monitor** (after `synchronized`); another thread holds it. `WAITING` is
   waiting indefinitely via `Object.wait()`, `LockSupport.park()`, or `join()` without timeout — typically on a
   `BlockingQueue`, `CountDownLatch`, `Future.get()`, etc. `BLOCKED` involves contention; `WAITING` involves
   voluntary handoff.

2. **What does `RUNNABLE` mean — is the thread always running?**
   No. `RUNNABLE` means the JVM considers it eligible to run. The OS may have it in its run-queue, or it may be
   blocked inside a **native syscall** (`socketRead0`, `epollWait`) that the JVM doesn't model as blocking. A
   thread stuck on I/O usually shows `RUNNABLE` with a native `read`/`poll` top frame.

3. **How do you find a deadlocked JVM in production?**
   `jstack <pid>` or `jcmd <pid> Thread.print` — both automatically detect monitor deadlocks and print the cycle.
   For `ReentrantLock` cycles use `ThreadMXBean.findDeadlockedThreads()` programmatically, or `jcmd Thread.print
   -l` to include ownable synchronizers.

4. **`findDeadlockedThreads` vs `findMonitorDeadlockedThreads`?**
   `findMonitorDeadlockedThreads` only detects cycles on intrinsic monitors (`synchronized`).
   `findDeadlockedThreads` (note: no "monitor" in the name) detects both monitor cycles **and** cycles on
   ownable synchronizers (`ReentrantLock`, `Semaphore`, `CountDownLatch`).

5. **How do you take a thread dump from inside a Docker container?**
   `docker exec <id> jcmd 1 Thread.print` (the JVM typically runs as PID 1). If you can't exec, run the JVM with
   `-XX:+StartAttachListener` and use `jhsdb jstack --pid <pid>` from a sidecar in the same PID namespace.

6. **What's the difference between on-CPU and off-CPU time?**
   On-CPU time is the time the OS scheduler has the thread running instructions. Off-CPU time is the time it's
   waiting (lock, I/O, sleep). Total wall-clock latency = on-CPU + off-CPU. A CPU profile shows only on-CPU; a
   wall-clock profile shows both. For latency investigations, use wall-clock.

7. **Why does `CompletableFuture.supplyAsync` lose my MDC?**
   Because MDC is backed by `ThreadLocal`, and `ThreadLocal` does not propagate across thread boundaries. The
   supplier runs on a different thread from the caller, so its MDC map is empty. Fix with a custom `Executor`
   that snapshots and restores MDC, or use Micrometer Context Propagation.

8. **What is `ScopedValue` and why is it better for virtual threads?**
   `ScopedValue` (Java 21+ in preview) is an **immutable, bounded** replacement for `ThreadLocal`. It's bound
   for the duration of a `ScopedValue.where(KEY, value).run(...)`. The JVM can inherit it across virtual-thread
   boundaries efficiently and avoids the lifetime leaks that `ThreadLocal` causes when millions of virtual
   threads each hold one.

9. **What is a pinned virtual thread and how do you detect it?**
   A virtual thread that cannot unmount from its carrier — typically because it's inside a `synchronized` block
   or a native frame while blocking. The carrier is stuck for the duration. Detect with
   `-Djdk.tracePinnedThreads=full` or the JFR `jdk.VirtualThreadPinned` event; the latter records duration and
   stack.

10. **Why is sampling better than instrumentation for production profiling?**
    Instrumentation adds code to every method invocation — fixed overhead per call, breaks inlining, hurts hot
    paths disproportionately. Sampling takes a statistical snapshot at a fixed interval — overhead is bounded and
    independent of call frequency, with the trade-off that rare events may be missed.

11. **How do you correlate a hot CPU thread from `top` with a Java thread in a dump?**
    `top -H -p <pid>` shows OS threads (TIDs). Convert the TID to hex; it matches the `nid=` field of a thread
    dump entry. Cross-reference to find which Java thread is the hot one.

12. **What is a flame graph and how do you read it?**
    Horizontal bars where width = sample count (proportional to time), vertical stacking = call depth. Wide bars
    near the top are hot leaves; wide towers are deep expensive chains. Compare CPU vs wall flame graphs: a bar
    wide in wall but absent in CPU = waiting time.

13. **You see 200 threads all parked on `LinkedBlockingQueue.take` in your dump. Is this a problem?**
    Probably not — these are **idle worker threads** in a thread pool, waiting for tasks. The pool size is just
    large. It's a problem only if your app is unresponsive (meaning tasks are arriving but being lost) or if you
    didn't intend the pool to be that large (memory overhead — each platform thread costs ~1 MB stack).

14. **What's a distributed trace and how does it differ from a thread dump?**
    A thread dump is a static, in-process snapshot of all threads' stacks. A distributed trace is the causal
    story of **one request** as it crosses threads, processes, and services, stitched together by a shared
    `traceId` propagated via headers. Thread dump = "what is every thread doing now"; distributed trace = "what
    did this one request do, end-to-end".

---

## 11. Mental Cheat-Sheet

> **Pick the tool by the symptom:**
>
> - **Hung / unresponsive** → `jstack` × 3 (10s apart), look for deadlocks and identical stacks
> - **High CPU** → `async-profiler -e cpu` or JFR `jdk.ExecutionSample`
> - **High latency, low CPU** → `async-profiler -e wall` or JFR lock events
> - **Pool exhaustion** → `jcmd Thread.print`, count by stack signature
> - **Random slow trace** → OpenTelemetry, look for span gaps
> - **Lost MDC** → custom `Executor` wrapper, or `ScopedValue`

```
   SYMPTOM                 TOOL                                  WHAT TO LOOK FOR
   ──────────────────────────────────────────────────────────────────────────────────────
   Hang                    jstack × 3                            deadlock banner; identical stacks
   High CPU                async-profiler CPU, JFR ExecutionSample   wide top bars
   Latency, low CPU        async-profiler wall, JFR JavaMonitorWait  off-CPU bars
   Deadlock                jstack, findDeadlockedThreads        "Found Java-level deadlock"
   Virtual-thread pinning  tracePinnedThreads, jdk.VirtualThreadPinned  duration + stack
   Async MDC loss          log inspection                        empty MDC on pool threads
   Cross-service latency   OpenTelemetry                         trace span gaps
```

### Related Files

- `Java/concurrency/thread-fundamentals.md` — `Thread.State` lifecycle, `start`/`join` semantics
- `Java/concurrency/synchronized-and-volatile.md` — intrinsic monitors (the source of `BLOCKED`)
- `Java/concurrency/locks-and-atomic.md` — `ReentrantLock`, `Condition`, lock contention
- `Java/concurrency/executors-and-thread-pools.md` — pools, the source of most `LinkedBlockingQueue.poll` stacks
- `Java/concurrency/virtual-threads.md` — pinning, JFR events, virtual-thread-aware `jcmd`
- `Java/concurrency/happens-before.md` — why visibility bugs are hard to spot in dumps
- `Java/jvm-internals/memory-model-and-gc.md` — GC-thread analysis, `jcmd` reference
