# Stream API

Stream API (Java 8+) — declarative processing of collections. Heavily asked at Middle interviews: operations, lazy
evaluation, parallel streams, collectors, and common pitfalls.

## What a Stream Is

A `Stream<T>` is a **pipeline** of operations on a data source (collection, array, I/O channel, generator). It is:

- **Not a data structure** — doesn't store elements.
- **Not a collection** — doesn't mutate the source.
- **Lazy** — intermediate ops are not executed until a terminal op is invoked.
- **Single-use** — after a terminal op, the stream is closed. Re-use throws `IllegalStateException`.
- **Functional** — operations take functions (lambdas / method refs), not state.

```java
List<String> result = list.stream()
        .filter(s -> s.length() > 3)
        .map(String::toUpperCase)
        .sorted()
        .toList();
```

## Stream Pipeline Structure

A pipeline has three parts:

1. **Source** — `stream()`, `Stream.of(...)`, `IntStream.range(...)`, `Files.lines(...)`.
2. **Intermediate operations** (0..n) — `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek`. Lazy,
   return a new `Stream`.
3. **Terminal operation** (exactly 1) — `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `min`/`max`.
   Triggers execution and produces a result or side effect.

Without a terminal op, **nothing happens** — the stream is just a description of what to do.

## Creating Streams

```java
// From collection
list.stream();

// From array
Arrays.

stream(array);
Stream.

of("a","b","c");

// Primitives
IntStream.

range(0,100);
IntStream.

rangeClosed(1,10);
DoubleStream.

of(1.0,2.0);

// Infinite
Stream.

generate(() ->"x");
        Stream.

iterate(1,n ->n *2);                 // 1, 2, 4, 8, ...
        Stream.

iterate(1,n ->n< 100,n ->n *2);   // Java 9+ with predicate

// From I/O
        Files.

lines(Path.of("file.txt"));              // lazy, line by line

// Empty
        Stream.

empty();
```

## Intermediate Operations

| Op                                | Purpose                         | Notes                                         |
|-----------------------------------|---------------------------------|-----------------------------------------------|
| `filter(Predicate)`               | Keep matching                   | Stateless                                     |
| `map(Function)`                   | Transform 1:1                   | Stateless; use `mapToInt` etc. for primitives |
| `flatMap(Function<T, Stream<R>>)` | Flatten nested structures       | Replaces nested loops                         |
| `distinct()`                      | Remove duplicates (by `equals`) | Stateful, uses `HashSet`                      |
| `sorted()` / `sorted(Comparator)` | Sort                            | Stateful, buffers all elements                |
| `limit(n)`                        | First n elements                | Short-circuiting                              |
| `skip(n)`                         | Skip first n                    | Stateful                                      |
| `peek(Consumer)`                  | Side-effect per element         | Mostly for debugging                          |

### map vs flatMap

`map` produces 1 output per input. `flatMap` produces 0..n outputs per input, flattened into a single stream.

```java
// map: 1:1
List<Integer> lengths = words.stream().map(String::length).toList();

// flatMap: 1:n
List<String> uniqueChars = words.stream()
        .flatMap(word -> word.chars().mapToObj(c -> (char) c))
        .distinct()
        .toList();
```

Classic example — `List<List<Order>>` flattened into `List<Order>`:

```java
List<Order> all = customers.stream()
        .flatMap(c -> c.getOrders().stream())
        .toList();
```

## Terminal Operations

| Op                                               | Returns                         | Notes                                                  |
|--------------------------------------------------|---------------------------------|--------------------------------------------------------|
| `collect(Collector)`                             | Collection / map / string       | Most common                                            |
| `toList()`                                       | `List<T>` (immutable, Java 16+) | Replaces `collect(toList())`                           |
| `forEach(action)`                                | `void`                          | Side-effect; order NOT guaranteed for parallel streams |
| `reduce(...)`                                    | `Optional<T>` or `T`            | Reduction / aggregation                                |
| `count()`                                        | `long`                          | Counts elements after filtering                        |
| `min` / `max(Comparator)`                        | `Optional<T>`                   | Needs comparator                                       |
| `findFirst()`                                    | `Optional<T>`                   | First element in encounter order                       |
| `findAny()`                                      | `Optional<T>`                   | Any element (faster on parallel)                       |
| `anyMatch` / `allMatch` / `noneMatch(Predicate)` | `boolean`                       | Short-circuiting                                       |
| `toArray()`                                      | `Object[]` / typed array        |                                                        |
| `iterator()`                                     | `Iterator<T>`                   | Escape hatch                                           |

### reduce

```java
// Sum
int sum = nums.stream().reduce(0, Integer::sum);

// Without identity → Optional (empty stream case)
Optional<Integer> product = nums.stream().reduce((a, b) -> a * b);

// Three-arg reduce for parallel (combiner merges partial results)
int parallelSum = nums.parallelStream().reduce(0, Integer::sum, Integer::sum);
```

## Collectors

`java.util.stream.Collectors` — common reduction strategies:

```java
// To list / set
.collect(Collectors.toList());      // mutable (Java 8-15), immutable (Java 16+ warnings)
        .

toList();                          // immutable (Java 16+)
.

collect(Collectors.toSet());

// To unmodifiable
        .

collect(Collectors.toUnmodifiableList());

// To map (beware of duplicate keys)
Map<Long, User> byId = users.stream()
        .collect(Collectors.toMap(User::id, Function.identity()));
// Duplicate key → IllegalStateException. Provide merge function:
.

collect(Collectors.toMap(User::id, Function.identity(), (a,b)->a));

// Grouping by
Map<Role, List<User>> byRole = users.stream()
        .collect(Collectors.groupingBy(User::role));

Map<Role, Long> countByRole = users.stream()
        .collect(Collectors.groupingBy(User::role, Collectors.counting()));

Map<Role, List<String>> namesByRole = users.stream()
        .collect(Collectors.groupingBy(User::role,
                Collectors.mapping(User::name, Collectors.toList())));

// Partitioning (boolean key)
Map<Boolean, List<Integer>> evensOdds = nums.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));

// Joining strings
String csv = names.stream().collect(Collectors.joining(", ", "[", "]"));

// Downstream collectors — composing
Map<Role, Optional<User>> oldestByRole = users.stream()
        .collect(Collectors.groupingBy(
                User::role,
                Collectors.maxBy(Comparator.comparing(User::age))));
```

## Lazy Evaluation

Intermediate ops don't run until a terminal op is invoked. This lets the pipeline fuse operations and skip unnecessary
work.

```java
Stream<Integer> s = Stream.of(1, 2, 3)
        .filter(n -> {
            System.out.println("filter " + n);
            return n > 1;
        })
        .map(n -> {
            System.out.println("map " + n);
            return n * 10;
        });
System.out.

println("pipeline built");
s.

forEach(n ->System.out.

println("forEach "+n));
```

Output:

```
pipeline built
filter 1
filter 2
map 2
forEach 20
filter 3
map 3
forEach 30
```

Each element flows through the entire pipeline one at a time — `filter`, then `map`, then `forEach` — not "filter all,
then map all". This is critical for **short-circuiting**.

## Short-Circuiting

Some operations can stop early without processing all elements:

- `limit(n)`, `findFirst()`, `findAny()`
- `anyMatch`, `allMatch`, `noneMatch`
- `Stream.generate(...).limit(n)` — needed to terminate infinite streams

```java
boolean hasEven = nums.stream()
        .peek(n -> System.out.println("peek " + n))
        .anyMatch(n -> n % 2 == 0);
// Stops after the first even — subsequent elements are not peeked.
```

## Parallel Streams

`.parallelStream()` or `.parallel()` — splits the source, processes chunks in parallel on the **common ForkJoinPool** (
size `nCpu - 1`).

```java
long sum = list.parallelStream().mapToLong(i -> i).sum();
```

**When parallel helps:**

- N is large (typically > 10,000 elements).
- Per-element work is CPU-bound and non-trivial.
- Operation is stateless and order-independent.

**When parallel hurts:**

- Small streams — overhead dominates.
- I/O-bound tasks — blocks the common pool, starving other code.
- `sorted`, `distinct` — require buffering / synchronization.
- Mutable shared state — race conditions, incorrect results.
- `limit` / `findFirst` — must preserve encounter order, limits parallelism.

**Custom pool:** pass a `Collector.ordered()` or use `ForkJoinPool` submit trick — but the right answer is usually to
use an explicit `ExecutorService` for non-CPU work.

```java
// DON'T — blocking on common pool
list.parallelStream().

map(this::fetchFromDb).

toList();

// DO — use CompletableFuture with explicit executor
```

## Common Patterns

### Group and count

```java
Map<String, Long> counts = items.stream()
        .collect(Collectors.groupingBy(Item::category, Collectors.counting()));
```

### Top N

```java
List<User> top10 = users.stream()
        .sorted(Comparator.comparing(User::score).reversed())
        .limit(10)
        .toList();
```

### Build map from list

```java
Map<Long, User> byId = users.stream()
        .collect(Collectors.toMap(User::id, Function.identity(), (a, b) -> a));
```

### Convert list of objects to list of one field

```java
List<String> names = users.stream().map(User::name).toList();
```

### Filter and find first

```java
Optional<User> admin = users.stream()
        .filter(u -> u.role() == Role.ADMIN)
        .findFirst();
```

### Sum of field

```java
int totalAge = users.stream().mapToInt(User::age).sum();
```

### Chained conditions

```java
List<Order> recent = orders.stream()
        .filter(o -> o.status() == Status.SHIPPED)
        .filter(o -> o.shippedAt().isAfter(LocalDate.now().minusDays(7)))
        .sorted(Comparator.comparing(Order::shippedAt).reversed())
        .toList();
```

## Primitive Streams

`IntStream`, `LongStream`, `DoubleStream` — avoid autoboxing overhead:

```java
int sum = ints.stream().mapToInt(Integer::intValue).sum();   // primitive
double avg = vals.stream().mapToDouble(Double::doubleValue).average().orElse(0);
IntStream.

range(0,100).

forEach(System.out::println);
```

Supports `sum()`, `average()`, `max()`, `min()` directly without a `Collector`.

## Common Pitfalls

1. **Reusing a stream** — streams are single-use. `IllegalStateException` on second terminal op. Build the stream again
   from the source.

2. **Side effects in lambdas** — mutating shared state in `map`/`filter` is broken under parallel streams. Use
   `collect` / `reduce` instead.

3. **`peek` for production logic** — `peek` is documented as for debugging. Relying on it for side effects is fragile;
   behavior may differ across stream variants.

4. **`count()` after `map`** — Java 9+ short-circuits: `stream.map(f).count()` doesn't actually map if it can determine
   the count from the source size. Avoid relying on `map` side effects.

5. **`Collectors.toMap` with duplicate keys** — throws `IllegalStateException`. Always provide a merge function when
   keys may collide.

6. **`forEach` in parallel streams** — order is non-deterministic. Use `forEachOrdered` if you need encounter order (
   loses parallelism benefit).

7. **Large `sorted` on infinite stream** — `Stream.iterate(...).sorted().limit(n)` will hang; `sorted` requires all
   elements before producing any. Use `Stream.iterate(...).limit(n).sorted()` instead.

8. **Closing streams from I/O** — `Files.lines()` and similar hold resources. Use try-with-resources:
   ```java
   try (Stream<String> lines = Files.lines(path)) {
       lines.filter(...).forEach(...);
   }
   ```

9. **Confusing `findFirst` and `findAny`** — `findFirst` preserves encounter order (slower on parallel); `findAny`
   returns any element (faster).

10. **Parallel stream for blocking I/O** — blocks the common pool. Use `CompletableFuture` with an explicit executor.

## Code Examples

### Word frequency

```java
Map<String, Long> freq = Files.lines(path)
        .flatMap(line -> Arrays.stream(line.split("\\s+")))
        .map(String::toLowerCase)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

### Matrix flattening

```java
int[][] matrix = {{1, 2}, {3, 4}};
List<Integer> flat = Arrays.stream(matrix)
        .flatMapToInt(Arrays::stream)
        .boxed()
        .toList();
```

### Nested grouping

```java
Map<Role, Map<Department, List<User>>> byRoleThenDept = users.stream()
        .collect(Collectors.groupingBy(User::role,
                Collectors.groupingBy(User::department)));
```

### Reduce to build a string

```java
String result = words.stream().reduce("", (a, b) -> a + " " + b);
// or use Collectors.joining — preferred for strings
```

## Common Interview Questions

1. **What is a Stream in Java?**
   → A pipeline of operations on a data source. Not a data structure, doesn't store elements, lazy, single-use,
   functional.

2. **What's the difference between `Collection` and `Stream`?**
   → Collection stores elements; Stream processes them. Collection is eager; Stream is lazy until a terminal op. Stream
   is single-use; Collection is reusable.

3. **What's the difference between `map` and `flatMap`?**
   → `map` transforms 1:1. `flatMap` transforms 1:n (each element → a stream) and flattens the result. `flatMap` is used
   for nested collections.

4. **What are intermediate vs terminal operations?**
   → Intermediate (`filter`, `map`, `sorted`) return a new Stream and are lazy. Terminal (`collect`, `forEach`,
   `reduce`) trigger execution and produce a result. A pipeline must have exactly one terminal op.

5. **Why are streams lazy?**
   → Allows fusing operations (one pass), short-circuiting (`findFirst`, `limit`), and processing infinite streams.
   Without a terminal op, no work happens.

6. **What is short-circuiting in streams?**
   → An operation that can produce a result without processing all elements: `limit`, `findFirst`, `findAny`,
   `anyMatch`, `allMatch`, `noneMatch`.

7. **What's the difference between `findFirst` and `findAny`?**
   → `findFirst` returns the first element in encounter order (slower on parallel streams, must preserve order).
   `findAny` returns any element (faster, non-deterministic on parallel).

8. **How do `Collectors.toMap` handle duplicate keys?**
   → By default, throws `IllegalStateException` on collision. Provide a merge function as the third arg: `(a, b) -> a`
   to keep the existing, `(a, b) -> b` to overwrite.

9. **When should you NOT use parallel streams?**
   → Small streams (overhead dominates), I/O-bound tasks (blocks common pool), operations requiring order (`sorted` +
   `limit`), mutable shared state.

10. **What's the common ForkJoinPool problem?**
    → `parallelStream` and default `CompletableFuture` use the shared common pool (size `nCpu - 1`). Blocking operations
    there starve unrelated code. Use an explicit executor for blocking work.

11. **Can a stream be reused?**
    → No. After a terminal op, the stream is closed. Reuse throws `IllegalStateException`. Build a new stream from the
    source.

12. **How do you handle checked exceptions in stream lambdas?**
    → Lambdas can't throw checked exceptions. Wrap in a try-catch inside the lambda, extract to a method that wraps the
    exception, or use a library like `Unchecked` from Guava. Common pattern:
    ```java
    .map(obj -> { try { return transform(obj); } catch (IOException e) { throw new UncheckedIOException(e); } })
    ```

## Related

- `Java/modern-java/lambda-expressions.md` — lambdas, method references
- `Java/modern-java/optional.md` — `Optional`, returned by many terminal ops
- `Java/concurrency/executors-and-thread-pools.md` — ForkJoinPool, parallel streams
- `Java/core/collections-hashmap-internals.md` — `HashMap` used internally by `groupingBy`
- `Engineering-Practices/Clean-Code/` — readability of stream pipelines

## Resources

- **Java docs:** `java.util.stream`, `java.util.stream.Collectors`
- **"Modern Java in Action" (Urma, Fusco, Mycroft)** — definitive Stream API reference
- **Tagir Valeev, "Stream API"** — talks and blog posts on pitfalls
- **Baeldung:** https://www.baeldung.com/java-streams
