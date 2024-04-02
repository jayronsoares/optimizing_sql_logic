# Optimizing SQL Queries for Improved Performance

In the realm of database querying, optimizing SQL queries is crucial to enhance both performance and readability. While subqueries are commonly employed in SQL queries, they can occasionally result in inefficient execution plans. 
Below, we present 20 examples of SQL queries that have been optimized by employing more efficient alternatives.

1. **Original Query:**
   ```sql
   SELECT *
   FROM employees
   WHERE department_id IN (SELECT department_id FROM departments WHERE location = 'New York');
   ```
   **Optimized Query:**
   ```sql
   SELECT e.*
   FROM employees e
   JOIN departments d ON e.department_id = d.department_id
   WHERE d.location = 'New York';
   ```

2. **Original Query:**
   ```sql
   SELECT AVG(salary)
   FROM employees
   WHERE hire_date >= (SELECT MAX(hire_date) FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT AVG(salary)
   FROM employees
   WHERE hire_date >= (SELECT hire_date FROM employees ORDER BY hire_date DESC LIMIT 1);
   ```

3. **Original Query:**
   ```sql
   SELECT *
   FROM orders
   WHERE customer_id IN (SELECT customer_id FROM customers WHERE country = 'USA');
   ```
   **Optimized Query:**
   ```sql
   SELECT o.*
   FROM orders o
   JOIN customers c ON o.customer_id = c.customer_id
   WHERE c.country = 'USA';
   ```

4. **Original Query:**
   ```sql
   SELECT employee_id, hire_date, salary
   FROM employees
   WHERE salary > (SELECT AVG(salary) FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT employee_id, hire_date, salary
   FROM employees
   WHERE salary > (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) FROM employees);
   ```

5. **Original Query:**
   ```sql
   SELECT product_id, product_name, unit_price
   FROM products
   WHERE category_id IN (SELECT category_id FROM categories WHERE category_name = 'Electronics');
   ```
   **Optimized Query:**
   ```sql
   SELECT p.product_id, p.product_name, p.unit_price
   FROM products p
   JOIN categories c ON p.category_id = c.category_id
   WHERE c.category_name = 'Electronics';
   ```

6. **Original Query:**
   ```sql
   SELECT employee_id, first_name, last_name, department_id
   FROM employees
   WHERE department_id = (SELECT department_id FROM departments WHERE department_name = 'HR');
   ```
   **Optimized Query:**
   ```sql
   SELECT employee_id, first_name, last_name, department_id
   FROM employees
   WHERE department_id = (SELECT department_id FROM departments WHERE department_name = 'HR' LIMIT 1);
   ```

7. **Original Query:**
   ```sql
   SELECT order_id, order_date, total_amount
   FROM orders
   WHERE order_date >= (SELECT MIN(order_date) FROM orders);
   ```
   **Optimized Query:**
   ```sql
   SELECT order_id, order_date, total_amount
   FROM orders
   WHERE order_date >= (SELECT order_date FROM orders ORDER BY order_date ASC LIMIT 1);
   ```

8. **Original Query:**
   ```sql
   SELECT *
   FROM products
   WHERE product_id NOT IN (SELECT product_id FROM order_details);
   ```
   **Optimized Query:**
   ```sql
   SELECT p.*
   FROM products p
   LEFT JOIN order_details od ON p.product_id = od.product_id
   WHERE od.product_id IS NULL;
   ```

9. **Original Query:**
   ```sql
   SELECT employee_id, hire_date
   FROM employees
   WHERE hire_date = (SELECT MIN(hire_date) FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT employee_id, hire_date
   FROM employees
   ORDER BY hire_date ASC
   LIMIT 1;
   ```

10. **Original Query:**
   ```sql
   SELECT customer_id, COUNT(*) AS order_count
   FROM orders
   GROUP BY customer_id
   HAVING COUNT(*) > (SELECT AVG(order_count) FROM (SELECT COUNT(*) AS order_count FROM orders GROUP BY customer_id) AS avg_orders);
   ```
   **Optimized Query:**
   ```sql
   SELECT customer_id, COUNT(*) AS order_count
   FROM orders
   GROUP BY customer_id
   HAVING COUNT(*) > (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY order_count) FROM (SELECT COUNT(*) AS order_count FROM orders GROUP BY customer_id) AS avg_orders);
   ```

11. **Original Query:**
   ```sql
   SELECT product_id, product_name
   FROM products
   WHERE product_id IN (SELECT product_id FROM order_details WHERE quantity >= 10);
   ```
   **Optimized Query:**
   ```sql
   SELECT p.product_id, p.product_name
   FROM products p
   JOIN order_details od ON p.product_id = od.product_id
   WHERE od.quantity >= 10;
   ```

12. **Original Query:**
   ```sql
   SELECT department_id, COUNT(*) AS employee_count
   FROM employees
   GROUP BY department_id
   H

AVING COUNT(*) > (SELECT COUNT(*) / 10 FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT department_id, COUNT(*) AS employee_count
   FROM employees
   GROUP BY department_id
   HAVING COUNT(*) > (SELECT COUNT(*) * 0.1 FROM employees);
   ```

13. **Original Query:**
   ```sql
   SELECT order_id, customer_id, order_date
   FROM orders
   WHERE order_date = (SELECT MAX(order_date) FROM orders);
   ```
   **Optimized Query:**
   ```sql
   SELECT order_id, customer_id, order_date
   FROM orders
   ORDER BY order_date DESC
   LIMIT 1;
   ```

14. **Original Query:**
   ```sql
   SELECT employee_id, salary
   FROM employees
   WHERE salary = (SELECT MAX(salary) FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT employee_id, salary
   FROM employees
   ORDER BY salary DESC
   LIMIT 1;
   ```

15. **Original Query:**
   ```sql
   SELECT product_id, product_name
   FROM products
   WHERE product_id IN (SELECT product_id FROM order_details WHERE order_id = 1001);
   ```
   **Optimized Query:**
   ```sql
   SELECT p.product_id, p.product_name
   FROM products p
   JOIN order_details od ON p.product_id = od.product_id
   WHERE od.order_id = 1001;
   ```

16. **Original Query:**
   ```sql
   SELECT customer_id, COUNT(*) AS order_count
   FROM orders
   GROUP BY customer_id
   HAVING COUNT(*) = (SELECT MAX(order_count) FROM (SELECT COUNT(*) AS order_count FROM orders GROUP BY customer_id) AS max_orders);
   ```
   **Optimized Query:**
   ```sql
   SELECT customer_id, COUNT(*) AS order_count
   FROM orders
   GROUP BY customer_id
   HAVING COUNT(*) = (SELECT MAX(order_count) FROM (SELECT COUNT(*) AS order_count FROM orders GROUP BY customer_id) AS max_orders);
   ```

17. **Original Query:**
   ```sql
   SELECT employee_id, hire_date
   FROM employees
   WHERE hire_date = (SELECT hire_date FROM employees ORDER BY hire_date LIMIT 1);
   ```
   **Optimized Query:**
   ```sql
   SELECT employee_id, hire_date
   FROM employees
   ORDER BY hire_date ASC
   LIMIT 1;
   ```

18. **Original Query:**
   ```sql
   SELECT department_id, COUNT(*) AS employee_count
   FROM employees
   GROUP BY department_id
   HAVING COUNT(*) = (SELECT MIN(employee_count) FROM (SELECT COUNT(*) AS employee_count FROM employees GROUP BY department_id) AS min_employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT department_id, COUNT(*) AS employee_count
   FROM employees
   GROUP BY department_id
   HAVING COUNT(*) = (SELECT MIN(employee_count) FROM (SELECT COUNT(*) AS employee_count FROM employees GROUP BY department_id) AS min_employees);
   ```

19. **Original Query:**
   ```sql
   SELECT product_id, product_name
   FROM products
   WHERE product_id NOT IN (SELECT product_id FROM order_details);
   ```
   **Optimized Query:**
   ```sql
   SELECT p.product_id, p.product_name
   FROM products p
   LEFT JOIN order_details od ON p.product_id = od.product_id
   WHERE od.product_id IS NULL;
   ```

20. **Original Query:**
   ```sql
   SELECT department_id, AVG(salary) AS avg_salary
   FROM employees
   GROUP BY department_id
   HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);
   ```
   **Optimized Query:**
   ```sql
   SELECT department_id, AVG(salary) AS avg_salary
   FROM employees
   GROUP BY department_id
   HAVING AVG(salary) > (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) FROM employees);
   ```

SQL Logic improvement!
