# SQL query optimization — common mistakes

## 1. NOT IN with NULLs → NOT EXISTS

**Category:** Subquery · NULL safety | **Issue:** Silent bug — returns 0 rows

### ❌ Broken — silently returns empty result set

```sql
-- Find products that have never been ordered
SELECT product_id, product_name
FROM products
WHERE product_id NOT IN (
  SELECT product_id        -- if ANY row here is NULL,
  FROM order_details       -- the entire result is empty
);
```

### ✅ Optimized — NULL-safe with NOT EXISTS

```sql
SELECT p.product_id, p.product_name
FROM products p
WHERE NOT EXISTS (
  SELECT 1
  FROM order_details od
  WHERE od.product_id = p.product_id
);
```

> **Why:** SQL's three-valued logic means `x NOT IN (1, 2, NULL)` always evaluates to `UNKNOWN`, never `TRUE` — so zero rows pass the filter. This is one of the most common silent data bugs in production. `NOT EXISTS` evaluates the predicate per row and is fully NULL-safe. It also short-circuits on the first match, making it faster on large sets.

---

## 2. Correlated subquery per-row → window function

**Category:** Window function · Full scan | **Issue:** Critical — O(n²) complexity

### ❌ Critical — subquery re-executes for every row

```sql
-- Find employees earning above the company average
SELECT employee_id, department_id, salary
FROM employees e
WHERE salary > (
  SELECT AVG(salary)       -- full table scan
  FROM employees           -- repeated once per row
);
```

### ✅ Optimized — single pass with window function

```sql
WITH company_avg AS (
  SELECT *,
    AVG(salary) OVER () AS avg_salary
  FROM employees
)
SELECT employee_id, department_id, salary
FROM company_avg
WHERE salary > avg_salary;
```

> **Why:** The correlated subquery triggers a full scan of `employees` once for each row being evaluated — O(n²). The window function `AVG() OVER ()` computes the global average in a single pass alongside the main scan. At 1M rows the difference is seconds vs hours.

---

## 3. Per-group comparison → partitioned window

**Category:** Window function | **Issue:** Performance — correlated scan per group

### ❌ Slow — one subquery execution per employee row

```sql
-- Find employees earning above their own department's average
SELECT e.employee_id, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary)
  FROM employees
  WHERE department_id = e.department_id  -- re-runs per row
);
```

### ✅ Optimized — all department averages in one scan

```sql
WITH dept_avg AS (
  SELECT *,
    AVG(salary) OVER (
      PARTITION BY department_id
    ) AS dept_avg_salary
  FROM employees
)
SELECT employee_id, salary, department_id
FROM dept_avg
WHERE salary > dept_avg_salary;
```

> **Why:** The correlated version re-scans the table for every unique `department_id` it encounters per row. `PARTITION BY` computes all group averages simultaneously in a single pass. The gain compounds with cardinality — the more departments, the bigger the win.

---

## 4. MAX() subquery with ties → RANK()

**Category:** Window function · Index usage | **Issue:** Correctness — silently drops tied rows

### ❌ Fragile — ties at MAX are silently lost

```sql
-- Get the most recent orders
-- If two orders share the same order_date, only one (arbitrary) is returned
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE order_date = (
  SELECT MAX(order_date) FROM orders
);
```

### ✅ Correct — ties preserved with RANK()

```sql
SELECT order_id, customer_id, order_date, total_amount
FROM (
  SELECT *,
    RANK() OVER (ORDER BY order_date DESC) AS rnk
  FROM orders
) ranked
WHERE rnk = 1;
```

> **Why:** The subquery version is logically correct only when MAX is unique. In practice, ties are common (e.g. batch jobs, same-day orders) and are silently dropped. `RANK()` is semantically explicit: every row tied at rank 1 is returned. Use `ROW_NUMBER()` instead if you intentionally want exactly one row regardless of ties.

---

## 5. DISTINCT deduplication → ROW_NUMBER()

**Category:** Window function · Full scan | **Issue:** Wrong semantics + performance

### ❌ Wrong — DISTINCT can't control which duplicate survives

```sql
-- Goal: one row per customer, keeping their most recent order
SELECT DISTINCT
  customer_id,
  order_date,
  total_amount,
  status
FROM orders;
-- DISTINCT collapses rows only when ALL columns match — this returns
-- one row per unique (customer_id, order_date, total_amount, status) combo,
-- not one row per customer
```

### ✅ Optimized — deterministic dedup with ROW_NUMBER()

```sql
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY order_date DESC, order_id DESC  -- secondary sort for true determinism
    ) AS rn
  FROM orders
)
SELECT customer_id, order_date, total_amount, status
FROM ranked
WHERE rn = 1;
```

### Going further — dedup during staging load with MERGE

```sql
-- ❌ Naive load: race condition window + misses within-batch duplicates
INSERT INTO customers_target
SELECT * FROM customers_staging
WHERE customer_id NOT IN (SELECT customer_id FROM customers_target);

-- ✅ MERGE: deduplicates within the batch and upserts atomically
MERGE INTO customers_target AS tgt
USING (
  SELECT *
  FROM (
    SELECT *,
      ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY updated_at DESC
      ) AS rn
    FROM customers_staging
  ) s
  WHERE rn = 1
) AS src
ON tgt.customer_id = src.customer_id
WHEN MATCHED THEN
  UPDATE SET
    tgt.email      = src.email,
    tgt.updated_at = src.updated_at
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, updated_at)
  VALUES (src.customer_id, src.email, src.updated_at);
```

> **Why:** `DISTINCT` deduplicates across every selected column — it cannot express "keep the latest per group." `ROW_NUMBER()` with `PARTITION BY` gives full control over which row survives. The MERGE pattern solves the real production problem: loading a staging table safely without leaving a gap window or losing within-batch duplicates.

---

## 6. HAVING with nested aggregate → CTE + window

**Category:** Window function · Full scan | **Issue:** Performance — triple aggregation

### ❌ Slow — three aggregations, two full scans

```sql
-- Find customers who order more than average
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > (
  SELECT AVG(order_count)           -- 3rd aggregation
  FROM (
    SELECT COUNT(*) AS order_count  -- 2nd aggregation + 2nd full scan
    FROM orders
    GROUP BY customer_id
  ) sub
);
```

### ✅ Optimized — single scan, one aggregation pass

```sql
WITH order_counts AS (
  SELECT
    customer_id,
    COUNT(*)          AS order_count,
    AVG(COUNT(*)) OVER () AS avg_order_count  -- window over grouped result
  FROM orders
  GROUP BY customer_id
)
SELECT customer_id, order_count
FROM order_counts
WHERE order_count > avg_order_count;
```

> **Why:** The original forces the engine to scan `orders` twice and aggregate three times. `AVG(COUNT(*)) OVER ()` is a window function applied to the already-grouped result — the outer average is computed in the same pass as the `GROUP BY`, collapsing three operations into one.

---

## 7. Implicit type cast on indexed column

**Category:** Index usage | **Issue:** Critical — index silently disabled

### ❌ Critical — engine casts every row, index ignored

```sql
-- user_id is INTEGER, but literal is VARCHAR
SELECT event_id, event_type, created_at
FROM events
WHERE user_id = '10045';   -- implicit CAST(user_id AS VARCHAR) per row
```

### ✅ Optimized — literal matches column type, index used

```sql
SELECT event_id, event_type, created_at
FROM events
WHERE user_id = 10045;     -- integer literal, direct index seek
```

> **Why:** When the literal type doesn't match the column type, the engine must cast the column value on every row to make them comparable — turning an index seek into a full scan. The index is on the raw column values, not on their cast representations. This also applies to `VARCHAR` columns queried with integer literals, date columns cast with `CONVERT()`, etc. Always match your literal's type to the column's declared type.

---

## 8. Non-sargable predicate → range scan

**Category:** Index usage · Full scan | **Issue:** Critical — function call blocks index

### ❌ Non-sargable — function wrapping prevents index use

```sql
-- Querying March 2024 orders using date functions
SELECT order_id, order_date, total_amount
FROM orders
WHERE YEAR(order_date)  = 2024   -- full scan: index on order_date is useless
  AND MONTH(order_date) = 3;
```

### ✅ Sargable — range predicate on raw column enables index seek

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date >= '2024-03-01'
  AND order_date <  '2024-04-01';
```

> **Why:** A predicate is sargable (Search ARGument ABLE) only when the indexed column appears alone on one side of the operator. Wrapping a column in any function — `YEAR()`, `MONTH()`, `CAST()`, `LOWER()`, `DATEPART()`, `COALESCE()` — forces the engine to evaluate every row before filtering can happen. Rewriting as a half-open range `[start, end)` on the raw column gives the optimizer a clean range scan. The same pattern applies to string normalization: `LOWER(email) = 'x'` → `email = 'x'` (enforce case at write time via a check constraint or collation instead).

---

## 9. Repeated subquery in SELECT + WHERE → CTE

**Category:** Subquery · Full scan | **Issue:** Performance — same query executes twice

### ❌ Slow — identical subquery runs once per context

```sql
-- Show each product alongside the catalog average price
SELECT
  product_id,
  unit_price,
  (SELECT AVG(unit_price) FROM products) AS avg_price,  -- scan #1
  unit_price - (SELECT AVG(unit_price) FROM products)   -- scan #2
    AS diff_from_avg
FROM products
WHERE unit_price > (
  SELECT AVG(unit_price) FROM products                  -- scan #3
);
```

### ✅ Optimized — compute once, reference everywhere

```sql
WITH price_avg AS (
  SELECT AVG(unit_price) AS avg_price
  FROM products                           -- single scan
)
SELECT
  p.product_id,
  p.unit_price,
  a.avg_price,
  p.unit_price - a.avg_price AS diff_from_avg
FROM products p
CROSS JOIN price_avg a
WHERE p.unit_price > a.avg_price;
```

> **Why:** Each identical scalar subquery is treated as an independent expression by most optimizers and executes separately. A CTE materializes the result once. `CROSS JOIN` on a single-row CTE is a zero-cost operation — the planner folds it into a constant. This pattern generalizes to any repeated aggregation: threshold values, lookup constants, configuration rows.

---

## 10. OR on indexed columns → UNION ALL

**Category:** Index usage · Full scan | **Issue:** Performance — OR disables index merge

### ❌ Slow — optimizer falls back to full scan on OR

```sql
-- Fetch all actionable orders across two statuses
SELECT order_id, customer_id, status, created_at
FROM orders
WHERE status = 'pending'
   OR status = 'failed';
-- Most engines cannot efficiently merge two index ranges for OR
-- and resort to a full table scan
```

### ✅ Optimized — each branch gets its own index seek

```sql
SELECT order_id, customer_id, status, created_at
FROM orders
WHERE status = 'pending'

UNION ALL

SELECT order_id, customer_id, status, created_at
FROM orders
WHERE status = 'failed';
```

> **Why:** Index merge optimization for `OR` is unreliable across engines — PostgreSQL, MySQL, and SQL Server all handle it differently, and many fall back to a full scan when the estimated row count is high. Two `UNION ALL` branches each resolve independently as fast index seeks. Use `IN ('pending', 'failed')` for simple same-column equality (most optimizers handle it well); prefer `UNION ALL` when the two branches have different filter columns, different sort orders, or different JOIN structures.

---

## Quick Reference

| # | Pattern | Category | Issue |
|---|---------|----------|-------|
| 1 | `NOT IN` + NULLs → `NOT EXISTS` | NULL safety | Silent bug |
| 2 | Correlated subquery → window function | Window fn | O(n²) scan |
| 3 | Per-group correlated → `PARTITION BY` | Window fn | Performance |
| 4 | `MAX()` subquery → `RANK()` | Window fn | Drops ties |
| 5 | `DISTINCT` dedup → `ROW_NUMBER()` + MERGE | Window fn | Wrong semantics |
| 6 | `HAVING` nested aggregate → CTE + window | Window fn | Triple scan |
| 7 | Implicit type cast on index column | Index | Disables index |
| 8 | Function in `WHERE` → range scan | Index | Non-sargable |
| 9 | Repeated subquery → CTE + `CROSS JOIN` | Subquery | Redundant scans |
| 10 | `OR` on indexed cols → `UNION ALL` | Index | Index merge fail |
