# SQL Fundamentals

SQL (Structured Query Language) is the standard language for interacting with relational databases. Interviews cover
query writing, joins, aggregations, subqueries, window functions, and the ability to read and optimize execution plans.

---

## Query Execution Order

SQL is written in one order but **executed** in another. Understanding this prevents common mistakes:

```
FROM        ← 1. Identify tables and joins
WHERE       ← 2. Filter rows
GROUP BY    ← 3. Group rows
HAVING      ← 4. Filter groups
SELECT      ← 5. Evaluate expressions, aliases
DISTINCT    ← 6. Remove duplicates
ORDER BY    ← 7. Sort results
LIMIT       ← 8. Restrict output
```

**Implication:** you cannot use a `SELECT` alias in `WHERE` or `GROUP BY` (it hasn't been computed yet), but you
**can** use it in `ORDER BY`.

```sql
-- ❌ error: alias not available in WHERE
SELECT price * quantity AS total FROM orders WHERE total > 100;

-- ✅ repeat the expression
SELECT price * quantity AS total FROM orders WHERE price * quantity > 100;

-- ✅ or use a subquery / CTE
SELECT * FROM (
    SELECT *, price * quantity AS total FROM orders
) sub WHERE total > 100;
```

---

## SELECT Basics

```sql
-- all columns
SELECT * FROM employees;

-- specific columns with alias
SELECT first_name, last_name, salary AS annual_salary FROM employees;

-- expressions
SELECT first_name, salary * 12 AS annual, UPPER(department) AS dept FROM employees;

-- distinct values
SELECT DISTINCT department FROM employees;

-- limit rows
SELECT * FROM employees LIMIT 10;           -- PostgreSQL, MySQL
SELECT * FROM employees FETCH FIRST 10 ROWS ONLY;  -- SQL standard
SELECT * FROM employees OFFSET 20 LIMIT 10; -- pagination (skip 20, take 10)
```

---

## Filtering (WHERE)

### Comparison Operators

```sql
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price BETWEEN 50 AND 150;  -- inclusive on both ends
SELECT * FROM products WHERE category IN ('Electronics', 'Books', 'Toys');
SELECT * FROM products WHERE name LIKE 'iPhone%';        -- starts with
SELECT * FROM products WHERE name LIKE '%Pro%';          -- contains
SELECT * FROM products WHERE name LIKE 'iPhone __';      -- exactly 2 chars after
SELECT * FROM products WHERE name ILIKE '%phone%';       -- case-insensitive (PostgreSQL)
SELECT * FROM products WHERE description IS NULL;
SELECT * FROM products WHERE description IS NOT NULL;
```

### Logical Operators

```sql
SELECT * FROM employees
WHERE department = 'Engineering'
  AND salary > 80000
  AND (role = 'Senior' OR role = 'Lead');
```

Precedence: `NOT` > `AND` > `OR`. Always use parentheses to make intent explicit.

### NULL Handling

`NULL` represents unknown/missing data. Any comparison with `NULL` returns `NULL` (not `true` or `false`):

```sql
-- ❌ these never match any rows where name IS NULL
SELECT * FROM users WHERE name = NULL;
SELECT * FROM users WHERE name != 'Alice';  -- also excludes NULLs

-- ✅ correct
SELECT * FROM users WHERE name IS NULL;
SELECT * FROM users WHERE name != 'Alice' OR name IS NULL;

-- COALESCE — replace NULL with a default
SELECT COALESCE(nickname, first_name, 'Anonymous') AS display_name FROM users;

-- NULLIF — returns NULL if both args are equal
SELECT NULLIF(dividend, 0);  -- avoids division by zero: x / NULLIF(y, 0)
```

---

## Sorting (ORDER BY)

```sql
SELECT * FROM employees ORDER BY salary DESC;
SELECT * FROM employees ORDER BY department ASC, salary DESC;
SELECT * FROM employees ORDER BY CASE
    WHEN role = 'Lead' THEN 1
    WHEN role = 'Senior' THEN 2
    ELSE 3
END;

-- NULLs sort last (PostgreSQL)
SELECT * FROM employees ORDER BY salary DESC NULLS LAST;
```

---

## Joins

### Join Types

```
Table A             Table B
┌────┐              ┌────┐
│ 1  │              │ 2  │
│ 2  │              │ 3  │
│ 3  │              │ 4  │
└────┘              └────┘
```

| Join                | Rows returned                                            | Unmatched rows    |
|---------------------|----------------------------------------------------------|-------------------|
| `INNER JOIN`        | Only rows that match in **both** tables                  | Excluded          |
| `LEFT JOIN`         | All from left + matching from right (NULLs if no match)  | Right → NULL      |
| `RIGHT JOIN`        | All from right + matching from left (NULLs if no match)  | Left → NULL       |
| `FULL OUTER JOIN`   | All from both (NULLs where no match on either side)      | Both → NULL       |
| `CROSS JOIN`        | Cartesian product (every row × every row)                | N/A               |
| `SELF JOIN`         | Table joined to itself (not a separate type)             | Depends on join type |

### Examples

```sql
-- INNER JOIN — employees with their department info
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- LEFT JOIN — all employees, even those without a department
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;

-- find employees WITHOUT a department
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;

-- CROSS JOIN — all combinations (e.g., sizes × colors)
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;

-- SELF JOIN — employee with their manager
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- multiple joins
SELECT o.id, c.name, p.product_name, oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

### Join vs Subquery

```sql
-- join (often more readable and optimizable)
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id;

-- subquery (equivalent)
SELECT e.name,
       (SELECT d.department_name FROM departments d WHERE d.id = e.department_id)
FROM employees e;
```

The optimizer often transforms one into the other. Prefer joins for readability; use subqueries when the logic is
inherently nested (e.g., "find employees whose salary is above the average").

---

## Aggregation (GROUP BY)

### Aggregate Functions

| Function                | Purpose                                         |
|-------------------------|-------------------------------------------------|
| `COUNT(*)`              | Number of rows (including NULLs)                |
| `COUNT(column)`         | Number of non-NULL values                       |
| `COUNT(DISTINCT column)`| Number of unique non-NULL values                |
| `SUM(column)`           | Sum of values                                   |
| `AVG(column)`           | Average (ignores NULLs)                         |
| `MIN(column)`           | Minimum value                                   |
| `MAX(column)`           | Maximum value                                   |
| `STRING_AGG(col, ',')`  | Concatenate values (PostgreSQL)                 |
| `ARRAY_AGG(column)`     | Collect values into an array (PostgreSQL)       |
| `BOOL_AND` / `BOOL_OR`  | Logical AND/OR across rows (PostgreSQL)         |

### GROUP BY

```sql
-- count employees per department
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department;

-- average salary per department, only departments with 5+ people
SELECT department, AVG(salary) AS avg_salary, COUNT(*) AS cnt
FROM employees
GROUP BY department
HAVING COUNT(*) >= 5
ORDER BY avg_salary DESC;
```

### `WHERE` vs `HAVING`

- `WHERE` — filters **rows** before grouping.
- `HAVING` — filters **groups** after aggregation.

```sql
-- WHERE filters rows before aggregation
SELECT department, AVG(salary)
FROM employees
WHERE role != 'Intern'      -- exclude interns from the calculation
GROUP BY department
HAVING AVG(salary) > 80000; -- only departments with high avg salary
```

### GROUP BY with Multiple Columns

```sql
SELECT department, role, COUNT(*), AVG(salary)
FROM employees
GROUP BY department, role;
```

**Rule:** every column in `SELECT` that is not inside an aggregate function must appear in `GROUP BY`.

### GROUPING SETS, ROLLUP, CUBE

```sql
-- ROLLUP — subtotals and grand total
SELECT department, role, SUM(salary)
FROM employees
GROUP BY ROLLUP (department, role);
-- produces: (dept, role), (dept, NULL), (NULL, NULL)

-- CUBE — all combinations
SELECT department, role, SUM(salary)
FROM employees
GROUP BY CUBE (department, role);
-- produces: (dept, role), (dept, NULL), (NULL, role), (NULL, NULL)

-- GROUPING SETS — explicit list
SELECT department, role, SUM(salary)
FROM employees
GROUP BY GROUPING SETS (
    (department, role),
    (department),
    ()
);
```

---

## Subqueries

### Scalar Subquery (returns one value)

```sql
SELECT name, salary,
       salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;
```

### Row Subquery (returns one row)

```sql
SELECT * FROM employees
WHERE (department, salary) = (
    SELECT department, MAX(salary) FROM employees WHERE department = 'Engineering'
);
```

### Table Subquery (returns multiple rows)

```sql
-- IN — is value in the result set?
SELECT * FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE location = 'NYC'
);

-- EXISTS — is there at least one matching row? (often faster than IN)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total > 1000
);

-- NOT EXISTS — find customers with no orders
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### Correlated vs Non-Correlated

```sql
-- non-correlated — subquery runs once, independently
SELECT * FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- correlated — subquery runs once per outer row (references outer query)
SELECT e.* FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary) FROM employees e2 WHERE e2.department = e.department
);
```

Correlated subqueries can be slow — the optimizer may rewrite them as joins, or you can rewrite manually.

### Derived Tables (Subquery in FROM)

```sql
SELECT dept_stats.department, dept_stats.avg_salary
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_stats
WHERE dept_stats.avg_salary > 80000;
```

---

## Common Table Expressions (CTE)

### Basic CTE

```sql
WITH dept_stats AS (
    SELECT department, AVG(salary) AS avg_salary, COUNT(*) AS cnt
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, ds.avg_salary
FROM employees e
JOIN dept_stats ds ON e.department = ds.department
WHERE e.salary > ds.avg_salary;
```

### Multiple CTEs

```sql
WITH
    active_orders AS (
        SELECT * FROM orders WHERE status = 'active'
    ),
    order_totals AS (
        SELECT customer_id, SUM(total) AS lifetime_total
        FROM active_orders
        GROUP BY customer_id
    )
SELECT c.name, ot.lifetime_total
FROM customers c
JOIN order_totals ot ON c.id = ot.customer_id
WHERE ot.lifetime_total > 5000;
```

### Recursive CTE

For hierarchical / tree data (org chart, categories, graph traversal):

```sql
-- find all subordinates of a manager (org chart)
WITH RECURSIVE subordinates AS (
    -- base case: the manager
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE id = 1

    UNION ALL

    -- recursive case: employees reporting to someone already in the result
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates ORDER BY depth, name;
```

**Guard against infinite loops:** add a depth limit (`WHERE s.depth < 10`) or use `CYCLE` detection (SQL standard).

---

## Window Functions

Window functions compute a value for each row based on a **window** of related rows — without collapsing rows like
`GROUP BY`.

### Syntax

```sql
function_name(...) OVER (
    [PARTITION BY columns]    -- divide rows into groups (optional)
    [ORDER BY columns]        -- sort within each partition (optional)
    [frame_clause]            -- define the window frame (optional)
)
```

### Ranking Functions

```sql
SELECT name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk,
    NTILE(4)     OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

| Function       | Ties handling                                   | Gaps?  |
|----------------|-------------------------------------------------|--------|
| `ROW_NUMBER()` | Always unique (arbitrary order for ties)        | No     |
| `RANK()`       | Same rank for ties                              | Yes    |
| `DENSE_RANK()` | Same rank for ties                              | No     |
| `NTILE(n)`     | Divides rows into n roughly equal groups        | N/A    |

Example — salaries `100, 90, 90, 80`:

| salary | ROW_NUMBER | RANK | DENSE_RANK |
|--------|-----------|------|------------|
| 100    | 1         | 1    | 1          |
| 90     | 2         | 2    | 2          |
| 90     | 3         | 2    | 2          |
| 80     | 4         | 4    | 3          |

### Aggregate Window Functions

```sql
SELECT name, department, salary,
    SUM(salary)   OVER (PARTITION BY department) AS dept_total,
    AVG(salary)   OVER (PARTITION BY department) AS dept_avg,
    COUNT(*)      OVER (PARTITION BY department) AS dept_count,
    salary::numeric / SUM(salary) OVER (PARTITION BY department) AS salary_share
FROM employees;
```

### Value Functions

```sql
SELECT name, salary, hire_date,
    LAG(salary, 1)  OVER (ORDER BY hire_date) AS prev_salary,
    LEAD(salary, 1) OVER (ORDER BY hire_date) AS next_salary,
    FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date) AS first_hired_salary,
    LAST_VALUE(salary)  OVER (
        PARTITION BY department ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_hired_salary
FROM employees;
```

| Function          | Returns                                             |
|-------------------|-----------------------------------------------------|
| `LAG(col, n)`     | Value from n rows **before** current row            |
| `LEAD(col, n)`    | Value from n rows **after** current row             |
| `FIRST_VALUE(col)`| First value in the window frame                     |
| `LAST_VALUE(col)` | Last value in the window frame (careful with default frame!) |
| `NTH_VALUE(col,n)`| Nth value in the window frame                       |

### Window Frame

The frame defines which rows relative to the current row are included in the computation:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- default for ORDER BY
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING            -- sliding window of 5 rows
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
```

**`ROWS` vs `RANGE` vs `GROUPS`:**
- `ROWS` — physical row count.
- `RANGE` — logical value range (treats ties as the same position).
- `GROUPS` — counts groups of peer rows (PostgreSQL 11+).

**Common gotcha:** `LAST_VALUE` with the default frame (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`) returns
the **current row**, not the actual last. Fix by specifying the full frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND
UNBOUNDED FOLLOWING`.

### Named Window

```sql
SELECT name, salary,
    SUM(salary) OVER w AS running_total,
    AVG(salary) OVER w AS running_avg
FROM employees
WINDOW w AS (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW);
```

### Practical Examples

**Top N per group:**

```sql
-- top 3 highest-paid employees per department
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 3;
```

**Running total:**

```sql
SELECT order_date, amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

**Month-over-month growth:**

```sql
SELECT month, revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))::numeric
        / LAG(revenue) OVER (ORDER BY month) * 100, 1
    ) AS mom_pct
FROM monthly_revenue;
```

**Moving average:**

```sql
SELECT order_date, amount,
    AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
        AS moving_avg_7day
FROM daily_sales;
```

---

## Set Operations

```sql
-- combine results, remove duplicates
SELECT city FROM customers UNION SELECT city FROM suppliers;

-- combine results, keep duplicates
SELECT city FROM customers UNION ALL SELECT city FROM suppliers;

-- rows in both results
SELECT city FROM customers INTERSECT SELECT city FROM suppliers;

-- rows in first but not second
SELECT city FROM customers EXCEPT SELECT city FROM suppliers;
```

All set operations require the same number of columns with compatible types.

---

## CASE Expression

```sql
-- searched CASE
SELECT name, salary,
    CASE
        WHEN salary >= 120000 THEN 'Senior'
        WHEN salary >= 80000  THEN 'Mid'
        ELSE 'Junior'
    END AS level
FROM employees;

-- simple CASE
SELECT name,
    CASE status
        WHEN 'A' THEN 'Active'
        WHEN 'I' THEN 'Inactive'
        WHEN 'S' THEN 'Suspended'
        ELSE 'Unknown'
    END AS status_label
FROM users;

-- CASE in aggregation (pivot-like)
SELECT department,
    COUNT(CASE WHEN role = 'Senior' THEN 1 END) AS seniors,
    COUNT(CASE WHEN role = 'Junior' THEN 1 END) AS juniors
FROM employees
GROUP BY department;

-- FILTER (PostgreSQL alternative to CASE in aggregation)
SELECT department,
    COUNT(*) FILTER (WHERE role = 'Senior') AS seniors,
    COUNT(*) FILTER (WHERE role = 'Junior') AS juniors
FROM employees
GROUP BY department;
```

---

## Data Modification (DML)

### INSERT

```sql
-- single row
INSERT INTO employees (name, department, salary) VALUES ('Alice', 'Engineering', 95000);

-- multiple rows
INSERT INTO employees (name, department, salary) VALUES
    ('Bob', 'Marketing', 70000),
    ('Carol', 'Engineering', 105000);

-- insert from select
INSERT INTO archive_orders (id, customer_id, total)
SELECT id, customer_id, total FROM orders WHERE created_at < '2024-01-01';

-- RETURNING (PostgreSQL) — get inserted row back
INSERT INTO employees (name, salary) VALUES ('Dave', 90000) RETURNING id, name;
```

### UPDATE

```sql
UPDATE employees SET salary = salary * 1.10 WHERE department = 'Engineering';

-- update from another table
UPDATE products p
SET price = np.new_price
FROM new_prices np
WHERE p.id = np.product_id;

-- RETURNING
UPDATE employees SET salary = salary * 1.10 WHERE id = 42 RETURNING id, salary;
```

### DELETE

```sql
DELETE FROM orders WHERE status = 'cancelled' AND created_at < '2024-01-01';

-- delete with join (PostgreSQL)
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id AND o.status = 'cancelled';

-- TRUNCATE — fast delete all rows (no row-level logging, cannot rollback in some DBs)
TRUNCATE TABLE temp_data;
```

### UPSERT (INSERT ON CONFLICT)

```sql
-- PostgreSQL
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

-- do nothing on conflict
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO NOTHING;
```

### MERGE (SQL Standard / PostgreSQL 15+)

```sql
MERGE INTO products target
USING new_products source ON target.sku = source.sku
WHEN MATCHED THEN
    UPDATE SET price = source.price, updated_at = NOW()
WHEN NOT MATCHED THEN
    INSERT (sku, name, price) VALUES (source.sku, source.name, source.price);
```

---

## Table Definition (DDL)

### CREATE TABLE

```sql
CREATE TABLE employees (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    department  VARCHAR(50),
    salary      NUMERIC(10, 2) CHECK (salary > 0),
    manager_id  BIGINT REFERENCES employees(id),
    hire_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Common Data Types (PostgreSQL)

| Type                     | Description                                  |
|--------------------------|----------------------------------------------|
| `INTEGER` / `BIGINT`     | 4 / 8 byte integers                         |
| `SERIAL` / `BIGSERIAL`   | Auto-incrementing integer                    |
| `NUMERIC(p, s)`          | Exact decimal (money, financial data)        |
| `REAL` / `DOUBLE PRECISION` | Floating point (approximate)             |
| `VARCHAR(n)` / `TEXT`    | Variable-length string                       |
| `BOOLEAN`                | true / false                                 |
| `DATE`                   | Date only                                    |
| `TIMESTAMP` / `TIMESTAMPTZ` | Date + time (without / with timezone)    |
| `UUID`                   | Universally unique identifier                |
| `JSONB`                  | Binary JSON (indexable, queryable)           |
| `ARRAY`                  | Array of any type                            |

### Constraints

| Constraint     | Purpose                                                |
|----------------|--------------------------------------------------------|
| `PRIMARY KEY`  | Unique + NOT NULL identifier                           |
| `UNIQUE`       | No duplicate values (NULLs are allowed, each is unique)|
| `NOT NULL`     | Value must be provided                                 |
| `CHECK`        | Custom boolean condition                               |
| `FOREIGN KEY`  | References a row in another table                      |
| `DEFAULT`      | Value when not specified                               |
| `EXCLUDE`      | Exclusion constraint (PostgreSQL — ranges, spatial)    |

### ALTER TABLE

```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees ALTER COLUMN salary SET NOT NULL;
ALTER TABLE employees ALTER COLUMN salary SET DEFAULT 50000;
ALTER TABLE employees RENAME COLUMN name TO full_name;
ALTER TABLE employees ADD CONSTRAINT salary_positive CHECK (salary > 0);
```

---

## EXPLAIN (Query Plans)

### Reading a Query Plan

```sql
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Engineering';
```

```
Seq Scan on employees  (cost=0.00..12.50 rows=5 width=120)
                        (actual time=0.015..0.089 rows=5 loops=1)
  Filter: (department = 'Engineering')
  Rows Removed by Filter: 95
Planning Time: 0.045 ms
Execution Time: 0.112 ms
```

| Field              | Meaning                                                |
|--------------------|--------------------------------------------------------|
| `Seq Scan`         | Full table scan (reads every row)                      |
| `Index Scan`       | Uses an index to find rows                             |
| `Index Only Scan`  | Index covers all needed columns (no table access)      |
| `Bitmap Index Scan`| Builds a bitmap of matching pages, then reads them     |
| `Nested Loop`      | For each row in outer, scan inner (good for small sets)|
| `Hash Join`        | Build hash table from inner, probe with outer          |
| `Merge Join`       | Both sides sorted, merge (good for large sorted sets)  |
| `Sort`             | Sorts rows (watch for `Sort Method: external merge` — spills to disk) |
| `cost`             | Estimated startup..total cost (arbitrary units)        |
| `rows`             | Estimated row count                                    |
| `actual time`      | Real execution time in ms (only with `ANALYZE`)        |
| `Rows Removed`     | Rows that didn't pass the filter                       |

### Key Indicators of Performance Issues

- **Seq Scan on a large table** with a selective `WHERE` → missing index.
- **Estimated rows** differ greatly from **actual rows** → stale statistics (`ANALYZE table`).
- **Nested Loop** with a large inner table → consider Hash Join (might need more `work_mem`).
- **Sort Method: external merge Disk** → increase `work_mem`.
- **Rows Removed by Filter** is very high relative to actual rows → filter is not selective or index is missing.

---

## Common Interview Questions

### What is the difference between `WHERE` and `HAVING`?

`WHERE` filters individual **rows** before aggregation. `HAVING` filters **groups** after `GROUP BY`. You cannot use
aggregate functions in `WHERE` — use `HAVING` instead.

### What is the difference between `INNER JOIN` and `LEFT JOIN`?

`INNER JOIN` returns only rows that match in both tables. `LEFT JOIN` returns all rows from the left table plus
matching rows from the right — unmatched right-side columns are filled with `NULL`.

### How do you find the Nth highest salary?

```sql
-- using DENSE_RANK (handles ties correctly)
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked WHERE rnk = 3;

-- using OFFSET (no ties handling)
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2;
```

### How do you find duplicates?

```sql
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

### How do you delete duplicates keeping one row?

```sql
-- keep the row with the lowest id
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id) FROM users GROUP BY email
);

-- or with a CTE (more efficient for large tables)
WITH ranked AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM users
)
DELETE FROM users WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

### What is the difference between `EXISTS` and `IN`?

Both check membership, but `EXISTS` short-circuits (stops as soon as one match is found) and handles `NULL`
correctly. `IN` compares values and can produce unexpected results with `NULL` in the subquery list.
For large subquery results, `EXISTS` is usually faster. For small static lists, `IN` is fine.

### What is the difference between `UNION` and `UNION ALL`?

`UNION` removes duplicate rows (requires sorting/hashing — slower). `UNION ALL` keeps all rows (faster). Use
`UNION ALL` when you know there are no duplicates or duplicates are acceptable.

### What is a CTE and when would you use one?

A CTE (`WITH ... AS`) is a named temporary result set scoped to a single query. Use for readability (breaking complex
queries into named steps), reusability (reference the same subquery multiple times), and recursion (hierarchical
queries). CTEs are not materialized by default in PostgreSQL 12+ — the optimizer can inline them.

### How do window functions differ from GROUP BY?

`GROUP BY` collapses rows into groups — one output row per group. Window functions compute a value for each row
**without** collapsing — all original rows are preserved. Window functions can access other rows in the partition
(via `LAG`, `LEAD`, `FIRST_VALUE`) which is impossible with plain aggregation.
