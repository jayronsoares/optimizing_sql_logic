
### 1. Complex JOIN Query

**Worst-Case Scenario:**
```sql
SELECT *
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
JOIN Products p ON o.product_id = p.product_id
WHERE o.order_date >= '2023-01-01'
  AND c.country = 'USA'
  AND p.category = 'Electronics'
ORDER BY o.order_date DESC;
```

**Optimized Version:**
```sql
SELECT o.order_id, o.order_date, c.customer_name, p.product_name
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
JOIN Products p ON o.product_id = p.product_id
WHERE o.order_date >= '2023-01-01'
  AND c.country = 'USA'
  AND p.category = 'Electronics'
ORDER BY o.order_date DESC
LIMIT 100;
```

**Optimization Techniques:** 
- Selected only necessary columns (`o.order_id`, `o.order_date`, `c.customer_name`, `p.product_name`).
- Added `LIMIT` to paginate results and reduce data retrieval.

### 2. Subquery with Aggregation

**Worst-Case Scenario:**
```sql
SELECT *
FROM Customers
WHERE customer_id IN (
    SELECT customer_id
    FROM Orders
    WHERE order_date >= '2023-01-01'
    GROUP BY customer_id
    HAVING COUNT(*) > 5
);
```

**Optimized Version:**
```sql
SELECT c.*
FROM Customers c
JOIN (
    SELECT customer_id
    FROM Orders
    WHERE order_date >= '2023-01-01'
    GROUP BY customer_id
    HAVING COUNT(*) > 5
) o ON c.customer_id = o.customer_id;
```

**Optimization Techniques:** 
- Replaced `IN` with `JOIN` to avoid correlated subquery.
- Selected only necessary columns (`c.*`).

### 3. Aggregates and Window Functions

**Worst-Case Scenario:**
```sql
SELECT customer_id,
       AVG(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS avg_order_amount
FROM Orders;
```

**Optimized Version:**
```sql
WITH avg_orders AS (
    SELECT customer_id,
           AVG(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS avg_order_amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM Orders
)
SELECT customer_id, avg_order_amount
FROM avg_orders
WHERE rn = 1;
```

**Optimization Techniques:** 
- Used a Common Table Expression (CTE) to compute aggregates and window functions efficiently.
- Filtered results using `ROW_NUMBER()` to get the latest average order amount per customer.

### 4. Complex Filtering and Subquery

**Worst-Case Scenario:**
```sql
SELECT *
FROM Products
WHERE category_id IN (
    SELECT category_id
    FROM Categories
    WHERE category_name LIKE 'Electronics%'
);
```

**Optimized Version:**
```sql
SELECT p.*
FROM Products p
JOIN Categories c ON p.category_id = c.category_id
WHERE c.category_name LIKE 'Electronics%';
```

**Optimization Techniques:** 
- Replaced `IN` with `JOIN` for better performance.
- Reduced unnecessary subquery execution.

### 5. Handling Large Datasets with Pagination

**Worst-Case Scenario:**
```sql
SELECT *
FROM Orders
ORDER BY order_date DESC
OFFSET 1000 ROWS FETCH NEXT 100 ROWS ONLY;
```

**Optimized Version:**
```sql
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY order_date DESC) AS rn
    FROM Orders
) AS o
WHERE rn BETWEEN 1001 AND 1100;
```

**Optimization Techniques:** 
- Used `ROW_NUMBER()` for efficient pagination.
- Avoided using `OFFSET` which can be inefficient for large offsets.

### 6. Handling Hierarchical Data

**Worst-Case Scenario:**
```sql
SELECT employee_id, employee_name
FROM Employees
WHERE supervisor_id IN (
    SELECT employee_id
    FROM Employees
    WHERE department_id = 10
);
```

**Optimized Version:**
```sql
WITH recursive supervisor_chain AS (
    SELECT employee_id, employee_name, supervisor_id
    FROM Employees
    WHERE department_id = 10
    UNION ALL
    SELECT e.employee_id, e.employee_name, e.supervisor_id
    FROM Employees e
    JOIN supervisor_chain sc ON e.supervisor_id = sc.employee_id
)
SELECT employee_id, employee_name
FROM supervisor_chain;
```

**Optimization Techniques:** 
- Used recursive Common Table Expression (CTE) for hierarchical querying.
- Avoided nested subquery for better performance.

### 7. Complex Filtering with Case Statements

**Worst-Case Scenario:**
```sql
SELECT order_id,
       CASE 
           WHEN order_amount > 1000 THEN 'High'
           WHEN order_amount > 500 THEN 'Medium'
           ELSE 'Low'
       END AS order_category
FROM Orders
WHERE order_date >= '2023-01-01';
```

**Optimized Version:**
```sql
SELECT order_id,
       CASE 
           WHEN order_amount > 1000 THEN 'High'
           WHEN order_amount > 500 THEN 'Medium'
           ELSE 'Low'
       END AS order_category
FROM Orders
WHERE order_date >= '2023-01-01'
  AND (order_amount > 1000 OR order_amount > 500);
```

**Optimization Techniques:** 
- Optimized the `WHERE` clause to avoid redundant conditions inside the `CASE` statement.

### 8. Using EXISTS for Efficient Subqueries

**Worst-Case Scenario:**
```sql
SELECT *
FROM Customers c
WHERE EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.customer_id = c.customer_id
      AND o.order_date >= '2023-01-01'
);
```

**Optimized Version:**
```sql
SELECT c.*
FROM Customers c
JOIN Orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01';
```

**Optimization Techniques:** 
- Replaced `EXISTS` with `JOIN` for improved performance.

### 9. Using UNION for Combined Queries

**Worst-Case Scenario:**
```sql
SELECT order_id, order_date
FROM Orders
WHERE order_date >= '2023-01-01'
UNION
SELECT order_id, order_date
FROM Archived_Orders
WHERE order_date >= '2023-01-01';
```

**Optimized Version:**
```sql
SELECT order_id, order_date
FROM Orders
WHERE order_date >= '2023-01-01'
UNION ALL
SELECT order_id, order_date
FROM Archived_Orders
WHERE order_date >= '2023-01-01';
```

**Optimization Techniques:** 
- Used `UNION ALL` instead of `UNION` to avoid duplicate elimination which can be costly.

### 10. Using Window Functions for Ranking

**Worst-Case Scenario:**
```sql
SELECT customer_id, order_id, order_date,
       RANK() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rank
FROM Orders;
```

**Optimized Version:**
```sql
SELECT customer_id, order_id, order_date
FROM (
    SELECT customer_id, order_id, order_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM Orders
) AS ranked_orders
WHERE rn = 1;
```

**Optimization Techniques:** 
- Used `ROW_NUMBER()` instead of `RANK()` for cases where exact row position is needed.

<p>These examples illustrate how optimizing SQL queries involves thoughtful consideration of indexing, query structure, and efficient use of SQL constructs.</p>
<p></p>Depending on your specific database system (e.g., MySQL, PostgreSQL, SQL Server), further optimizations such as query plan analysis and database configuration tuning may also be necessary to achieve optimal performance.</p>
