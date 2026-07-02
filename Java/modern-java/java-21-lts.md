# Java 21 LTS (September 2023)

Project Loom lands. Pattern matching matures. The most impactful LTS since Java 8 — virtual threads change the
concurrency story for I/O-bound work.

## Theme

Java 21 brings **virtual threads** (JEP 444), pattern matching for switch (JEP 441), record patterns (JEP 440), and
sequenced collections. Spring Boot 3.2+ leverages virtual threads. This is the LTS that "modern concurrency" finally
arrived.

## Key Features

### Virtual Threads (JEP 444)

Lightweight threads scheduled by the JVM on carrier (platform) threads. Millions can run concurrently. The biggest
concurrency shift since `java.util.concurrent` in Java 5.

```java
try(ExecutorService es = Executors.newVirtualThreadPerTaskExecutor()){
        for (Socket client : clients) {
            es.submit(() -> handleRequest(client));
        }
    }
```

See `Java/concurrency/virtual-threads.md` for the full deep dive: pinning, structured concurrency, scoped values.

### Pattern Matching for Switch (JEP 441)

`switch` now supports pattern labels and guards:

```java
String describe(Shape s) {
    return switch (s) {
        case Circle c -> "circle radius " + c.radius();
        case Square s -> "square side " + s.side();
        case Triangle t -> "triangle";
        case null -> "null shape";          // explicit null case
        default -> "unknown";
    };
}
```

With **guards** (`when`):

```java
case Circle c
when c.

radius() >100->"big circle";
        case
Circle c                       ->"small circle";
```

**Exhaustiveness:** sealed types allow the compiler to verify all cases are covered. No `default` needed if all
permitted subtypes are matched.

**`null` case:** traditionally `switch` on `null` threw NPE silently. Now you can handle it explicitly:
`case null -> ...`.

### Record Patterns (JEP 440)

Destructure records inside patterns:

```java
Object obj = new Point(3, 4);
if(obj instanceof

Point(int x, int y)){
        System.out.

println("distance: "+Math.hypot(x, y));
        }
```

Nested record patterns:

```java
record Box(Point corner, int size) {
}
if(box instanceof

Box(Point(int x, int y),

int s)){...}
```

Pairs naturally with sealed types and pattern matching for switch.

### Sequenced Collections (JEP 431)

New interfaces with defined **encounter order**:

- `SequencedCollection<E>` — `addFirst`, `addLast`, `getFirst`, `getLast`, `reversed()`
- `SequencedSet<E>` — same but no duplicates
- `SequencedMap<K, V>` — `firstEntry`, `lastEntry`, `reversed()`

Existing collections retrofitted:

- `List` is a `SequencedCollection`
- `LinkedHashSet` is a `SequencedSet`
- `LinkedHashMap` is a `SequencedMap`

```java
List<Integer> reversed = list.reversed();              // new view
list.

addFirst(0);                                       // was: list.add(0, 0)

Map.Entry<K, V> first = linkedMap.firstEntry();         // was: poll/iterate
```

### String Templates (Preview, JEP 430)

⚠️ **This was a preview in Java 21 but was withdrawn in Java 23** (deferred for redesign). Mention for awareness, don't
rely on.

```java
String name = "Alice";
String greeting = STR."Hello \{name}";    // "Hello Alice"
```

### Unnamed Patterns and Variables (Preview, JEP 443)

`_` for "ignore":

```java
try{...}
        catch(Exception _){

log("failed"); }   // no need to name the exception

        if(obj instanceof

Point(_, int y)){...}   // ignore x
```

Finalized later (Java 22).

### Pattern Matching Improvements

`instanceof` pattern binding (Java 16+) + record patterns (Java 21) compose:

```java
if(obj instanceof

Point(int x, int y) &&x ==y){...}
```

## What's New in Tooling

- **JFR (Java Flight Recorder)** stable, low overhead — even in production.
- **`jcmd`** enhanced for virtual thread dumps.
- **Spring Boot 3.2+** supports `spring.threads.virtual.enabled=true`.

## Migration Notes

- **Pinning:** `synchronized` with blocking calls inside pins the carrier in Java 21. Use `ReentrantLock` for hot
  paths. (Fixed in Java 25 — see `java-25-lts.md`.)
- **Pattern matching for switch** is final — refactor verbose `if/else if instanceof` chains to `switch`.
- **Sequenced Collections** — replace `linkedMap.entrySet().iterator().next()` with `linkedMap.firstEntry()`.

## Common Interview Questions

1. **What's new in Java 21?**
   → Virtual threads (JEP 444), pattern matching for switch (JEP 441), record patterns (JEP 440), sequenced
   collections (JEP 431), string templates (preview, later withdrawn).

2. **What are virtual threads and why are they a big deal?**
   → JVM-scheduled lightweight threads that unmount on blocking I/O. Allow millions of concurrent I/O-bound tasks
   without thread-per-request limits. See `Java/concurrency/virtual-threads.md`.

3. **What is pattern matching for switch?**
   → `switch` cases can match types and bind variables: `case Circle c -> ...`. Guards via `when`. Exhaustiveness
   checked for sealed types.

4. **What are record patterns?**
   → Destructure records in patterns: `instanceof Point(int x, int y)`. Nested record patterns compose naturally.

5. **What are sequenced collections?**
   → New interfaces (`SequencedCollection`, `SequencedSet`, `SequencedMap`) with defined encounter order and methods
   like `addFirst`, `getFirst`, `reversed()`. Existing collections retrofitted.

6. **What is pinning in the context of virtual threads?**
   → A virtual thread can't unmount from its carrier when it holds a `synchronized` monitor or is in a native frame.
   Detect via JFR event `jdk.VirtualThreadPinned`. Fix: `ReentrantLock` (Java 21) or upgrade to Java 25.

7. **What is the `null` case in switch (Java 21)?**
   → `case null -> ...` lets you handle nulls explicitly instead of getting an NPE. Combined with patterns, switch is
   finally null-safe.

8. **Was String Templates in Java 21 stable?**
   → No — preview only. It was withdrawn in Java 23 for redesign. Don't use in production code yet.

## Related

- `Java/modern-java/java-17-lts.md` — previous LTS
- `Java/modern-java/java-25-lts.md` — next LTS, fixes pinning
- `Java/concurrency/virtual-threads.md` — full deep dive
- `Java/concurrency/locks-and-atomic.md` — `ReentrantLock` as pinning workaround

## Resources

- **JEP index:** https://openjdk.org/jeps/0 — Java 18-21 JEPs
- **Oracle Java 21 docs:** https://docs.oracle.com/en/java/javase/21/
- **Inside Java podcasts:** Project Loom episodes
