# SQL optimization — Joins, Pagination & Advanced Patterns
---

## 1. Multi-table JOIN — row inflation from missing dedup

**Category:** JOIN correctness | **Issue:** Silent row multiplication

### ❌ Broken — fan-out from one-to-many JOIN inflates results

```sql
-- Orders joined to Products where a product belongs to multiple categories
-- (product_categories is a bridge table)
SELECT o.order_id, o.order_date, p.product_name, c.category_name
FROM orders o
JOIN order_items oi     ON o.order_id    = oi.order_id
JOIN products p         ON oi.product_id = p.product_id
JOIN product_categories pc ON p.product_id = pc.product_id
JOIN categories c       ON pc.category_id = c.category_id
WHERE o.order_date >= '2023-01-01'
ORDER BY o.order_date DESC;
-- If a product has 3 categories, every order line for that product
-- appears 3 times — silently
```

### ✅ Correct — filter categories before joining, or aggregate explicitly

```sql
-- Option A: restrict to one category before the JOIN
SELECT o.order_id, o.order_date, p.product_name, c.category_name
FROM orders o
JOIN order_items oi ON o.order_id    = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
JOIN (
  SELECT DISTINCT product_id, category_id
  FROM product_categories
  WHERE category_id = 5          -- anchor to the category you actually want
) pc ON p.product_id = pc.product_id
JOIN categories c ON pc.category_id = c.category_id
WHERE o.order_date >= '2023-01-01'
ORDER BY o.order_date DESC;

-- Option B: aggregate category names instead of multiplying rows
SELECT o.order_id, o.order_date, p.product_name,
       STRING_AGG(c.category_name, ', ') AS categories
FROM orders o
JOIN order_items oi     ON o.order_id    = oi.order_id
JOIN products p         ON oi.product_id = p.product_id
JOIN product_categories pc ON p.product_id = pc.product_id
JOIN categories c       ON pc.category_id = c.category_id
WHERE o.order_date >= '2023-01-01'
GROUP BY o.order_id, o.order_date, p.product_name
ORDER BY o.order_date DESC;
```

> **Why:** Multi-table JOINs crossing one-to-many relationships silently multiply rows. Adding `SELECT *` or even an explicit column list won't reveal the problem — you have to know your cardinality. Always check: does any JOIN key appear multiple times in that table? If so, either restrict before joining, or group after. `COUNT(DISTINCT order_id)` vs `COUNT(*)` on the result is a fast sanity check.

---

## 2. Subquery with GROUP BY + HAVING → EXISTS

**Category:** Subquery · NULL safety | **Issue:** Duplicate risk + potential NULL bug

### ❌ Risky — IN with grouped subquery can inflate or silently filter

```sql
-- Find customers who placed more than 5 orders in 2023
SELECT *
FROM customers
WHERE customer_id IN (
  SELECT customer_id
  FROM orders
  WHERE order_date >= '2023-01-01'
  GROUP BY customer_id
  HAVING COUNT(*) > 5
);
-- If customer_id is nullable in orders, IN silently drops rows
-- Replacing IN with JOIN can produce duplicate customer rows
```

### ✅ Correct — EXISTS is NULL-safe and returns exactly one row per customer

```sql
SELECT c.customer_id, c.customer_name, c.email
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
    AND o.order_date >= '2023-01-01'
  GROUP BY o.customer_id
  HAVING COUNT(*) > 5
);
```

> **Why:** `IN` with a grouped subquery is fragile: if any `customer_id` in the subquery is NULL, those rows are silently excluded. Replacing `IN` with `JOIN` introduces a different bug — customers with exactly 6 orders each returned once, but the JOIN can still fan out if the subquery isn't perfectly deduplicated. `EXISTS` is NULL-safe, short-circuits on the first qualifying group, and guarantees exactly one output row per customer regardless of cardinality in `orders`.

---

## 3. Running average — correct frame specification

**Category:** Window function | **Issue:** Ambiguous frame default changes behavior across engines

### ❌ Ambiguous — frame behavior differs when ORDER BY is present

```sql
-- Running average of order amount per customer
SELECT customer_id, order_id, order_date,
  AVG(order_amount) OVER (
    PARTITION BY customer_id
    ORDER BY order_date
    -- No ROWS clause: defaults to RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    -- But RANGE uses value-based matching — ties on order_date share the same frame,
    -- producing unexpected averages when dates repeat
  ) AS running_avg
FROM orders;
```

### ✅ Explicit — ROWS clause makes frame deterministic across all engines

```sql
SELECT customer_id, order_id, order_date, order_amount,
  -- True running average: includes all prior rows by physical position
  AVG(order_amount) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id          -- secondary sort for determinism
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND CURRENT ROW
  ) AS running_avg,

  -- Rolling 7-day average: last 6 rows + current
  AVG(order_amount) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
    ROWS BETWEEN 6 PRECEDING
             AND CURRENT ROW
  ) AS rolling_7day_avg

FROM orders;
```

> **Why:** Without a `ROWS` or `RANGE` clause, the frame default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` when `ORDER BY` is present. `RANGE` groups rows with equal `ORDER BY` values into the same frame — so two orders on the same date share a frame and get the same average, which is usually not what you want for a running total. `ROWS` always uses physical row position, making the frame deterministic. Always specify the frame clause explicitly when writing window functions with `ORDER BY`.

---

## 4. OFFSET pagination → keyset (cursor) pagination

**Category:** Index usage · Full scan | **Issue:** Critical — performance degrades linearly with page depth

### ❌ Slow — OFFSET forces the engine to scan and discard all prior rows

```sql
-- Page 100 of results (rows 9901–10000)
SELECT order_id, order_date, customer_id, total_amount
FROM orders
ORDER BY order_date DESC, order_id DESC
OFFSET 9900 ROWS FETCH NEXT 100 ROWS ONLY;
-- The engine must sort and traverse all 9900 prior rows just to discard them
-- Page load time grows linearly with page number
```

### ✅ Keyset pagination — seeks directly to the next page boundary

```sql
-- Client stores the last (order_date, order_id) from the previous page
-- e.g. last_date = '2023-06-15', last_id = 48291

SELECT order_id, order_date, customer_id, total_amount
FROM orders
WHERE (order_date, order_id) < ('2023-06-15', 48291)  -- PostgreSQL tuple syntax
ORDER BY order_date DESC, order_id DESC
LIMIT 100;

-- SQL Server / MySQL equivalent (no tuple comparison):
SELECT order_id, order_date, customer_id, total_amount
FROM orders
WHERE order_date < '2023-06-15'
   OR (order_date = '2023-06-15' AND order_id < 48291)
ORDER BY order_date DESC, order_id DESC
FETCH NEXT 100 ROWS ONLY;
```

> **Why:** `OFFSET N` forces the optimizer to generate and discard N rows before returning results — page 1 is fast, page 1000 is 1000x slower. Keyset pagination passes the last-seen values as a `WHERE` predicate, letting the engine seek directly to the next page boundary via the index on `(order_date DESC, order_id DESC)`. This gives O(1) page retrieval regardless of depth. Trade-off: you lose random page access (no "jump to page 47") — keyset is sequential only. Use `OFFSET` only for small, bounded datasets or admin UIs where page depth is limited.

---

## 5. Shallow IN subquery vs full org traversal — knowing what you need

**Category:** Recursive CTE · Correctness | **Issue:** Wrong tool for the actual problem

### ❌ Wrong for deep hierarchies — but also commonly misreplaced

```sql
-- Find all employees who report directly to anyone in department 10
SELECT employee_id, employee_name
FROM employees
WHERE supervisor_id IN (
  SELECT employee_id
  FROM employees
  WHERE department_id = 10
);
-- This is correct for ONE level of depth only.
-- Replacing this with a recursive CTE for the same goal is over-engineering.
```

### ✅ Match the tool to the actual requirement

```sql
-- If you need EXACTLY one level (direct reports): the original IN is fine.
-- Rewrite it as EXISTS for NULL safety:
SELECT e.employee_id, e.employee_name
FROM employees e
WHERE EXISTS (
  SELECT 1 FROM employees mgr
  WHERE mgr.employee_id  = e.supervisor_id
    AND mgr.department_id = 10
);

-- If you need ALL descendants at ANY depth: use a recursive CTE
WITH org_tree AS (
  -- Anchor: start from department 10 managers
  SELECT employee_id, employee_name, supervisor_id, 1 AS depth
  FROM employees
  WHERE department_id = 10

  UNION ALL

  -- Recursive: walk down the reporting chain
  SELECT e.employee_id, e.employee_name, e.supervisor_id, ot.depth + 1
  FROM employees e
  JOIN org_tree ot ON e.supervisor_id = ot.employee_id
)
SELECT employee_id, employee_name, depth
FROM org_tree
ORDER BY depth, employee_name;
```

> **Why:** A recursive CTE for a one-level lookup is over-engineering that hurts readability and performance. Conversely, using `IN` for multi-level org traversal silently returns only direct reports when the business requirement is all descendants. The key question is: **does the depth vary?** Fixed depth → `EXISTS`. Variable/unlimited depth → recursive CTE. Always add a `depth` counter and a `MAXRECURSION` guard in production recursive CTEs to prevent infinite loops on bad data (circular `supervisor_id` references).

---

## 6. CASE with redundant WHERE filter → filtered aggregation

**Category:** Conditional logic · Performance | **Issue:** Logic error disguised as optimization

### ❌ Wrong — the added WHERE condition is a tautology

```sql
-- Categorize orders by amount
SELECT order_id,
  CASE
    WHEN order_amount > 1000 THEN 'High'
    WHEN order_amount > 500  THEN 'Medium'
    ELSE                          'Low'
  END AS order_category
FROM orders
WHERE order_date >= '2023-01-01'
  AND (order_amount > 1000 OR order_amount > 500);
  -- This simplifies to: AND order_amount > 500
  -- It silently excludes all 'Low' category orders — different result set
```

### ✅ Real optimization — use CASE inside aggregation with FILTER

```sql
-- If the goal is to COUNT or SUM by category, use filtered aggregation
-- instead of scanning once and grouping afterwards
SELECT
  COUNT(*) FILTER (WHERE order_amount > 1000)               AS high_count,
  COUNT(*) FILTER (WHERE order_amount BETWEEN 501 AND 1000) AS medium_count,
  COUNT(*) FILTER (WHERE order_amount <= 500)               AS low_count,
  SUM(order_amount) FILTER (WHERE order_amount > 1000)      AS high_revenue
FROM orders
WHERE order_date >= '2023-01-01';

-- For row-level categorization with no filter needed, CASE is already optimal:
SELECT order_id,
  CASE
    WHEN order_amount > 1000 THEN 'High'
    WHEN order_amount > 500  THEN 'Medium'
    ELSE                          'Low'
  END AS order_category
FROM orders
WHERE order_date >= '2023-01-01';
```

> **Why:** Adding `AND (order_amount > 1000 OR order_amount > 500)` to a query that CASE-classifies all rows is a logic error — it silently drops the 'Low' category. The real optimization for CASE-based classification is to push conditions into **filtered aggregation** using `FILTER (WHERE ...)` (PostgreSQL, DuckDB) or `SUM(CASE WHEN ... END)` (universal). This eliminates a separate `GROUP BY` pass and computes all buckets in one scan. For row-level CASE without aggregation, there is no meaningful optimization — CASE short-circuits on the first matching branch already.

---

## 7. EXISTS → JOIN regression — when EXISTS is already correct

**Category:** Subquery · JOIN correctness | **Issue:** Replacing EXISTS with JOIN introduces duplicates

### ❌ Regression — JOIN produces one row per matching order, not per customer

```sql
-- Find customers who placed at least one order in 2023
-- Incorrectly "optimized" by replacing EXISTS with JOIN:
SELECT c.customer_id, c.customer_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01';
-- A customer with 10 orders in 2023 appears 10 times
-- Requires DISTINCT to fix — which then adds a sort step
```

### ✅ EXISTS is already the right answer — keep it

```sql
-- EXISTS: one row per customer, short-circuits on first qualifying order
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
    AND o.order_date >= '2023-01-01'
);
```

> **Why:** `EXISTS` short-circuits on the first matching row — it's already the most efficient construct for "does at least one related row satisfy this condition." Replacing it with `JOIN` forces the engine to find and return all matching rows before deduplication. Adding `DISTINCT` to the JOIN version fixes correctness but reintroduces a sort step that `EXISTS` never needed. The rule is: use `JOIN` when you need columns from the joined table; use `EXISTS` when you only need to test for the presence of a match.

---

## 8. UNION vs UNION ALL — correctness before performance

**Category:** Set operations | **Issue:** UNION ALL is faster, but only correct when duplicates are impossible

### ❌ Slower — UNION deduplicates even when rows can't overlap

```sql
-- Combine current and archived orders for reporting
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date >= '2023-01-01'
UNION
SELECT order_id, order_date, total_amount
FROM archived_orders
WHERE order_date >= '2023-01-01';
-- UNION sorts the full result set to find and remove duplicates
-- On millions of rows, this sort is expensive
```

### ✅ UNION ALL — skip the sort, but only when you've verified no overlap

```sql
-- Safe only when orders and archived_orders are mutually exclusive
-- (e.g. archival process moves rows, doesn't copy them)
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date >= '2023-01-01'
UNION ALL
SELECT order_id, order_date, total_amount
FROM archived_orders
WHERE order_date >= '2023-01-01';

-- If overlap IS possible, keep UNION — or deduplicate intentionally:
WITH combined AS (
  SELECT order_id, order_date, total_amount, 'active'   AS source FROM orders
  UNION ALL
  SELECT order_id, order_date, total_amount, 'archived' AS source FROM archived_orders
)
SELECT DISTINCT ON (order_id)   -- PostgreSQL: keep first occurrence
  order_id, order_date, total_amount, source
FROM combined
ORDER BY order_id, source;      -- 'active' sorts before 'archived'
```

> **Why:** `UNION` performs a full sort of the combined result to eliminate duplicates — expensive at scale. `UNION ALL` skips the sort entirely. But switching blindly to `UNION ALL` on tables that can share rows introduces silent duplicates in reports. Always confirm mutual exclusivity first: do the two tables share a primary key space? Is the archival process a move or a copy? Document the assumption in a comment so the next engineer doesn't blindly change it back.

---

## 9. RANK() vs ROW_NUMBER() — choosing the right semantic

**Category:** Window function · Correctness | **Issue:** Swapping them silently changes results on tied rows

### ❌ Wrong — ROW_NUMBER() arbitrarily discards tied rows

```sql
-- Get the top-ranked order per customer by amount
-- "Optimized" by swapping RANK() for ROW_NUMBER():
SELECT customer_id, order_id, order_amount
FROM (
  SELECT customer_id, order_id, order_amount,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY order_amount DESC
    ) AS rn
  FROM orders
) ranked
WHERE rn = 1;
-- If two orders tie for the highest amount, one is silently discarded
-- The choice of which survives is non-deterministic
```

### ✅ Match the function to the business requirement

```sql
-- Need exactly ONE row per customer regardless of ties?
-- → ROW_NUMBER() with a deterministic tiebreaker
SELECT customer_id, order_id, order_amount
FROM (
  SELECT customer_id, order_id, order_amount,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY order_amount DESC, order_id DESC  -- order_id breaks the tie
    ) AS rn
  FROM orders
) ranked
WHERE rn = 1;

-- Need ALL rows that share the top rank (ties preserved)?
-- → RANK() or DENSE_RANK()
SELECT customer_id, order_id, order_amount
FROM (
  SELECT customer_id, order_id, order_amount,
    RANK() OVER (
      PARTITION BY customer_id
      ORDER BY order_amount DESC
    ) AS rnk
  FROM orders
) ranked
WHERE rnk = 1;

-- RANK() vs DENSE_RANK() when used for filtering past rank 1:
-- RANK()       → gaps: 1, 1, 3, 4  (two rows tied at 1, next is 3)
-- DENSE_RANK() → no gaps: 1, 1, 2, 3  (safer for "top N" filters)
```

> **Why:** `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` are not interchangeable. `ROW_NUMBER()` assigns a unique position to every row — ties are broken arbitrarily unless you add a deterministic tiebreaker column to `ORDER BY`. `RANK()` preserves ties but creates gaps in the sequence. `DENSE_RANK()` preserves ties without gaps. Swapping between them to "optimize" silently changes which rows are returned. Choose based on the business rule: one winner per group → `ROW_NUMBER()` with tiebreaker; all co-winners → `RANK()`; top-N buckets → `DENSE_RANK()`.

---

## 10. LIKE leading wildcard → full-text search

**Category:** Index usage · Full scan | **Issue:** Leading wildcard disables B-tree index entirely

### ❌ Non-sargable — leading wildcard forces full table scan

```sql
-- Search for products containing 'Electronics' anywhere in category_name
SELECT p.product_id, p.product_name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE c.category_name LIKE '%Electronics%';
-- The leading % means the index on category_name cannot be used
-- Full scan on every query, scales linearly with table size
```

### ✅ Full-text search — index-backed substring and relevance search

```sql
-- PostgreSQL: use GIN index with pg_trgm for arbitrary substring match
CREATE INDEX idx_category_name_trgm
  ON categories USING GIN (category_name gin_trgm_ops);

-- Query is now index-backed even with leading wildcard
SELECT p.product_id, p.product_name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE c.category_name LIKE '%Electronics%';  -- uses trigram index automatically

-- For relevance ranking (results ordered by match quality):
SELECT p.product_id, p.product_name, c.category_name,
       similarity(c.category_name, 'Electronics') AS score
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE c.category_name % 'Electronics'          -- trigram similarity operator
ORDER BY score DESC;

-- SQL Server equivalent: full-text index + CONTAINS
-- CREATE FULLTEXT INDEX ON categories(category_name) ...
SELECT p.product_id, p.product_name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE CONTAINS(c.category_name, '"Electronics*"');
```

> **Why:** A B-tree index on `category_name` can only be used when the engine knows the leading characters — `LIKE 'Elec%'` uses the index, `LIKE '%Electronics%'` cannot. For arbitrary substring or fuzzy search, the correct fix is a **trigram index** (`pg_trgm` in PostgreSQL) or a **full-text index** (SQL Server, MySQL). These split strings into overlapping character n-grams and index them, allowing substring lookups in O(log n). Full-text search also adds relevance ranking, stemming, and stop-word filtering that `LIKE` can never provide.

---

## Quick Reference

| # | Pattern | Category | Issue |
|---|---------|----------|-------|
| 1 | Multi-table JOIN fan-out → dedup or aggregate | JOIN correctness | Silent row multiplication |
| 2 | GROUP BY + HAVING subquery → EXISTS | NULL safety | Duplicates + NULL bug |
| 3 | Implicit window frame → explicit ROWS clause | Window fn | Non-deterministic results |
| 4 | OFFSET pagination → keyset pagination | Index | O(n) page cost |
| 5 | Single-level IN vs recursive CTE | Correctness | Wrong depth assumption |
| 6 | Tautological WHERE filter → FILTER aggregation | Logic error | Silently drops rows |
| 7 | EXISTS → JOIN regression | JOIN correctness | Introduces duplicates |
| 8 | UNION → UNION ALL with correctness check | Set ops | Silent dupes if overlap exists |
| 9 | ROW_NUMBER() vs RANK() vs DENSE_RANK() | Window fn | Wrong tie semantics |
| 10 | LIKE leading wildcard → trigram / full-text index | Index | Full scan on every query |
