# HashMap Internals

The single most-asked Java question on interviews. Covers data structures, hashing, complexity, and Java 8+
optimizations.

## Overview

`HashMap<K, V>` is a hash-table based implementation of `Map<K, V>`. Provides **O(1)** average time complexity for
`get`/`put`, degrades to **O(log n)** with tree buckets (Java 8+) and **O(n)** in the worst case (poor hash distribution
or targeted attacks).

Not thread-safe. For concurrent use → `ConcurrentHashMap`. For sorted iteration → `TreeMap`. For insertion-order →
`LinkedHashMap`.

## Internal Structure

```
HashMap
├── Node<K,V>[] table            // bucket array (length is power of 2)
├── size                        // number of key-value pairs
├── loadFactor                  // default 0.75
└── threshold                   // capacity * loadFactor, triggers resize
```

A bucket is either a **linked list** of `Node` objects or a **red-black tree** of `TreeNode` objects (when bucket size ≥
8 AND table size ≥ 64).

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next;
}
```

## Default Constants

| Constant                   | Value | Purpose                                                           |
|----------------------------|-------|-------------------------------------------------------------------|
| `DEFAULT_INITIAL_CAPACITY` | 16    | Initial bucket array size                                         |
| `DEFAULT_LOAD_FACTOR`      | 0.75  | Resize when `size > capacity × loadFactor`                        |
| `TREEIFY_THRESHOLD`        | 8     | Min bucket size to convert list → tree                            |
| `UNTREEIFY_THRESHOLD`      | 6     | Min bucket size to convert tree → list (on resize/remove)         |
| `MIN_TREEIFY_CAPACITY`     | 64    | Min table size before treeification is allowed (else just resize) |
| `MAXIMUM_CAPACITY`         | 2³⁰   | Hard cap on table size                                            |

Pre-sizing formula: `HashMap(int initialCapacity)` does NOT immediately allocate the array — the table stays `null`
until the first `put`. `tableSizeFor(int)` returns the next power of 2 ≥ the argument, applied lazily on first
insertion.

## How `put(key, value)` Works

1. **Hash the key:** `h = key.hashCode(); h ^ (h >>> 16)` — spreads higher bits into lower to reduce collisions when
   table is small.
2. **Compute index:** `i = (table.length - 1) & hash` — bitwise AND is faster than modulo; works because `table.length`
   is always a power of 2.
3. **Locate bucket at index `i`:**
    - Empty → create new `Node`, place it.
    - Same key (by `hash` AND `equals`) → replace value, return old.
    - Otherwise → traverse the linked list/tree. If a matching key found, replace; else append.
4. **Treeify check:** if the resulting list size ≥ 8 AND table size ≥ 64 → convert list to red-black tree.
5. **Resize check:** if `size > threshold` → resize table (double capacity).

## How `get(key)` Works

1. Hash the key, compute index.
2. Check first node in bucket — match by `hash` AND `equals`.
3. If first node doesn't match and `next != null`:
    - If bucket is a tree → tree search O(log n).
    - Else → traverse linked list O(n).
4. Return value or `null`.

## Resize (Rehashing)

Triggered when `size > threshold` after a `put`. Steps:

1. Create a new table with **double capacity** (e.g., 16 → 32).
2. Re-hash every existing node and place into the new table.
3. Old table becomes garbage.

Because capacity is always a power of 2, each node's new index is either `oldIndex` or `oldIndex + oldCapacity` (no full
re-hash needed, just one bit check). This is the main reason for the power-of-2 constraint.

**Cost:** O(n). Frequent resizing is a real performance issue — always pre-size if you know the cardinality:

```java
new HashMap<>(expectedSize);  // HashMap rounds up to: expectedSize / 0.75 + 1
```

## Why Load Factor 0.75?

Trade-off between **space** and **collision probability**:

- Higher (1.0) → more collisions, slower lookups.
- Lower (0.5) → wastes memory, resizes more often.
- 0.75 is empirically the sweet spot (Poisson distribution analysis in OpenJDK).

## equals / hashCode Contract

**Contract:** if `a.equals(b)` → then `a.hashCode() == b.hashCode()`. The reverse is NOT required (collisions allowed).

**Consequences of violation:**

- Two equal objects could land in **different buckets** → `get` returns `null` even though the key exists.
- Map silently breaks; debug hell.

**Always override both together.** Use `Objects.equals` and `Objects.hash`:

```java
public final class User {
    private final String email;
    private final long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return id == user.id && email.equals(user.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, email);
    }
}
```

**Pitfall:** using a mutable field in `hashCode` → object's hash changes after insertion → lost in the map. Make
`hashCode` based on **immutable** fields only.

## Java 8+ Treeification

When a single bucket accumulates ≥ 8 entries **and** table size ≥ 64, the linked list is converted to a **red-black tree
** (`TreeNode`). This protects against:

- Hash collisions from poorly-written `hashCode` (e.g., constant return).
- Hash-flooding DoS attacks (sending many keys with the same hash).

Tree lookup is **O(log n)** vs O(n) for a list — much safer worst case. Note: tree nodes take ~2x memory, so it's a
deliberate trade-off.

### Why Treeify Threshold is 8

OpenJDK's comment references a **Poisson distribution** with λ=0.5 for bucket sizes under a well-distributed hash
function:

```
P(0) = 0.60653   P(1) = 0.30327   P(2) = 0.07582
P(3) = 0.01264   P(4) = 0.00158   P(5) = 0.00016
P(6) = 0.00001   P(7) = 0.00000   P(8) = 0.00000006
```

With a uniform hash, hitting 8 entries in one bucket has probability ~6×10⁻⁸ — so treeification only triggers when the
hash function is broken or under attack. Threshold 8 avoids paying the tree overhead in normal operation.

### Untreeification

On `resize` or `remove`, if a tree bucket drops to ≤ 6 entries (`UNTREEIFY_THRESHOLD`), it's converted back to a linked
list. The gap between 8 and 6 prevents flapping between list and tree on repeated inserts/removes at the boundary.

## ConcurrentHashMap Internals (Java 8+)

Worth knowing separately — interviewers often ask "how does CHM differ internally from HashMap?"

**Structure:** same `Node[] table`, but:

- `Node.val` and `Node.next` are **`volatile`** → reads are visible across threads without locking.
- First node of each bucket can be locked with `synchronized` only during write contention; otherwise writes use **CAS
  ** (`Unsafe.compareAndSwapInt`) on the bucket slot.
- A **`ForwardingNode`** marks buckets that have already been migrated during a concurrent resize — readers follow it to
  the new table.
- **`ReservationNode`** — placeholder inserted via CAS when a `computeIfAbsent` computation is in progress, blocking
  concurrent compute on the same key.

**Concurrent resize:** multiple threads help migrate buckets in parallel (transfer protocol). Unlike HashMap, CHM never
throws `ConcurrentModificationException` and its iterators are **weakly consistent** (reflect the map state at iterator
creation, may not see later updates, never throw).

**Null keys/values forbidden:** nulls would be ambiguous under concurrency — `get(k)` returning `null` couldn't
distinguish "absent" from "mapped to null" because the value could change between `containsKey` and `get`. HashMap
sidesteps this by being single-threaded.

## Historical: Java 7 HashMap Resize Infinite Loop

Under concurrent use (which `HashMap` explicitly does NOT support), Java 7's `transfer()` could form a **circular linked
list** during resize. Subsequent `get` would spin forever on the cycle → 100% CPU on a thread. This was a real
production incident class.

Java 8's resize explicitly breaks the cycle (tail handling), but **concurrent mutation still corrupts the map silently
** — use `ConcurrentHashMap`, never `HashMap` + external synchronization on hot paths.

## modCount and Fail-Fast Iterators

`HashMap` keeps a `modCount` counter — incremented on every structural modification (put/remove/resize). Iterators
capture this counter at creation; on each `next()` they compare the stored value with the live one and throw
`ConcurrentModificationException` if they differ.

**Best-effort, not hard guarantee:** race conditions can miss a modification. Never use CME for program correctness —
treat it as a debugging aid.

`ConcurrentHashMap` iterators are **weakly consistent**: they traverse the bucket array once at creation, may or may not
reflect concurrent updates, and never throw CME.

## View Operations

`keySet()`, `values()`, `entrySet()` return **live views** backed by the map — not copies. Changes to the map are
reflected in the views; `iterator().remove()` on a view removes the underlying entry.

```java
Map<String, Integer> m = new HashMap<>();
m.

put("a",1);

Set<String> keys = m.keySet();
keys.

remove("a");             // map is now empty
keys.

add("b");                // ❌ UnsupportedOperationException — view can't create mappings
```

`Collections.unmodifiableMap(map)` wraps the map so view mutations throw.

## HashMap vs Hashtable vs ConcurrentHashMap

| Feature          | HashMap        | Hashtable                    | ConcurrentHashMap               |
|------------------|----------------|------------------------------|---------------------------------|
| Thread-safe      | ❌              | ✅ (all methods synchronized) | ✅ (per-bucket locking, Java 8+) |
| Null keys/values | ✅ (1 null key) | ❌                            | ❌                               |
| Performance      | Fastest        | Slowest (coarse lock)        | Fast (fine-grained)             |
| Legacy?          | No             | Yes (legacy)                 | No (preferred)                  |
| Iterators        | Fail-fast      | Fail-fast                    | **Weakly consistent**           |

For concurrent code, **always prefer `ConcurrentHashMap`**. `Hashtable` is legacy;
`Collections.synchronizedMap(new HashMap<>())` is a coarse lock on every operation.

## Common Pitfalls

1. **Using mutable fields in `hashCode`** → lost entries.
2. **Assuming insertion order** → use `LinkedHashMap`.
3. **Assuming sorted iteration** → use `TreeMap`.
4. **Not pre-sizing in hot paths** → expensive resizes.
5. **Concurrent modification** → `ConcurrentModificationException`. Use `ConcurrentHashMap` or wrap reads with proper
   synchronization.
6. **Boxing overhead** in `Map<Integer, V>` with autoboxing in tight loops → consider `Int2ObjectMap` (Eclipse
   Collections) or primitives-friendly libs.

## Code Examples

### Basic Usage

```java
Map<String, Integer> wordCount = new HashMap<>();
wordCount.

put("hello",1);
wordCount.

merge("hello",1,Integer::sum);  // "hello" → 2
System.out.

println(wordCount.getOrDefault("world", 0));  // 0
```

### Pre-sizing

```java
int expected = 1_000_000;
int capacity = (int) (expected / 0.75f) + 1;
Map<String, User> cache = new HashMap<>(capacity);
```

### Compute Patterns (Java 8+)

```java
map.computeIfAbsent(key, k ->

expensiveCreate(k));
        map.

computeIfPresent(key, (k, v) ->v +1);
        map.

merge(key, 1,Integer::sum);  // increment counter
```

### Why Not `Hashtable`

```java
Map<K, V> safe = Collections.synchronizedMap(new HashMap<>());
// vs
Map<K, V> better = new ConcurrentHashMap<>();
// `better` allows concurrent reads and per-bucket writes.
```

## Common Interview Questions

1. **How does HashMap work internally?**
   → Array of buckets, each is a list/tree, hashing distributes keys across buckets, collisions handled by
   chaining/tree.

2. **Why is HashMap's capacity always a power of 2?**
   → To use `& (n - 1)` instead of `% n` for index, and to enable efficient resize (each node moves to either old or
   old + n index).

3. **What happens if two keys have the same hashCode?**
   → They land in the same bucket. Stored in the list/tree at that index. `equals` is used to find the exact match
   during `get`.

4. **Why is ConcurrentHashMap faster than Hashtable?**
   → Hashtable locks the whole map for every operation. ConcurrentHashMap uses per-bucket locks (Java 7) or CAS +
   synchronized per first-node (Java 8+), allowing concurrent writes to different buckets.

5. **What is the difference between `==`, `equals`, and `hashCode`?**
   → `==` compares references (or primitive values). `equals` compares logical value (override-defined). `hashCode`
   returns an int derived from object's state; contract: equal objects must have equal hashes.

6. **Can you store a null key in ConcurrentHashMap?**
   → No. `HashMap` allows one null key; `ConcurrentHashMap` and `Hashtable` don't.

7. **What is the fail-fast iterator behavior?**
   → `HashMap`'s iterator throws `ConcurrentModificationException` if the map is structurally modified during
   iteration (modCount check). Use `ConcurrentHashMap` for safe concurrent iteration.

8. **How do you make a HashMap immutable for use as a key?**
   → Use a record (Java 16+) or a class with all `final` fields, override `equals`/`hashCode` based only on those
   fields, prevent subclassing.

9. **What is the default initial capacity and load factor?**
   → 16 and 0.75. Resize triggers when `size > capacity × 0.75` (e.g., 12 entries for capacity 16).

10. **Why is the treeify threshold exactly 8?**
    → OpenJDK's Poisson analysis shows that with a uniform hash, the probability of 8 entries in one bucket is ~6×10⁻⁸ —
    so treeification only triggers when the hash function is broken. Threshold 8 avoids paying tree overhead in normal
    operation.

11. **What's the difference between fail-fast and weakly consistent iterators?**
    → Fail-fast (`HashMap`) throws `ConcurrentModificationException` on detected structural change during iteration (via
    `modCount`). Weakly consistent (`ConcurrentHashMap`) traverses once at creation, may miss later updates, never
    throws.

12. **What happens when you put a `null` key into a HashMap?**
    → `null.hashCode()` is treated as 0, so the null key always lands in bucket 0. Only one null key is allowed (a
    second `put(null, v)` overwrites the first).

13. **Why was the Java 7 HashMap resize a production hazard?**
    → Under concurrent modification (unsupported), `transfer()` could form a circular linked list → `get` spun forever
    at 100% CPU. Java 8's resize breaks the cycle, but `HashMap` still corrupts under concurrency — use
    `ConcurrentHashMap`.

14. **How does `ConcurrentHashMap` differ internally from `HashMap` (Java 8+)?**
    → Volatile `val`/`next` on nodes, CAS for uncontended writes, `synchronized` on the bucket's first node only under
    contention, `ForwardingNode` for concurrent resize, parallel transfer protocol. No null keys/values (ambiguity under
    concurrency).

## Related

- `Java/core/collections-overview.md` — full Collections framework
- `Java/core/equals-and-hashcode.md` — contract deep dive
- `Java/modern-java/collections-factory-methods.md` — `Map.of`, `Map.copyOf`
- `Java/concurrency/concurrent-collections.md` — `ConcurrentHashMap` details
- `CS-Fundamentals/Complexity-Analysis/big-o-notation.md` — complexity of operations

## Resources

- **OpenJDK source:** `java.util.HashMap` — the canonical reference
- **Effective Java (Joshua Bloch), Item 11:** `equals`/`hashCode` contract
- **Aleksey Shipilëv:** "HashMap Performance" (JokerConf talk)
- **Baeldung:** https://www.baeldung.com/java-hashmap
