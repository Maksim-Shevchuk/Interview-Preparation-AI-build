# EXPLAIN and Query Optimization

Understanding execution plans is the single most important skill for diagnosing slow queries. This file covers how
the PostgreSQL query planner works, how to read `EXPLAIN` output, and practical optimization techniques. The concepts
apply to other RDBMS with minor syntax differences.

---

## How the Query Planner Works

When PostgreSQL receives a query, it goes through several stages:

```
SQL query
    │
    ▼
Parser          → syntax tree (validates syntax)
    │
    ▼
Rewriter        → applies rules and views
    │
    ▼
Planner/Optimizer → generates candidate plans, estimates costs, picks cheapest
    │
    ▼
Executor        → runs the chosen plan, returns rows
```

The planner considers:
- **Which indexes** are available and useful.
- **Join order** and **join strategy** (nested loop, hash, merge).
- **Scan method** for each table (sequential, index, bitmap).
- **Statistics** about the data (row count, value distribution, correlation).

It assigns a **cost** to each candidate plan and picks the one with the lowest estimated total cost.

---

## EXPLAIN Variants

| Command                    | What it does                                             |
|----------------------------|----------------------------------------------------------|
| `EXPLAIN`                  | Shows the plan without executing the query               |
| `EXPLAIN ANALYZE`          | Executes the query and shows actual timings + row counts |
| `EXPLAIN (ANALYZE, BUFFERS)` | + buffer hit/read statistics (I/O)                    |
| `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` | + output columns, schema-qualified names    |
| `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` | Machine-readable JSON output             |
| `EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)` | Skip per-node timing (less overhead)      |

**Always use `EXPLAIN (ANALYZE, BUFFERS)` for real investigation** — without `ANALYZE` you only see estimates,
which can be wildly wrong.

**Warning:** `EXPLAIN ANALYZE` actually executes the query. For `UPDATE`/`DELETE`, wrap in a transaction and rollback:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) DELETE FROM orders WHERE created_at < '2020-01-01';
ROLLBACK;
```

---

## Reading the Plan

### Basic Structure

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM employees WHERE department = 'Engineering';
```

```
Seq Scan on employees  (cost=0.00..12.50 rows=5 width=120)
                        (actual time=0.015..0.089 rows=5 loops=1)
  Filter: ((department)::text = 'Engineering'::text)
  Rows Removed by Filter: 95
  Buffers: shared hit=5
Planning Time: 0.045 ms
Execution Time: 0.112 ms
```

### Understanding Each Field

**Cost:**

```
(cost=0.00..12.50 rows=5 width=120)
       │      │     │      │
       │      │     │      └── estimated average row width in bytes
       │      │     └── estimated number of rows returned
       │      └── total cost to retrieve all rows
       └── startup cost (before first row can be returned)
```

Costs are in arbitrary units (sequential page reads ≈ 1.0 per page). They are **not** milliseconds. Useful for
**comparing** plans, not for predicting wall-clock time.

**Actual:**

```
(actual time=0.015..0.089 rows=5 loops=1)
              │       │     │      │
              │       │     │      └── how many times this node executed
              │       │     └── actual rows returned (per loop)
              │       └── time to return the last row (ms)
              └── time to return the first row (ms)
```

When `loops > 1`, multiply `actual time` and `rows` by `loops` for the true total.

**Buffers:**

```
Buffers: shared hit=5 read=3 dirtied=0 written=0
                  │      │        │          │
                  │      │        │          └── pages written to disk
                  │      │        └── pages modified in buffer cache
                  │      └── pages read from disk (cache miss)
                  └── pages found in buffer cache (cache hit)
```

High `read` count = cold cache or data too large to fit in memory. High `hit` count = data served from shared buffers.

### Plan Tree

Plans are trees — read **from inside out, bottom to top**:

```
Sort  (cost=150.20..152.70 rows=1000 width=40)
  Sort Key: salary DESC
  Sort Method: quicksort  Memory: 71kB
  ->  Hash Join  (cost=10.50..100.00 rows=1000 width=40)
        Hash Cond: (e.department_id = d.id)
        ->  Seq Scan on employees e  (cost=0.00..80.00 rows=5000 width=36)
        ->  Hash  (cost=8.00..8.00 rows=200 width=4)
              ->  Seq Scan on departments d  (cost=0.00..8.00 rows=200 width=4)
                    Filter: (location = 'NYC')
```

Execution order:
1. Seq Scan on `departments` with filter (innermost, bottom).
2. Build hash table from filtered departments.
3. Seq Scan on `employees`.
4. Probe hash table (Hash Join).
5. Sort result by salary.

---

## Scan Methods

| Scan Method          | How it works                                            | When used                         |
|----------------------|---------------------------------------------------------|-----------------------------------|
| **Seq Scan**         | Reads every row in the table sequentially               | No useful index, or most rows needed |
| **Index Scan**       | Traverse index → fetch matching rows from table (heap)  | Selective filter, few rows        |
| **Index Only Scan**  | Read from index only, no table access                   | Index covers all needed columns   |
| **Bitmap Index Scan** | Build bitmap of matching pages → Bitmap Heap Scan reads pages | Moderate selectivity (1–20% of rows) |
| **Bitmap Heap Scan** | Reads pages indicated by bitmap, applies recheck        | Paired with Bitmap Index Scan     |

### Seq Scan vs Index Scan

The planner chooses Seq Scan when:
- No index exists on the filter column.
- The query returns a **large fraction** of the table (index overhead > sequential read).
- The table is very small (a few pages).

**Rule of thumb:** an index is beneficial when the query selects **less than ~10-15%** of rows. For larger fractions,
a sequential scan with a filter is cheaper because sequential I/O is much faster than random I/O.

### Bitmap Scan

A two-step process for **medium selectivity** (too many rows for Index Scan, too few for Seq Scan):

```
Bitmap Heap Scan on orders  (cost=50.00..500.00 rows=2000 width=80)
  Recheck Cond: (status = 'pending')
  ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..48.00 rows=2000 width=0)
        Index Cond: (status = 'pending')
```

1. **Bitmap Index Scan** — traverse the index, build a bitmap of matching page numbers.
2. **Bitmap Heap Scan** — read those pages sequentially (converting random I/O to sequential).
3. **Recheck** — re-evaluate the condition (bitmap is lossy for very large result sets).

Multiple bitmap scans can be combined with `BitmapAnd` / `BitmapOr` — using multiple indexes on the same table.

### Index Only Scan

The fastest scan — reads only the index, never touches the table:

```
Index Only Scan using idx_employees_dept_salary on employees
  Index Cond: (department = 'Engineering')
  Heap Fetches: 0
```

Requirements:
- All columns in `SELECT`, `WHERE`, and `ORDER BY` are in the index.
- The **visibility map** is up-to-date (`VACUUM` keeps it current). `Heap Fetches > 0` means some pages weren't
  visible in the map and required a table lookup.

---

## Join Strategies

| Strategy         | Algorithm                                          | Best when                                 |
|------------------|----------------------------------------------------|-------------------------------------------|
| **Nested Loop**  | For each row in outer → scan inner                 | Small outer set, indexed inner             |
| **Hash Join**    | Build hash table from inner → probe with outer     | Medium/large sets, equality join, enough memory |
| **Merge Join**   | Both sides pre-sorted → merge in order             | Large pre-sorted sets, equality or range join |

### Nested Loop

```
Nested Loop  (cost=0.42..50.00 rows=10 width=80)
  ->  Index Scan on departments d  (cost=0.14..8.00 rows=2 width=40)
        Filter: (location = 'NYC')
  ->  Index Scan on employees e  (cost=0.28..20.00 rows=5 width=40)
        Index Cond: (department_id = d.id)
```

Good when the outer side is small. The inner side should have an index — otherwise it degrades to O(n × m).

### Hash Join

```
Hash Join  (cost=10.50..100.00 rows=1000 width=80)
  Hash Cond: (e.department_id = d.id)
  ->  Seq Scan on employees e
  ->  Hash
        ->  Seq Scan on departments d
              Buckets: 1024  Batches: 1  Memory Usage: 40kB
```

- `Batches: 1` — hash table fits in `work_mem`. Good.
- `Batches: 4` — hash table spilled to disk. Increase `work_mem` or reduce data.

### Merge Join

```
Merge Join  (cost=200.00..350.00 rows=5000 width=80)
  Merge Cond: (e.department_id = d.id)
  ->  Sort  (cost=150.00..160.00 rows=5000 width=40)
        Sort Key: e.department_id
        ->  Seq Scan on employees e
  ->  Sort  (cost=50.00..52.00 rows=200 width=40)
        Sort Key: d.id
        ->  Seq Scan on departments d
```

Efficient for large sorted datasets. If both sides already have index-sorted data, the Sort steps are eliminated.

---

## Sort and Aggregate Operations

### Sort

```
Sort  (cost=150.00..152.50 rows=1000 width=40)
  Sort Key: salary DESC
  Sort Method: quicksort  Memory: 71kB        ← good: in-memory
```

```
Sort  (cost=150.00..152.50 rows=1000 width=40)
  Sort Key: salary DESC
  Sort Method: external merge  Disk: 10240kB   ← bad: spilled to disk
```

**If Sort spills to disk:**
- Increase `work_mem` (per-operation memory limit).
- Add an index that provides the sort order.
- Reduce the data set with more selective filters.

### Aggregate

| Node               | Strategy                                                |
|---------------------|---------------------------------------------------------|
| `Aggregate`         | Single-group aggregation (no GROUP BY, or one group)   |
| `HashAggregate`     | Hash table for groups — fast, needs memory             |
| `GroupAggregate`     | Pre-sorted input — streaming, low memory               |

```
HashAggregate  (cost=100.00..105.00 rows=200 width=40)
  Group Key: department
  Batches: 1  Memory Usage: 40kB     ← fits in memory
  ->  Seq Scan on employees
```

If `HashAggregate` has `Batches > 1` or shows `Disk`, increase `work_mem`.

---

## Cost Parameters

The planner uses configurable cost constants to estimate plan costs:

| Parameter                    | Default | Meaning                                         |
|------------------------------|---------|--------------------------------------------------|
| `seq_page_cost`              | 1.0     | Cost of reading one page sequentially           |
| `random_page_cost`           | 4.0     | Cost of reading one random page (index lookup)  |
| `cpu_tuple_cost`             | 0.01    | Cost of processing one row                      |
| `cpu_index_tuple_cost`       | 0.005   | Cost of processing one index entry              |
| `cpu_operator_cost`          | 0.0025  | Cost of evaluating one operator/function        |
| `effective_cache_size`       | 4GB     | Estimate of available OS + shared_buffers cache  |

**Key insight:** `random_page_cost = 4 × seq_page_cost` reflects the random vs sequential I/O penalty on HDDs.
On SSDs, set `random_page_cost` to 1.1–1.5 — this makes the planner more willing to use index scans.

```sql
-- for SSD-backed databases
SET random_page_cost = 1.1;

-- check current values
SHOW random_page_cost;
SHOW work_mem;
SHOW effective_cache_size;
```

---

## Statistics

The planner relies on table statistics to estimate row counts and value distribution. Stale or inaccurate statistics
lead to bad plans.

### How Statistics Are Collected

```sql
ANALYZE employees;                          -- update stats for one table
ANALYZE;                                    -- update stats for all tables (autovacuum does this)
```

`autovacuum` runs `ANALYZE` automatically when ~10% of a table's rows have changed (default threshold).

### Viewing Statistics

```sql
-- row count and page count estimates
SELECT relname, reltuples, relpages FROM pg_class WHERE relname = 'employees';

-- column statistics
SELECT attname, n_distinct, most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats WHERE tablename = 'employees' AND attname = 'department';
```

| Field               | What it tells the planner                              |
|---------------------|--------------------------------------------------------|
| `n_distinct`        | Number of distinct values (-1 = all unique)           |
| `most_common_vals`  | Most frequent values                                   |
| `most_common_freqs` | Frequencies of those values                            |
| `histogram_bounds`  | Distribution of non-MCV values (equi-depth histogram) |
| `correlation`       | Physical row order vs logical order (-1 to 1)         |

### Increasing Statistics Target

Default sample size is 100 most common values. For columns with many distinct values or skewed distribution:

```sql
ALTER TABLE employees ALTER COLUMN department SET STATISTICS 500;
ANALYZE employees;
```

Higher target → more accurate estimates → better plans, but slightly more `ANALYZE` overhead.

---

## Reading Plan Misestimates

The most important diagnostic: **compare estimated rows vs actual rows**.

```
Seq Scan on orders  (cost=0.00..25000.00 rows=100 width=80)
                     (actual time=0.015..150.000 rows=50000 loops=1)
```

Planner expected 100 rows, got 50000 — a **500x misestimate**. This causes:
- Wrong join strategy (nested loop instead of hash join).
- Wrong join order.
- Wrong scan method.

### Common Causes of Misestimates

| Cause                              | Fix                                                      |
|------------------------------------|----------------------------------------------------------|
| Stale statistics                   | Run `ANALYZE`                                            |
| Correlated columns                 | Create a multi-column statistics object                  |
| Complex expressions in WHERE       | Create an expression index or use extended statistics    |
| Data skew                          | Increase `default_statistics_target`                     |
| Functions in WHERE (planner can't estimate) | Use `ROWS` hint on function or restructure query  |
| JOINs amplifying misestimates      | Fix underlying table estimates first                     |

### Multi-Column Statistics (Extended Statistics)

By default, the planner assumes columns are independent. If they're correlated (e.g., `city` and `zip_code`), the
planner underestimates combined selectivity:

```sql
CREATE STATISTICS stats_city_zip (dependencies, ndistinct)
    ON city, zip_code FROM addresses;
ANALYZE addresses;
```

---

## Index Optimization

### When to Create an Index

- Columns frequently in `WHERE`, `JOIN ON`, `ORDER BY`.
- High-selectivity filters (few matching rows out of many).
- Foreign keys (avoid sequential scans on the child table during joins and cascading deletes).

### When NOT to Create an Index

- Small tables (< 1000 rows) — Seq Scan is faster.
- Low selectivity (e.g., boolean with 50/50 distribution).
- Write-heavy tables with rare reads — each index slows down `INSERT`/`UPDATE`/`DELETE`.
- Columns rarely used in queries.

### Index Types (PostgreSQL)

| Type    | Use case                                          | Example                          |
|---------|---------------------------------------------------|----------------------------------|
| B-tree  | Equality and range queries (default)              | `=`, `<`, `>`, `BETWEEN`, `IN`  |
| Hash    | Equality only (rare, B-tree is usually better)    | `=`                              |
| GIN     | Full-text search, JSONB, arrays                   | `@>`, `?`, `@@`                 |
| GiST    | Geometric, range types, full-text                 | `&&`, `@>`, `<->`              |
| BRIN    | Large, naturally ordered tables (timestamps, IDs) | Range queries on append-only data|
| SP-GiST | Space-partitioned data (phone numbers, IPs)      | Various                          |

### Composite (Multi-Column) Index

```sql
CREATE INDEX idx_emp_dept_salary ON employees (department, salary);
```

**Leftmost prefix rule:** this index is useful for:
- `WHERE department = 'Engineering'` — yes (uses first column).
- `WHERE department = 'Engineering' AND salary > 80000` — yes (uses both columns).
- `WHERE salary > 80000` — **no** (can't skip the first column).
- `ORDER BY department, salary` — yes.
- `ORDER BY salary, department` — **no** (wrong column order).

### Covering Index (INCLUDE)

Include non-key columns to enable Index Only Scan without bloating the index tree:

```sql
CREATE INDEX idx_emp_dept ON employees (department) INCLUDE (name, salary);
```

The planner can answer `SELECT name, salary FROM employees WHERE department = 'Engineering'` purely from the index.

### Partial Index

Index only a subset of rows — smaller, faster, less write overhead:

```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status = 'pending';
```

Only useful when queries include the matching `WHERE` clause.

### Expression Index

Index the result of an expression or function:

```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```

Now `WHERE LOWER(email) = 'alice@example.com'` uses the index instead of Seq Scan.

### Index and ORDER BY

An index provides pre-sorted data — eliminates the need for a `Sort` node:

```sql
CREATE INDEX idx_orders_date ON orders (created_at DESC);

-- this query uses the index order, no Sort needed
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;
```

### Finding Unused Indexes

```sql
SELECT schemaname, relname, indexrelname, idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

Unused indexes waste disk space and slow down writes. Drop them (after confirming they're not needed for unique
constraints or rarely-run reports).

### Finding Missing Indexes

```sql
SELECT relname, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch,
       seq_tup_read / GREATEST(seq_scan, 1) AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND seq_tup_read / GREATEST(seq_scan, 1) > 1000
ORDER BY seq_tup_read DESC;
```

High `seq_scan` count with many `seq_tup_read` per scan on a large table → a likely candidate for an index.

---

## Common Query Optimization Patterns

### 1. Avoid `SELECT *`

```sql
-- ❌ reads all columns, prevents Index Only Scan
SELECT * FROM employees WHERE department = 'Engineering';

-- ✅ only needed columns
SELECT id, name, salary FROM employees WHERE department = 'Engineering';
```

### 2. Avoid Functions on Indexed Columns

```sql
-- ❌ prevents index usage (function applied to every row)
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2025;

-- ✅ use range instead
SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';

-- ❌ implicit cast can prevent index
SELECT * FROM users WHERE phone = 12345;  -- phone is VARCHAR, 12345 is INT

-- ✅ match the type
SELECT * FROM users WHERE phone = '12345';
```

### 3. Use EXISTS Instead of IN for Large Subqueries

```sql
-- ❌ IN materializes the entire subquery result
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);

-- ✅ EXISTS short-circuits on first match
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total > 1000);
```

In modern PostgreSQL the optimizer often rewrites `IN` to a semi-join (same as `EXISTS`), but `EXISTS` is the safer
choice.

### 4. Avoid `OFFSET` for Deep Pagination

```sql
-- ❌ scans and discards 100,000 rows
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- ✅ keyset (cursor-based) pagination
SELECT * FROM orders WHERE id > :last_seen_id ORDER BY id LIMIT 20;
```

`OFFSET` has O(n) cost — the database must fetch and discard all skipped rows. Keyset pagination uses the index
directly and is O(1).

### 5. Optimize OR to UNION ALL

```sql
-- ❌ OR can prevent index usage (planner may fall back to Seq Scan)
SELECT * FROM orders WHERE customer_id = 42 OR status = 'pending';

-- ✅ UNION ALL can use separate indexes
SELECT * FROM orders WHERE customer_id = 42
UNION ALL
SELECT * FROM orders WHERE status = 'pending' AND customer_id != 42;
```

Note: PostgreSQL can sometimes use `BitmapOr` for `OR` conditions, but `UNION ALL` is more predictable.

### 6. Materialize Expensive CTEs (When Needed)

```sql
-- PostgreSQL 12+ inlines CTEs by default (can optimize through them)
-- force materialization if the CTE is referenced multiple times and expensive
WITH expensive AS MATERIALIZED (
    SELECT customer_id, SUM(total) AS lifetime FROM orders GROUP BY customer_id
)
SELECT * FROM expensive WHERE lifetime > 10000
UNION ALL
SELECT * FROM expensive WHERE lifetime < 100;
```

### 7. Batch Inserts

```sql
-- ❌ 1000 separate round trips
INSERT INTO logs (message) VALUES ('msg1');
INSERT INTO logs (message) VALUES ('msg2');
...

-- ✅ single statement
INSERT INTO logs (message) VALUES ('msg1'), ('msg2'), ... ('msg1000');

-- ✅ or COPY for bulk loading
COPY logs (message) FROM '/path/to/data.csv' WITH (FORMAT csv);
```

### 8. Avoid Correlated Subqueries When Possible

```sql
-- ❌ subquery executes once per outer row
SELECT e.name, e.salary,
    (SELECT AVG(salary) FROM employees e2 WHERE e2.department = e.department) AS dept_avg
FROM employees e;

-- ✅ join to a CTE or derived table — aggregation runs once
SELECT e.name, e.salary, d.dept_avg
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS dept_avg FROM employees GROUP BY department
) d ON e.department = d.department;
```

### 9. Use Partial Indexes for Common Filters

```sql
-- if 90% of queries filter by status = 'active'
CREATE INDEX idx_orders_active ON orders (customer_id, created_at)
    WHERE status = 'active';

-- much smaller than a full index, maintained only for active rows
```

### 10. Denormalize for Read Performance (Trade-off)

When joins across many tables dominate query time:
- Add a materialized view for complex aggregations.
- Store computed columns (denormalized) and update them with triggers.
- Use `JSONB` for semi-structured data to avoid multi-table joins.

This sacrifices write performance and data consistency guarantees for read speed.

---

## Server Configuration for Performance

| Parameter                | Default    | When to tune                                          |
|--------------------------|------------|-------------------------------------------------------|
| `shared_buffers`         | 128MB      | Set to 25% of total RAM                              |
| `effective_cache_size`   | 4GB        | Set to 50–75% of total RAM (tells planner about OS cache) |
| `work_mem`               | 4MB        | Increase for complex sorts/aggregations (per-operation!) |
| `maintenance_work_mem`   | 64MB       | Increase for `VACUUM`, `CREATE INDEX`, `ALTER TABLE`  |
| `random_page_cost`       | 4.0        | Set to 1.1–1.5 for SSDs                              |
| `effective_io_concurrency`| 1         | Set to 200 for SSDs                                   |
| `max_parallel_workers_per_gather` | 2 | Increase for large analytical queries                |
| `jit`                    | on         | Disable if queries are short (JIT compilation has startup cost) |

**Warning:** `work_mem` is per-operation, not per-query. A query with 10 sorts uses 10 × `work_mem`. Setting it too
high can cause out-of-memory with many concurrent connections.

---

## Diagnostic Queries

### Slow Query Log

```sql
-- log queries slower than 500ms
ALTER SYSTEM SET log_min_duration_statement = 500;
SELECT pg_reload_conf();
```

### Currently Running Queries

```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state, wait_event_type
FROM pg_stat_activity
WHERE state != 'idle' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY duration DESC;
```

### Table Bloat and Autovacuum Status

```sql
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 1) AS dead_pct,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

High `n_dead_tup` means `VACUUM` needs to run — dead tuples waste space and slow down scans.

### Cache Hit Ratio

```sql
SELECT
    sum(heap_blks_hit) / GREATEST(sum(heap_blks_hit) + sum(heap_blks_read), 1) AS cache_hit_ratio
FROM pg_statio_user_tables;
-- should be > 0.99 for OLTP workloads
```

### Index Hit Ratio

```sql
SELECT
    sum(idx_blks_hit) / GREATEST(sum(idx_blks_hit) + sum(idx_blks_read), 1) AS index_hit_ratio
FROM pg_statio_user_indexes;
```

---

## Common Interview Questions

### How do you optimize a slow query?

1. Run `EXPLAIN (ANALYZE, BUFFERS)` to get the actual plan.
2. Compare estimated vs actual row counts — fix misestimates with `ANALYZE` or extended statistics.
3. Look for Seq Scans on large tables with selective filters → add indexes.
4. Look for `Sort Method: external merge Disk` → increase `work_mem` or add index.
5. Look for Nested Loop with large inner table → check for missing inner index or consider hash join.
6. Check `Buffers: read` — high reads mean cold cache or data doesn't fit in memory.
7. Rewrite the query: remove `SELECT *`, replace `OFFSET` with keyset pagination, use `EXISTS` instead of `IN`.

### What is the difference between Seq Scan and Index Scan?

Seq Scan reads the entire table sequentially — O(n) but good I/O pattern. Index Scan traverses the B-tree to find
matching rows, then fetches them from the table via random I/O. For selective queries (< ~10-15% of rows) Index Scan
is faster. For large fractions, Seq Scan wins because sequential I/O is cheaper than random I/O.

### Why does PostgreSQL choose Seq Scan even when an index exists?

- The query returns a large fraction of the table — Seq Scan is cheaper.
- Statistics are stale — run `ANALYZE`.
- `random_page_cost` is too high for the actual storage (set lower for SSDs).
- The WHERE condition uses a function that doesn't match the index expression.
- The data type in the query doesn't match the column type (implicit cast prevents index use).

### What is an Index Only Scan and when does it work?

Index Only Scan reads data only from the index without accessing the table. It works when all columns in SELECT,
WHERE, and ORDER BY are in the index (covering index). It also requires the table's visibility map to be up-to-date
(maintained by VACUUM) — otherwise it falls back to fetching tuples from the table (`Heap Fetches > 0`).

### What is `work_mem` and how does it affect query plans?

`work_mem` is the amount of memory each sort or hash operation can use before spilling to disk. Low `work_mem` causes
disk-based sorts and hash joins (visible as `Sort Method: external merge Disk` or `Hash Batches > 1`). Increasing it
allows in-memory operations — much faster. But it's per-operation, not per-query, so setting it too high with many
concurrent queries can exhaust server memory.

### How do you find and fix a row count misestimate?

Compare `rows` (estimated) vs `actual rows` in `EXPLAIN ANALYZE`. If they differ by more than 10x:
1. Run `ANALYZE` on the table.
2. Check if columns are correlated → create extended statistics.
3. Check if the planner can't estimate a function → use a simpler expression or create an expression index.
4. Increase `default_statistics_target` for columns with many distinct values or skewed distribution.
