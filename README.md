# SQL query optimization - Practical patterns for writing faster, safer SQL.
---

## 1. IN subquery → EXISTS

**Category:** Subquery | **Issue:** Performance

### ❌ Slow — full subquery scan

```sql
SELECT *
FROM employees
WHERE department_id IN (
  SELECT department_id
  FROM departments
  WHERE location = 'New York'
);
```

### ✅ Optimized

```sql
SELECT e.*
FROM employees e
WHERE EXISTS (
  SELECT 1 FROM departments d
  WHERE d.department_id = e.department_id
    AND d.location = 'New York'
);
```

> **Why:** `IN` materializes the entire subquery result set. `EXISTS` short-circuits on the first matching row — much faster on large lookup tables.

---

## 2. NOT IN with NULLs → NOT EXISTS

**Category:** Subquery, NULL safety | **Issue:** Silent bug

### ❌ Bug — returns 0 rows if any NULL exists in subquery

```sql
SELECT *
FROM products
WHERE product_id NOT IN (
  SELECT product_id
  FROM order_details
);
```

### ✅ Optimized

```sql
SELECT p.*
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_details od
  WHERE od.product_id = p.product_id
);
```

> **Why:** If any row in the subquery has a `NULL` product_id, `NOT IN` returns zero rows silently. `NOT EXISTS` is NULL-safe and performs better.

---

## 3. Correlated subquery per-row → window function

**Category:** Window function, Full scan | **Issue:** Critical performance

### ❌ Critical — re-executes for every row (O(n²))

```sql
SELECT employee_id, salary
FROM employees
WHERE salary > (
  SELECT AVG(salary) FROM employees
);
```

### ✅ Optimized

```sql
WITH avg_salary AS (
  SELECT *, AVG(salary) OVER () AS avg_sal
  FROM employees
)
SELECT employee_id, salary
FROM avg_salary
WHERE salary > avg_sal;
```

> **Why:** The slow version scans the entire table once per row. The window function computes the average in a single pass — O(n) vs O(n²).

---

## 4. Per-department comparison → partitioned window

**Category:** Window function | **Issue:** Performance

### ❌ Slow — correlated subquery per department

```sql
SELECT e.employee_id, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary) FROM employees
  WHERE department_id = e.department_id
);
```

### ✅ Optimized

```sql
WITH dept_avg AS (
  SELECT *,
    AVG(salary) OVER (
      PARTITION BY department_id
    ) AS dept_avg_sal
  FROM employees
)
SELECT employee_id, salary, department_id
FROM dept_avg
WHERE salary > dept_avg_sal;
```

> **Why:** Partitioned window functions replace correlated subqueries entirely. One table scan computes all department averages simultaneously.

---

## 5. MAX() subquery + ties → RANK()

**Category:** Window function, Index usage | **Issue:** Performance + correctness

### ❌ Slow — silently drops tied rows

```sql
SELECT order_id, customer_id, order_date
FROM orders
WHERE order_date = (
  SELECT MAX(order_date) FROM orders
);
```

### ✅ Optimized

```sql
SELECT order_id, customer_id, order_date
FROM (
  SELECT *,
    RANK() OVER (
      ORDER BY order_date DESC
    ) AS rnk
  FROM orders
) ranked
WHERE rnk = 1;
```

> **Why:** The original is correct but fragile — multiple rows at MAX may be mishandled by some optimizers. `RANK()` is explicit about tie-handling and semantically clearer.

---

## 6. Deduplication — DISTINCT → ROW_NUMBER()

**Category:** Window function, Full scan | **Issue:** Wrong semantics + performance

### ❌ Wrong — DISTINCT can't control which duplicate to keep

```sql
-- Goal: one row per customer, keeping their most recent order
SELECT DISTINCT
  customer_id,
  order_date,
  total_amount,
  status
FROM orders;
```

### ✅ Optimized — ROW_NUMBER() deduplicates with full control

```sql
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY order_date DESC, order_id DESC
    ) AS rn
  FROM orders
)
SELECT customer_id, order_date, total_amount, status
FROM ranked
WHERE rn = 1;
```

> **Why:** `DISTINCT` collapses rows only when **every selected column matches** — it can never answer "keep the latest per group." `ROW_NUMBER()` partitions by the dedup key and orders by the tiebreak column, giving you deterministic, intentional dedup. It also outperforms `DISTINCT` on large tables since it avoids a full cross-column sort.

### Going further — dedup on write with MERGE (upsert pattern)

A common production scenario: loading a staging table into a target, deduplicating in flight.

```sql
-- ❌ Naive: INSERT + DELETE leaves a gap window, and misses within-batch dupes
INSERT INTO customers_target
SELECT * FROM customers_staging
WHERE customer_id NOT IN (SELECT customer_id FROM customers_target);

-- ✅ MERGE deduplicates within the batch and handles upsert atomically
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
    tgt.email       = src.email,
    tgt.updated_at  = src.updated_at
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, updated_at)
  VALUES (src.customer_id, src.email, src.updated_at);
```

---

## 7. IN → JOIN with dedup risk

**Category:** Subquery, NULL safety | **Issue:** Silent bug

### ❌ Bug — inflates result rows without DISTINCT

```sql
SELECT p.product_id, p.product_name
FROM products p
JOIN order_details od
  ON p.product_id = od.product_id
WHERE od.quantity >= 10;
```

### ✅ Optimized

```sql
SELECT p.product_id, p.product_name
FROM products p
WHERE EXISTS (
  SELECT 1 FROM order_details od
  WHERE od.product_id = p.product_id
    AND od.quantity >= 10
);
```

> **Why:** If a product appears in many order lines, the JOIN duplicates product rows. `EXISTS` returns exactly one row per product with no dedup overhead.

---

## 8. HAVING with aggregate subquery → CTE

**Category:** Window function, Full scan | **Issue:** Performance

### ❌ Slow — triple aggregation, two table scans

```sql
SELECT customer_id, COUNT(*) AS cnt
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > (
  SELECT AVG(order_count) FROM (
    SELECT COUNT(*) AS order_count
    FROM orders GROUP BY customer_id
  ) sub
);
```

### ✅ Optimized

```sql
WITH order_counts AS (
  SELECT customer_id,
    COUNT(*) AS cnt,
    AVG(COUNT(*)) OVER () AS avg_cnt
  FROM orders
  GROUP BY customer_id
)
SELECT customer_id, cnt
FROM order_counts
WHERE cnt > avg_cnt;
```

> **Why:** The original aggregates three times across two table scans. The CTE uses a window function to compute group count and the average of counts in a single pass.

---

## 9. Implicit type cast on indexed column

**Category:** Index usage | **Issue:** Critical — silently disables index

### ❌ Critical — forces cast on every row, ignores index

```sql
SELECT *
FROM events
WHERE user_id = '10045';
```

### ✅ Optimized

```sql
SELECT event_id, event_type, created_at, user_id
FROM events
WHERE user_id = 10045;
```

> **Why:** When `user_id` is `INTEGER` but the literal is `VARCHAR`, the engine casts every row to compare — the index is ignored entirely. Always match literal types to column types exactly.

---

## 10. Function on indexed column in WHERE

**Category:** Index usage, Full scan | **Issue:** Critical — forces full table scan

### ❌ Critical — non-sargable predicate, full scan

```sql
SELECT order_id, order_date
FROM orders
WHERE YEAR(order_date) = 2024
  AND MONTH(order_date) = 3;
```

### ✅ Optimized — sargable range scan

```sql
SELECT order_id, order_date
FROM orders
WHERE order_date >= '2024-03-01'
  AND order_date < '2024-04-01';
```

> **Why:** Wrapping a column in a function (`YEAR`, `MONTH`, `CAST`, `LOWER`) makes the predicate non-sargable — the index cannot be used. Rewrite as a range scan on the raw column.

---

## 11. Repeated subquery in SELECT + WHERE

**Category:** Subquery, Full scan | **Issue:** Performance

### ❌ Slow — identical subquery executes twice

```sql
SELECT product_id,
  (SELECT AVG(unit_price) FROM products) AS avg_price,
  unit_price
FROM products
WHERE unit_price > (
  SELECT AVG(unit_price) FROM products
);
```

### ✅ Optimized

```sql
WITH avg AS (
  SELECT AVG(unit_price) AS avg_price
  FROM products
)
SELECT p.product_id,
       a.avg_price,
       p.unit_price
FROM products p
CROSS JOIN avg a
WHERE p.unit_price > a.avg_price;
```

> **Why:** The identical subquery executes twice — once per row in `SELECT`, once in `WHERE`. Materializing it in a CTE computes it once and reuses the result everywhere.

---

## 12. OR on indexed columns → UNION ALL

**Category:** Index usage, Full scan | **Issue:** Performance

### ❌ Slow — OR disables index merge on many engines

```sql
SELECT order_id, status, customer_id
FROM orders
WHERE status = 'pending'
   OR status = 'failed';
```

### ✅ Optimized

```sql
SELECT order_id, status, customer_id
FROM orders WHERE status = 'pending'
UNION ALL
SELECT order_id, status, customer_id
FROM orders WHERE status = 'failed';
```

> **Why:** Many optimizers cannot efficiently merge indexes for `OR` predicates and fall back to a full scan. Two `UNION ALL` branches each get their own fast index seek. Use `IN()` for simple cases; use `UNION ALL` when branches differ structurally.

---

## Quick Reference

| # | Pattern | Category | Issue type |
|---|---------|----------|------------|
| 1 | `IN` subquery → `EXISTS` | Subquery | Performance |
| 2 | `NOT IN` with NULLs → `NOT EXISTS` | Subquery, NULL | Silent bug |
| 3 | Correlated subquery → window function | Window fn | Critical perf |
| 4 | Per-dept correlated → partitioned window | Window fn | Performance |
| 5 | `MAX()` subquery → `RANK()` | Window fn | Perf + ties |
| 6 | `DISTINCT` dedup → `ROW_NUMBER()` | Window fn | Wrong semantics |
| 7 | `JOIN` without dedup → `EXISTS` | Subquery, NULL | Silent bug |
| 8 | `HAVING` + aggregate subquery → CTE | Window fn | Performance |
| 9 | Implicit type cast on indexed col | Index | Critical bug |
| 10 | Function in `WHERE` → range scan | Index | Critical perf |
| 11 | Repeated subquery → CTE | Subquery | Performance |
| 12 | `OR` on indexed cols → `UNION ALL` | Index | Performance |
