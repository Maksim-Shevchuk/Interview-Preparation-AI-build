# NoSQL Performance Issues — Diagnosis and Resolution

NoSQL databases trade SQL's rich query model and strong consistency for horizontal scale, flexible schema, and
throughput — but they introduce their **own characteristic performance pathologies**. This file is a field guide to
the common issues and concrete fixes, organised by symptom (hotspot, latency, throughput drop) and by engine
(MongoDB, Cassandra, DynamoDB, Redis, Elasticsearch).

A frequent interview topic at senior-backend level — expect to discuss read amplification, hot partitions, working
set vs RAM, LSM-tree compaction, and the trade-offs of secondary indexes.

---

## Quick Reference — Symptoms ➜ Likely Cause ➜ Fix

| Symptom                                | Likely cause                                         | First fix                                   |
|----------------------------------------|------------------------------------------------------|---------------------------------------------|
| Single shard/partition at 100% CPU     | Hot partition key (skewed write/read distribution)   | Salt the key; pre-split; rewrite access     |
| Read latency spike after data growth   | Working set no longer fits in RAM                    | Add RAM; shard; cache in Redis              |
| `COLLSCAN` / full-collection scans     | Missing or unused index                              | Add covering/compound index                 |
| Slow aggregation                       | Pipeline scans too many docs early                   | Add `$match` first; use indexes             |
| Write timeouts in Cassandra            | Tombstones, oversized partitions, or read-repair     | Tune compaction; fix data model             |
| DynamoDB `ProvisionedThroughputExceeded` | Throttling on a hot partition                     | Redesign key; switch to on-demand           |
| Redis latency spikes                   | Big `KEYS`, slow script, `SAVE` fork, big values     | `SCAN`, pipeline, snapshot with repl-diskless |
| ES search latency, GC churn            | Too many shards, mappings explode, deep aggregations | Force-merge, dense mappings, `size: 0`      |
| Replication lag growing                | Secondary can't keep up with writes                  | Throttle writes; bigger secondary; batch    |
| CPU 100% on all nodes                  | Compaction / GC / index rebuild storm                | Schedule during off-peak; tune LSM          |

---

## 1. Root Causes — A Mental Model

Most NoSQL perf problems boil down to **one of six root causes**:

```
   ┌────────────────────────────────────────────────────────────────────┐
   │                    NoSQL PERFORMANCE ISSUES                        │
   ├────────────────────────────────────────────────────────────────────┤
   │ 1. DATA MODEL      │ "You used it like an RDBMS" — joins,          │
   │                    │  normalisation, ad-hoc filters.                │
   │ 2. HOTSPOTS        │ Uneven distribution across partitions/shards. │
   │ 3. INDEXING        │ Missing indexes; wrong order; over-indexing.  │
   │ 4. WORKING SET     │ Active data > RAM → disk reads dominate.      │
   │ 5. ENGINE INTERNALS│ LSM compaction, B+tree splits, GC, forks.     │
   │ 6. NETWORK / OPS   │ Replication lag, cross-region, conn pool.     │
   └────────────────────────────────────────────────────────────────────┘
```

Unlike SQL — where you usually start with `EXPLAIN` — NoSQL debugging starts with **the data model and key design**.
You can't index your way out of a bad partition key.

---

## 2. Data-Model Issues

NoSQL engines are **not** SQL engines with no schema. The query patterns must be designed **up front** and embedded
into the data layout. Treating them like Postgres leads to:

### 2.1 Joins / cross-document lookups

Most NoSQL engines either forbid joins (Cassandra, DynamoDB) or only support them as expensive operations (MongoDB
`$lookup`, ES parent/child). A "join" in a NoSQL world means N+1 queries.

**Fix — denormalise for read patterns:**

```js
// Bad — 2 round trips per order, every time
db.orders.find({...}).forEach(o => db.users.findOne({_id: o.userId}))

// Good — embed the frequently-needed fields
db.orders.findOne({...})
// → { _id, total, user: { id, name, email } }   // denormalised snapshot
```

Trade-off: denormalised copies must be **updated** when the source changes. Use change streams / CDC to keep them
eventually consistent.

### 2.2 Unbounded arrays / large documents

MongoDB documents, Cassandra rows, DynamoDB items — all have size limits (Mongo: 16 MB; DynamoDB: 400 KB). Even
within limits, large documents cause:

- Slower wire transfer.
- Larger working-set in RAM.
- Document moves on update (WiredTiger rewrites the whole doc).
- More contention.

**Fix:** cap arrays, move unbounded data to a separate collection, use the **bucket pattern** (MongoDB) for
time-series.

### 2.3 Schema explosion (Elasticsearch)

Dynamically mapped fields in ES explode the number of unique columns → "mapping explosion" → huge heap usage, slow
cluster.

**Fix:**

```json
PUT index/_mapping
{
  "dynamic": "strict",          // reject unknown fields
  "dynamic_templates": [ /* explicit rules */ ]
}
```

### 2.4 Wrong access pattern for the engine

| If your reads are…               | Pick the engine with the right layout                          |
|----------------------------------|-----------------------------------------------------------------|
| Single-key lookups               | Key-value (Redis, DynamoDB)                                     |
| Document by attributes           | Document store (MongoDB)                                        |
| Time-series by entity + range    | Wide-column (Cassandra, ScyllaDB) — partition by entity         |
| Graph traversal                  | Graph DB (Neo4j)                                                |
| Full-text / faceted search       | Inverted index (Elasticsearch, OpenSearch)                      |

A common interview answer: *"In NoSQL you design the schema for the query, not the other way around."*

---

## 3. Hotspots / Uneven Distribution

The most common NoSQL killer. Hash partitioning evens out random keys but **cannot save you from skewed access
patterns**: a celebrity's Twitter handle, the most popular product, "all orders today go to one shard".

### 3.1 MongoDB — hot shard

Causes:
- **Monotonic `_id`** (e.g., `ObjectId` is time-based) — all new inserts hit the same shard.
- A shard key that does not spread writes (`{ region: "EU" }` when 90% of users are in EU).

Fixes:
- Use a **hashed shard key** for monotonic fields: `sh.shardCollection("db.orders", { _id: "hashed" })`.
- Use a **compound key** with high-cardinality prefix: `{ userId: 1, createdAt: 1 }`.
- **Pre-split** chunks to avoid the migration storm on a fresh cluster.

### 3.2 Cassandra — hot partition

Causes:
- Partition key with **low cardinality** (e.g., partition by `country` → Italy has 60M users, Malta 0.5M).
- **Time-series with partition = bucket**: every write for "today" lands on one node.

Fixes:
```sql
-- Bucketing by time + salt
CREATE TABLE events (
  bucket_date date,
  salt        int,
  event_id    timeuuid,
  ...
  PRIMARY KEY ((bucket_date, salt), event_id)
) WITH CLUSTERING ORDER BY (event_id DESC);
```
Salt the partition key with `0..N` and read across all `N` partitions in parallel.

Rules:
- Cardinality: high enough to spread.
- Size: keep partitions under **~100 MB** and **< 100k rows**.

### 3.3 DynamoDB — hot partition

DynamoDB distributes data across partitions using the partition key. A single hot key (e.g., a viral product page)
can saturate one partition's **3000 RCUs / 1000 WCUs** and trigger `ProvisionedThroughputExceeded` even when overall
capacity is fine.

Fixes:
- **Spread the writes** by suffixing the key with a random number (`product-123#0..9`); aggregate on read.
- Move to **on-demand** billing (pay-per-request) to absorb bursts.
- Use **DAX** (DynamoDB Accelerator) for microsecond-read caching of hot items.
- Avoid "hot" GSI keys — GSIs are subject to the same partition limits.

### 3.4 Redis — single-threaded hotspot

Redis is single-threaded for command execution. A single **slow command** (`KEYS *`, `SMEMBERS` on a set with 10M
members, `LRANGE 0 -1` on a 10MB list) **blocks the whole server**.

Fixes:
- Replace `KEYS` with `SCAN` (cursor-based).
- Keep individual values small (< 100 KB); store big objects in chunks or in another store.
- Use **pipelining** / **Lua scripts** to round-trip many commands in one RTT.
- Move CPU-heavy work (sorts, set ops) to the client side.

---

## 4. Indexing Issues

NoSQL indexes are not free — each extra index slows writes and consumes RAM. The rules are similar to SQL but the
defaults differ per engine.

### 4.1 MongoDB

```js
// Covered query — index alone can answer; no FETCH stage
db.orders.createIndex({ userId: 1, status: 1, total: 1 });

db.orders.find({ userId: "u1", status: "PAID" }, { total: 1, _id: 0 });
// → IXSCAN only; no FETCH, no doc read
```

ESR rule (Equality, Sort, Range) for compound index column order:

```
db.orders.createIndex({ status: 1, createdAt: -1, amount: 1 });
//                  equality ──┘ sort ─────────┘ range ───────┘
```

Verify with `explain("executionStats")`:

```
winningPlan.stage = "IXSCAN"     ✓ good
winningPlan.stage = "COLLSCAN"   ✗ full collection scan
totalDocsExamined >> nReturned   ✗ index not selective enough
executionTimeMillis              ✗ too high → investigate
```

### 4.2 Cassandra — secondary indexes are dangerous

Cassandra's `SAI` (Storage-Attached Indexing) and legacy `2i` indexes query **every node** in the ring for
non-key predicates — they don't scale on high-cardinality columns.

Rules:
- Use partition key + clustering columns for the hot path.
- Materialise lookup tables for reverse queries:

```sql
CREATE TABLE user_by_email (email text PRIMARY KEY, user_id uuid);
CREATE TABLE user_by_phone (phone text PRIMARY KEY, user_id uuid);
```

- For analytics, push data to Spark/Elasticsearch/OLAP, not to C* secondary indexes.

### 4.3 DynamoDB — LSIs vs GSIs

| Type | Key schema                       | Limit                    | Consistency               |
|------|----------------------------------|--------------------------|---------------------------|
| LSI  | Same partition key + sort        | 10 per table, ≤ 10 GB    | Strong or eventual        |
| GSI  | Any attributes as partition/sort | 20 per table             | **Eventual only**         |

GSIs have **their own** capacity and are subject to the same hot-partition rules. A "hot" GSI can throttle writes
to the **base table**.

### 4.4 Elasticsearch — too many / wrong mappings

Each `text` field gets a backing inverted index + doc values + norms. The cost compounds with field count.

```json
PUT products
{
  "mappings": {
    "properties": {
      "description": { "type": "text" },                 // indexed + analysed
      "id":          { "type": "keyword" },              // exact match, aggregatable
      "price":       { "type": "scaled_float", "scaling_factor": 100 },
      "tags":        { "type": "keyword" }
    }
  }
}
```

Disable `doc_values`, `_source`, `norms`, `index` where you don't need them.

---

## 5. Working Set vs RAM

> **The single most important metric in any disk-backed NoSQL store: does the working set fit in RAM?**

When the **active** subset of data (hot documents + indexes) exceeds RAM, the engine starts hitting disk on most
requests → latency jumps from microseconds to milliseconds.

### Diagnose

- **MongoDB** — `serverStatus.wiredTiger.cache`. Look at `bytes currently in the cache` vs `maximum bytes configured`,
  and `pages requested from the cache` (should be low).
- **Cassandra** — `TableMetrics`/`nodetool tablestats` — look at `Space used (total)` vs `Space used by snapshots`.
  Compare with `CacheHitRate`.
- **Redis** — `INFO memory`. `used_memory` vs `maxmemory`, `evicted_keys` rate.
- **ES** — `_nodes/stats/os,indices`. Field-data cache and shard-level memory.

### Fixes

| Strategy               | When to apply                                  |
|------------------------|------------------------------------------------|
| **Add RAM**            | Cheapest short-term fix                        |
| **Shard / partition**  | Spread working set across nodes                |
| **Cache (Redis/DAX)**  | Hot 1% read repeatedly                         |
| **TTL / archive**      | Move cold data to cold storage (S3, Glacier)   |
| **Compression**        | Snappy/LZ4 shrinks working set 3–5x            |
| **Capped collections** | MongoDB rolling log — fixed-size, FIFO         |

---

## 6. Engine-Internal Issues

### 6.1 LSM-tree compaction (Cassandra, ScyllaDB, RocksDB, HBase)

LSM writes append to an in-memory `MemTable`, flush to `SSTables` on disk, and **compact** periodically. Write
amplification, read amplification, and space amplification are the costs.

Symptoms of compaction problems:
- CPU spike at predictable times.
- Disk usage creeps upward (compaction can't keep up).
- Read latency increases (more SSTables to consult per read).

Fixes:
- Pick the right **compaction strategy**:

| Strategy        | Best for                                |
|-----------------|-----------------------------------------|
| `SizeTiered`    | Write-heavy, time-series append         |
| `Leveled`       | Read-heavy, predictable read latency    |
| `TimeWindow`    | Time-series with TTL-driven expiry      |

- Tune `compaction_throughput_mb_per_sec`.
- Avoid **tombstones**: deletes are writes; many tombstones → read repair & slowdowns. Run `nodetool garbagecollect`
  periodically.
- Watch `nodetool compactionstats`.

### 6.2 B+tree splits & checkpoints (MongoDB WiredTiger)

- WiredTiger uses **MVCC checkpoints** every 60s by default. A large write batch right at checkpoint time can cause a
  latency spike.
- Heavy `update` on large documents causes **document moves** (WT rewrites the whole doc).
- Tune `wiredTiger.cacheSizeGB`, `storage.wiredTiger.engineConfig.journalCompressor`.

### 6.3 GC and forks (JVM-based engines: Cassandra, ES, HBase)

- **Long GC pauses** cause false **node-down** detection, then unnecessary repairs.
- Use **G1GC** (Java 11+) or **ZGC/Shenandoah** for low-pause needs.
- Heap size: **8–16 GB sweet spot** for ES; bigger is not better due to GC pauses.

### 6.4 Redis `BGSAVE` fork

- On a multi-GB dataset, `fork()` to save a snapshot can block the main thread for tens or hundreds of ms.
- Use **`repl-diskless yes`** and `activedefrag yes` on systems with `Transparent Huge Pages` disabled.

---

## 7. Query-Pattern Issues

### 7.1 MongoDB aggregation pipeline

The pipeline runs left-to-right. Put the **most selective** stages early:

```js
// BAD — sorts 10M docs, then filters
db.orders.aggregate([
  { $sort: { createdAt: -1 } },
  { $match: { status: "PAID" } },
  { $limit: 10 }
]);

// GOOD — match first using the index, then sort
db.orders.aggregate([
  { $match:  { status: "PAID" } },     // uses index { status: 1, createdAt: -1 }
  { $sort:   { createdAt: -1 } },
  { $limit:  10 }
]);
```

Use `$project` early to shrink documents flowing through the pipeline. Add `allowDiskUse: true` for big pipelines
and investigate if it triggers — disk spills are slow.

### 7.2 Cassandra — avoid `ALLOW FILTERING`

```sql
-- Bad — coordinator must scan every partition and filter
SELECT * FROM users WHERE age > 18 ALLOW FILTERING;

-- Good — design a table that answers the query directly
SELECT * FROM users_by_country WHERE country = 'FR' AND age > 18;
```

### 7.3 Elasticsearch — slow aggregations

```
GET orders/_search
{
  "size": 0,                          // we want aggregations, not hits
  "aggs": {
    "by_day": {
      "date_histogram": { "field": "createdAt", "calendar_interval": "day" },
      "aggs": { "revenue": { "sum": { "field": "total" } } }
    }
  }
}
```

- `size: 0` when you don't need hits.
- Use **`keyword`** for terms aggregations, never analysed `text`.
- Set `"collect_mode": "breadth_first"` for high-cardinality buckets.
- Consider **rollups** / **transforms** for pre-computed dashboards.

---

## 8. Concurrency, Connection & Network Issues

### 8.1 Connection pool exhaustion

Each connection consumes memory on both client and server. Symptoms: timeouts, refused connections, OOM kills.

- Set **max pool size** based on the Little's law estimate: `poolSize ≈ throughput × avg_query_latency`.
- Use **serverless / connection pooling** (RDS Proxy, PgBouncer, MongoDB SRV with sharding).
- Cassandra: cap `max_connections_per_host` and `max_requests_per_connection`.

### 8.2 Write contention / lock waits

- MongoDB WiredTiger uses **document-level** locking — but updates to the **same document** still serialise.
  Counter-style hot updates (`{ $inc: { views: 1 } }` on a single doc) are a classic bottleneck.
  Fix: **pre-aggregate** into buckets (`views_5s`, `views_1m`) or push to Kafka.
- Redis: a single Lua script blocks other commands; keep scripts **sub-millisecond**.
- ES: `?refresh=wait_for` or explicit `refresh()` after every write is ruinous — bulk-write and refresh
  periodically.

### 8.3 Replication lag

A common cause of "I wrote it, but I can't read it back".

- Cause 1: **Secondary can't keep up** with write volume.
- Cause 2: **Read-your-writes consistency** not enforced — reads go to a stale secondary.
- Fixes:
  - Use **read-your-writes sessions** (MongoDB ` causalConsistency`, Cassandra `LOCAL_QUORUM`, DynamoDB
    `consistentRead`).
  - Throttle bursty writes; batch them.
  - Right-size the secondaries (CPU, IOPS).
  - Watch `replicationLag` metrics — alert at > 30s.

### 8.4 Cross-region latency

If your app in EU reads from a US-primary DynamoDB table, **every** read pays 100ms+.

- Use **global tables** with multi-region replicas.
- Apply **CQRS**: writes to the home region, reads from local replica (with eventual-consistency acceptance).
- Cache aggressively with CloudFront / Redis.

---

## 9. Diagnostic Toolkit

| Engine       | Tools                                                                                  |
|--------------|----------------------------------------------------------------------------------------|
| MongoDB      | `db.setProfilingLevel(2)`, `explain("executionStats")`, `mongotop`, `mongostat`, Cloud Atlas Profiler |
| Cassandra    | `nodetool cfstats`, `nodetool compactionstats`, `nodetool tpstats`, `tracing on`       |
| DynamoDB     | CloudWatch `ThrottledRequests`, `ConsumedReadCapacityUnits`, DAX metrics              |
| Redis        | `INFO`, `SLOWLOG GET`, `LATENCY DOCTOR`, `MEMORY USAGE key`, RedisInsight             |
| Elasticsearch| `_cluster/health`, `_cat/shards`, `_nodes/hot_threads`, slow log, `_search/profile`   |

General:
- **APM**: Datadog, New Relic, OpenTelemetry for end-to-end transaction tracing.
- **Distributed tracing** to find which NoSQL call dominates latency.

---

## 10. Resolution Playbook

When latency or throughput degrades, work the checklist in this order:

1. **Reproduce the slow query.** Capture the exact operation + parameters.
2. **Check the plan.** `explain`, `EXPLAIN ANALYZE` equivalent, or engine profiler.
3. **Verify the data model.** Does this query match a key/index design? If not, you'll have to redesign — no index
   will save you.
4. **Look for full scans and hotspots.** `COLLSCAN`, `ALLOW FILTERING`, hot partition metrics.
5. **Check the working set vs RAM.** Cache hit ratio, page-fault rate.
6. **Check engine internals.** Compaction, GC, forks, refreshes.
7. **Check concurrency.** Connection pool, lock waits, replication lag.
8. **Then** add index, denormalise, shard, cache, redesign schema — in that order.

---

## 11. Common Interview Questions

1. **What is a hot partition and how do you fix it?**
   A partition that receives a disproportionate share of reads/writes, saturating the node hosting it while others
   idle. Fixes: salt the partition key, switch to a higher-cardinality key, use a hashed shard key, pre-split, or
   move to on-demand billing.

2. **Why are secondary indexes dangerous in Cassandra?**
   Cassandra's secondary indexes (and to a lesser degree SAI) require querying every node in the ring for non-key
   predicates, because the coordinator doesn't know which node owns the matching rows. They don't scale to
   high-cardinality columns. Denormalise via lookup tables instead.

3. **LSM trees vs B+ trees — what are the trade-offs?**
   LSM trees optimise for **write throughput** (append-only) at the cost of read amplification (multiple SSTables to
   consult) and compaction CPU. B+ trees optimise for **read latency** (single root-to-leaf lookup) at the cost of
   write amplification (page splits). Cassandra/RocksDB/HBase use LSM; MongoDB WiredTiger uses B+ (with optional LSM).

4. **What is "tombstone" in Cassandra and why is it a problem?**
   A delete in Cassandra is a write — a special marker called a tombstone. Until compaction removes them, reads must
   consult tombstones to know a value was deleted. Too many tombstones (`tombstone_warn_threshold`) slow reads
   dramatically and can fail them entirely (`tombstone_failure_threshold`).

5. **How do you choose a MongoDB shard key?**
   High **cardinality** (to spread writes), **frequency** (no single dominant value), and **monotonicity** (avoid
   monotonically increasing values that funnel writes to one chunk). Compound or hashed keys are common; default
   `ObjectId` is monotonic → use `_id: "hashed"`.

6. **What is the ESR rule for MongoDB compound indexes?**
   Order fields as **Equality** first, then **Sort**, then **Range**: `{ equalityField: 1, sortField: 1,
   rangeField: 1 }`. This lets the index satisfy equality, deliver results in sort order, and apply range bounds.

7. **DynamoDB: when do you get `ProvisionedThroughputExceeded` despite enough capacity?**
   When a single hot partition exceeds 3000 RCUs / 1000 WCUs even if the table's total provisioned capacity is
   higher. Distribute keys better, switch to on-demand, or front with DAX.

8. **LSI vs GSI in DynamoDB?**
   An LSI uses the **same** partition key as the base table but a different sort key — strong consistency, 10 GB
   limit, 10 per table. A GSI can use any attributes as keys — eventual consistency only, separate capacity, 20 per
   table, but subject to its own hot-partition rules.

9. **What makes a Redis command slow?**
   Anything O(N) where N is large: `KEYS *`, `SMEMBERS` on a 10M-member set, `SORT`, `LRANGE 0 -1` on a long list,
   a heavy Lua script. Redis is single-threaded for command execution, so one slow command blocks everything.

10. **Why does Elasticsearch recommend heap size ≤ 32 GB?**
    Two reasons: (1) the JVM compressed-oops pointer optimisation only works under ~32 GB; (2) Lucene relies on the
    OS file cache for the index itself, so you want to leave 50% of RAM to the OS — too-big heap starves the cache.

11. **How do you handle read-your-writes consistency in a NoSQL app?**
    Use the engine's mechanism: MongoDB causally-consistent sessions, Cassandra `LOCAL_QUORUM` or `SERIAL` reads,
    DynamoDB `ConsistentRead=true`, Redis session pinning. Alternatively, route reads-through-writes through a
    primary replica or Redis cache updated on write.

12. **Working set vs RAM — what does it mean and why does it matter?**
    The working set is the data your hot queries touch repeatedly. If it fits in RAM, latencies stay low (µs–ms).
    If it spills to disk, every read pays a disk seek (ms–10s of ms). "Add RAM" is the cheapest fix until you have
    to shard.

13. **How would you fix a slow MongoDB aggregation pipeline?**
    Reorder stages so the most selective `$match` runs first and uses an index; add `$project` early to shrink
    documents; replace `$lookup` with denormalised fields when possible; check `explain()` for `COLLSCAN` and
    `executionTimeMillis`; enable `allowDiskUse` only if needed — investigate if it triggers.

14. **What is the cardinality/frequency/monotonicity rule for shard/partition keys?**
    **Cardinality**: enough distinct values to spread across partitions. **Frequency**: no value should dominate
    (a celebrity account). **Monotonicity**: avoid ever-increasing keys (timestamps, `ObjectId`) without hashing —
    they funnel new writes to one shard.

---

## 12. Mental Cheat-Sheet

> **Three questions to ask when a NoSQL app is slow:**
>
> 1. **Is the access pattern matching the data model?** (NoSQL can't fix a bad schema.)
> 2. **Is the load spread evenly across partitions?** (Hotspot is the #1 killer.)
> 3. **Does the working set fit in RAM?** (If not, latency will triple.)

```
   ┌─────────────────────────────────────────────────────────────┐
   │                  RESOLUTION CHECKLIST                       │
   ├─────────────────────────────────────────────────────────────┤
   │ □ Profile / EXPLAIN the slow query                         │
   │ □ Add or fix index (ESR rule, covered query)               │
   │ □ Reorder aggregation / push selective stage first         │
   │ □ Denormalise to remove cross-doc lookups (N+1)            │
   │ □ Salt / hash the partition key to fix hotspots            │
   │ □ Cap document/array sizes; bucket time-series             │
   │ □ Add RAM, shard, cache, or archive cold data              │
   │ □ Tune engine: compaction, GC, refresh interval, cache     │
   │ □ Enforce read-your-writes; monitor replication lag        │
   │ □ Fix the connection pool; use pipelining / bulk APIs      │
   └─────────────────────────────────────────────────────────────┘
```
