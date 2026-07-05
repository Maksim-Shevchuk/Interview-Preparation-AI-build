# Transactions and Isolation Levels

Transactions are the foundation of data integrity in relational databases. Interviews cover ACID properties, isolation
levels, concurrency anomalies, locking mechanisms, and how PostgreSQL implements MVCC.

---

## ACID Properties

| Property        | Guarantee                                                              |
|-----------------|------------------------------------------------------------------------|
| **Atomicity**   | All operations in a transaction succeed or all are rolled back — no partial state |
| **Consistency** | A transaction moves the database from one valid state to another (constraints, triggers, rules are respected) |
| **Isolation**   | Concurrent transactions don't interfere with each other (as if executed serially) |
| **Durability**  | Once committed, data survives crashes (written to WAL / redo log before commit) |

### How Each Property Is Implemented (PostgreSQL)

| Property    | Mechanism                                                           |
|-------------|---------------------------------------------------------------------|
| Atomicity   | WAL (Write-Ahead Log) — undo uncommitted changes on crash recovery |
| Consistency | Constraints (`CHECK`, `UNIQUE`, `FK`), triggers, serializable isolation |
| Isolation   | MVCC (Multi-Version Concurrency Control) + locks                   |
| Durability  | WAL flushed to disk on commit (`synchronous_commit = on`)          |

---

## Transaction Basics

```sql
BEGIN;                          -- or START TRANSACTION
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                         -- make changes permanent

-- or
ROLLBACK;                       -- discard all changes
```

### Savepoints

Partial rollback within a transaction:

```sql
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (1, 500);
SAVEPOINT sp1;
INSERT INTO order_items (order_id, product_id) VALUES (1, 999);  -- fails: product 999 doesn't exist
ROLLBACK TO sp1;                -- undo only the failed insert
INSERT INTO order_items (order_id, product_id) VALUES (1, 42);   -- retry with valid product
COMMIT;                         -- order + valid item committed
```

### Auto-Commit

Most databases (PostgreSQL, MySQL) run in auto-commit mode by default — each statement is its own transaction.
`BEGIN` starts an explicit transaction block that requires `COMMIT` or `ROLLBACK`.

In Spring/JDBC, `@Transactional` wraps the method body in `BEGIN...COMMIT/ROLLBACK` automatically.

---

## Concurrency Anomalies

These are the problems that occur when transactions run concurrently without proper isolation:

### 1. Dirty Read

A transaction reads data written by another **uncommitted** transaction.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;
                                                     T2: SELECT balance FROM accounts WHERE id = 1;
                                                         → reads 0 (uncommitted!)
T1: ROLLBACK;
                                                     T2: used a value that never existed
```

### 2. Non-Repeatable Read

A transaction reads the same row twice and gets **different values** because another transaction committed a change
in between.

```
T1: SELECT balance FROM accounts WHERE id = 1;      → 1000
                                                     T2: UPDATE accounts SET balance = 500 WHERE id = 1;
                                                     T2: COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;      → 500 (different!)
```

### 3. Phantom Read

A transaction re-executes a range query and gets **different rows** because another transaction inserted or deleted
matching rows.

```
T1: SELECT COUNT(*) FROM orders WHERE status = 'pending';  → 5
                                                     T2: INSERT INTO orders (status) VALUES ('pending');
                                                     T2: COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'pending';  → 6 (phantom row!)
```

### 4. Lost Update

Two transactions read the same value, both modify it, and the last commit overwrites the first — one update is lost.

```
T1: SELECT balance FROM accounts WHERE id = 1;      → 1000
                                                     T2: SELECT balance FROM accounts WHERE id = 1;  → 1000
T1: UPDATE accounts SET balance = 1000 + 100;        -- sets to 1100
T1: COMMIT;
                                                     T2: UPDATE accounts SET balance = 1000 + 200;   -- sets to 1200
                                                     T2: COMMIT;
                                                     -- T1's +100 is lost, balance is 1200 instead of 1300
```

### 5. Write Skew

Two transactions read overlapping data, make decisions based on it, and write to different rows — the combined result
violates a business constraint.

```
-- constraint: at least one doctor must be on call
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;   → 2
T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;   → 2

T1: UPDATE doctors SET on_call = false WHERE id = 1;     -- "still 1 left"
T2: UPDATE doctors SET on_call = false WHERE id = 2;     -- "still 1 left"

-- both commit: no doctors on call (constraint violated)
```

Write skew is only prevented by Serializable isolation.

---

## SQL Standard Isolation Levels

| Level              | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update | Write Skew |
|--------------------|------------|---------------------|--------------|-------------|------------|
| Read Uncommitted   | Possible   | Possible            | Possible     | Possible    | Possible   |
| Read Committed     | Prevented  | Possible            | Possible     | Possible    | Possible   |
| Repeatable Read    | Prevented  | Prevented           | Possible*    | Prevented   | Possible*  |
| Serializable       | Prevented  | Prevented           | Prevented    | Prevented   | Prevented  |

*PostgreSQL's Repeatable Read is stricter than the SQL standard — it also prevents phantom reads (due to
snapshot isolation). Write skew is still possible.

### Setting Isolation Level

```sql
-- per transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- or
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- per session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

```java
// Spring
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transfer(Long from, Long to, BigDecimal amount) { }
```

---

## How PostgreSQL Implements Each Level

PostgreSQL uses **MVCC** — it does not implement Read Uncommitted (silently upgrades to Read Committed).

### Read Committed (Default)

- Each **statement** sees a snapshot of committed data as of the statement's start.
- Different statements within the same transaction can see different data (if other transactions commit in between).
- Re-executing the same query may return different results.

```
T1: BEGIN;
T1: SELECT balance FROM accounts WHERE id = 1;       → 1000
                                                      T2: UPDATE accounts SET balance = 500 WHERE id = 1;
                                                      T2: COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;       → 500 (sees T2's commit)
```

**UPDATE behavior in Read Committed:**

If T1 tries to update a row that T2 has already modified and committed, T1 **re-evaluates** the WHERE clause against
the new version:

```
T1: UPDATE accounts SET balance = balance + 100 WHERE balance > 500;
                                                      T2: UPDATE accounts SET balance = 400 WHERE id = 1;
                                                      T2: COMMIT;
T1: -- re-evaluates WHERE: balance is now 400, condition fails → row is skipped
```

### Repeatable Read

- The transaction sees a snapshot of committed data as of the **transaction's start**.
- All queries within the transaction see the same consistent snapshot.
- Prevents non-repeatable reads and phantom reads.
- If another transaction modifies and commits a row that this transaction also tries to modify → **serialization error**:

```
T1: BEGIN ISOLATION LEVEL REPEATABLE READ;
T1: SELECT balance FROM accounts WHERE id = 1;       → 1000
                                                      T2: UPDATE accounts SET balance = 500 WHERE id = 1;
                                                      T2: COMMIT;
T1: UPDATE accounts SET balance = balance + 100 WHERE id = 1;
    → ERROR: could not serialize access due to concurrent update
T1: must ROLLBACK and retry
```

### Serializable (SSI — Serializable Snapshot Isolation)

- Built on top of Repeatable Read snapshot isolation.
- Adds **dependency tracking** (read-write conflicts between transactions).
- Detects cycles in the dependency graph → aborts one transaction with a serialization error.
- Guarantees the result is equivalent to **some** serial execution order.

```
T1: BEGIN ISOLATION LEVEL SERIALIZABLE;
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;   → 2
                                                      T2: BEGIN ISOLATION LEVEL SERIALIZABLE;
                                                      T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;  → 2
T1: UPDATE doctors SET on_call = false WHERE id = 1;
                                                      T2: UPDATE doctors SET on_call = false WHERE id = 2;
T1: COMMIT;                                           → OK
                                                      T2: COMMIT;
                                                      → ERROR: could not serialize access (write skew detected)
```

**Cost:** serialization errors require **retry logic** in the application. Throughput is lower than Read Committed
because more transactions are aborted.

---

## MVCC (Multi-Version Concurrency Control)

### How It Works

Instead of overwriting data in-place, PostgreSQL keeps **multiple versions** of each row:

```
┌─────────────────────────────────────────────────────────────┐
│  Row version 1:  xmin=100  xmax=105  balance=1000           │  ← created by TX 100, "deleted" by TX 105
│  Row version 2:  xmin=105  xmax=∞    balance=1200           │  ← created by TX 105, currently live
└─────────────────────────────────────────────────────────────┘
```

| Field   | Meaning                                                       |
|---------|---------------------------------------------------------------|
| `xmin`  | Transaction ID that created this row version                  |
| `xmax`  | Transaction ID that deleted/updated this version (∞ if live)  |
| `ctid`  | Physical location (page, offset) of the row                   |

### Visibility Rules

A row version is visible to a transaction if:
1. `xmin` is committed AND `xmin` < snapshot.
2. `xmax` is not set, OR `xmax` is uncommitted, OR `xmax` > snapshot.

This means:
- **Readers never block writers.** A reader sees the old version while a writer creates a new version.
- **Writers never block readers.** The old version remains visible until the writer commits.
- **Writers block writers** only when modifying the same row (row-level lock).

### VACUUM

Old row versions (dead tuples) accumulate and waste space. `VACUUM` reclaims this space:

```sql
VACUUM employees;              -- mark dead tuples as reusable (doesn't shrink file)
VACUUM FULL employees;         -- rewrites the table (exclusive lock, shrinks file)
VACUUM (VERBOSE) employees;    -- show what was cleaned up
```

`autovacuum` runs automatically in the background. Critical for:
- Reclaiming disk space from dead tuples.
- Updating the **visibility map** (enables Index Only Scans).
- Preventing **transaction ID wraparound** (every ~2 billion transactions).

---

## Locking

### Lock Types

PostgreSQL has multiple lock levels (from least to most restrictive):

| Lock Mode                  | Conflicts with                     | Acquired by                        |
|----------------------------|------------------------------------|------------------------------------|
| `ACCESS SHARE`             | `ACCESS EXCLUSIVE`                 | `SELECT`                           |
| `ROW SHARE`                | `EXCLUSIVE`, `ACCESS EXCLUSIVE`    | `SELECT FOR UPDATE/SHARE`          |
| `ROW EXCLUSIVE`            | `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` |
| `SHARE`                    | `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `CREATE INDEX` |
| `ACCESS EXCLUSIVE`         | All                                | `ALTER TABLE`, `DROP`, `VACUUM FULL`, `TRUNCATE` |

**Key insight:** `SELECT` and `INSERT/UPDATE/DELETE` don't conflict with each other — MVCC handles this.
Only DDL operations (`ALTER TABLE`, `DROP`) take `ACCESS EXCLUSIVE` which blocks everything.

### Row-Level Locks

```sql
-- exclusive lock — prevent other transactions from modifying or locking these rows
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- shared lock — prevent modifications but allow other FOR SHARE locks
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- skip locked rows (useful for job queues)
SELECT * FROM tasks WHERE status = 'pending' LIMIT 1 FOR UPDATE SKIP LOCKED;

-- don't wait — fail immediately if row is locked
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

### `FOR UPDATE` Use Cases

**Prevent lost updates (pessimistic locking):**

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;   -- lock the row
-- calculate new balance in application
UPDATE accounts SET balance = new_balance WHERE id = 1;
COMMIT;
```

**Job queue pattern:**

```sql
-- worker picks a job and locks it
BEGIN;
SELECT id, payload FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- process the job...
UPDATE jobs SET status = 'completed' WHERE id = :id;
COMMIT;
```

### Advisory Locks

Application-level locks — not tied to any table or row:

```sql
-- session-level lock (released on disconnect)
SELECT pg_advisory_lock(12345);
-- do exclusive work...
SELECT pg_advisory_unlock(12345);

-- transaction-level lock (released on COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);

-- try without blocking
SELECT pg_try_advisory_lock(12345);   -- returns true/false
```

Use cases: preventing duplicate cron runs, distributed mutex, rate limiting.

---

## Deadlocks

A deadlock occurs when two or more transactions wait for each other's locks indefinitely:

```
T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- locks row 1
                                                     T2: UPDATE accounts SET balance - 50 WHERE id = 2;   -- locks row 2
T1: UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- waits for T2's lock on row 2
                                                     T2: UPDATE accounts SET balance + 50 WHERE id = 1;   -- waits for T1's lock on row 1
-- DEADLOCK: T1 waits for T2, T2 waits for T1
```

PostgreSQL detects deadlocks automatically (every `deadlock_timeout`, default 1s) and aborts one transaction with:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678;
        blocked by process 5679.
        Process 5679 waits for ShareLock on transaction 5677;
        blocked by process 1234.
```

### Preventing Deadlocks

1. **Consistent lock ordering** — always lock rows in the same order (e.g., by ascending ID).
   ```sql
   -- always lock the lower ID first
   SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
   ```
2. **Keep transactions short** — less time holding locks = less chance of conflict.
3. **Use `NOWAIT` or `SKIP LOCKED`** — fail fast instead of waiting.
4. **Reduce isolation level** — Serializable has more conflicts than Read Committed.
5. **Avoid explicit locking when possible** — let MVCC handle concurrency.

---

## Optimistic vs Pessimistic Locking

### Pessimistic Locking

Lock the data **before** reading to prevent conflicts:

```sql
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE;  -- lock
-- modify in application
UPDATE products SET stock = stock - 1 WHERE id = 42;
COMMIT;
```

- **Pros:** guarantees no conflicts, simple logic.
- **Cons:** reduces concurrency, risk of deadlocks, holds locks during application processing.
- **Use when:** conflicts are frequent, transactions are short.

### Optimistic Locking

No locks during read. On write, check that data hasn't changed:

```sql
-- read with version
SELECT id, stock, version FROM products WHERE id = 42;
-- → stock = 10, version = 5

-- update only if version matches
UPDATE products SET stock = 9, version = 6
WHERE id = 42 AND version = 5;

-- if 0 rows affected → someone else modified it → retry or abort
```

In JPA / Spring Data:

```java
@Entity
public class Product {
    @Id
    private Long id;
    private int stock;

    @Version                // JPA manages this automatically
    private int version;    // throws OptimisticLockException on conflict
}
```

- **Pros:** high concurrency, no locks held during processing.
- **Cons:** requires retry logic, wasted work on conflict.
- **Use when:** conflicts are rare, transactions are long or involve user interaction.

---

## Practical Patterns

### Idempotent Transactions

Ensure that retrying a transaction produces the same result:

```sql
-- idempotent upsert
INSERT INTO payments (idempotency_key, amount, status)
VALUES ('key-123', 100, 'completed')
ON CONFLICT (idempotency_key) DO NOTHING;
```

### Read-Only Transactions

Declare intent — allows the database to optimize:

```sql
BEGIN TRANSACTION READ ONLY;
SELECT ...;
COMMIT;
```

```java
@Transactional(readOnly = true)
public List<Order> findAll() { }
```

Benefits in Spring: Hibernate sets `FlushMode.MANUAL` (no dirty checking), connection can be routed to a read
replica.

### Retry Logic for Serialization Errors

```java
@Retryable(
    retryFor = {CannotAcquireLockException.class, SerializationException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transfer(Long from, Long to, BigDecimal amount) {
    // ...
}
```

---

## MySQL vs PostgreSQL Differences

| Aspect                    | PostgreSQL                               | MySQL (InnoDB)                          |
|---------------------------|------------------------------------------|-----------------------------------------|
| Default isolation         | Read Committed                           | Repeatable Read                         |
| MVCC implementation       | Old versions in main table (VACUUM needed) | Old versions in undo log (auto-purged) |
| Read Uncommitted          | Silently upgraded to Read Committed      | Actually allows dirty reads             |
| Repeatable Read           | True snapshot isolation (no phantoms)    | Snapshot for reads, gap locks for writes |
| Serializable              | SSI (non-blocking detection)             | All reads become `SELECT ... FOR SHARE` (locking) |
| Gap locks                 | No (uses predicate locks in Serializable)| Yes (in Repeatable Read and Serializable)|
| Deadlock detection        | Automatic (configurable timeout)         | Automatic (immediate)                   |

### MySQL Gap Locks

InnoDB in Repeatable Read uses **gap locks** to prevent phantom reads — it locks not just matching rows but also the
**gaps** between index values where new rows could be inserted:

```sql
-- locks the gap between existing rows where salary could be 50000-60000
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 60000 FOR UPDATE;
-- other transactions cannot INSERT rows with salary in this range
```

Gap locks increase contention but prevent phantoms without full serialization.

---

## Common Interview Questions

### What are ACID properties?

**Atomicity** — all-or-nothing execution. **Consistency** — database moves from one valid state to another.
**Isolation** — concurrent transactions don't see each other's uncommitted changes. **Durability** — committed data
survives crashes. Together they guarantee reliable transaction processing.

### What is the default isolation level and why?

PostgreSQL defaults to **Read Committed** — a good balance between consistency and performance. Each statement sees
only committed data, which prevents dirty reads while allowing high concurrency. Most applications work correctly
at this level. MySQL defaults to **Repeatable Read** — stricter, preventing non-repeatable reads within a transaction.

### What is the difference between Repeatable Read and Serializable?

Repeatable Read gives each transaction a consistent snapshot — all reads see the same data from transaction start.
But write skew is possible (two transactions read overlapping data and make conflicting writes to different rows).
Serializable adds dependency tracking and aborts transactions that would create non-serializable schedules —
preventing write skew at the cost of potential serialization failures that require retries.

### What is MVCC and why does it matter?

MVCC (Multi-Version Concurrency Control) keeps multiple versions of each row. Readers see a consistent snapshot
without acquiring locks, so readers never block writers and writers never block readers. This enables high concurrency
compared to lock-based isolation where readers and writers block each other.

### What is a deadlock and how do you prevent it?

A deadlock occurs when two transactions each hold a lock and wait for the other's lock. Prevention: always acquire
locks in a consistent order (e.g., by primary key), keep transactions short, use `NOWAIT` or `SKIP LOCKED`, avoid
unnecessary locking.

### When should you use pessimistic vs optimistic locking?

**Pessimistic** (`SELECT FOR UPDATE`) — when conflicts are frequent and transactions are short. Guarantees no
conflicts but reduces concurrency. **Optimistic** (`@Version` column) — when conflicts are rare. Higher concurrency
but requires retry logic when conflicts occur. Common choice for web applications where most requests don't
conflict.

### What is a lost update and how do you prevent it?

A lost update occurs when two transactions read the same value, both modify it independently, and the last commit
overwrites the first. Prevention: use `SELECT FOR UPDATE` (pessimistic), use a version column (optimistic), use
Repeatable Read or Serializable isolation (the database detects the conflict), or use atomic SQL
(`UPDATE accounts SET balance = balance + 100` instead of read-modify-write).

### What happens when two transactions update the same row?

Depends on the isolation level:
- **Read Committed:** the second transaction waits for the first to commit, then re-evaluates its WHERE clause against
  the new data and proceeds.
- **Repeatable Read:** the second transaction gets a serialization error and must be retried.
- **Serializable:** same as Repeatable Read — serialization error.

In all cases, the row-level lock prevents actual data corruption — the question is whether the second transaction
proceeds automatically, fails, or overwrites.
