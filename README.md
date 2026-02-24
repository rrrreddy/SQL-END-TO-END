# SQL-END-TO-END

### Sample database setup
We’ll use a simple e‑commerce schema so every question feels realistic.

```
sql
-- Create database (if your DB needs it)
CREATE DATABASE interview_sql;
USE interview_sql;
```
```
-- Customers
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    first_name    VARCHAR(50),
    last_name     VARCHAR(50),
    country       VARCHAR(50),
    signup_date   DATE
);

-- Products
CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(100),
    category      VARCHAR(50),
    price         DECIMAL(10,2)
);

-- Orders (header)
CREATE TABLE orders (
    order_id      INT PRIMARY KEY,
    customer_id   INT,
    order_date    DATE,
    status        VARCHAR(20),
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Order items (detail)
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id      INT,
    product_id    INT,
    quantity      INT,
    unit_price    DECIMAL(10,2),
    CONSTRAINT fk_items_order
        FOREIGN KEY (order_id) REFERENCES orders(order_id),
    CONSTRAINT fk_items_product
        FOREIGN KEY (product_id) REFERENCES products(product_id)
);
Sample data (small but enough to explore):

sql
INSERT INTO customers VALUES
(1,'Alice','Smith','USA','2024-01-10'),
(2,'Bob','Jones','USA','2024-02-05'),
(3,'Carlos','Diaz','Mexico','2024-02-20'),
(4,'Diana','Khan','India','2024-03-01');

INSERT INTO products VALUES
(10,'Laptop','Electronics',900.00),
(11,'Mouse','Electronics',25.00),
(12,'Desk','Furniture',150.00),
(13,'Chair','Furniture',80.00);

INSERT INTO orders VALUES
(100,1,'2024-03-10','shipped'),
(101,1,'2024-03-15','shipped'),
(102,2,'2024-03-20','cancelled'),
(103,3,'2024-03-22','shipped');

INSERT INTO order_items VALUES
(1000,100,10,1,900.00),
(1001,100,11,2,25.00),
(1002,101,12,1,150.00),
(1003,102,13,4,80.00),
(1004,103,11,1,25.00),
(1005,103,13,2,80.00);
```
This schema lets you practice joins, aggregations, window functions, CTEs, subqueries, constraints, skewed data handling (conceptual), partitioning and salting (data‑engineering‑side)—all topics named in your screenshots.
​

Core SQL interview patterns
Below I’ll give for each topic:

Concept

Typical interview questions

Example query with explanation

Selecting data (SELECT, WHERE)
Concept
SELECT chooses columns; FROM chooses table; WHERE filters rows before grouping/aggregation.
​

Common questions
“Select all USA customers who signed up in 2024.”

“What is the difference between WHERE and HAVING?”

Example
sql
-- Q1: USA customers signed up in 2024
SELECT customer_id, first_name, last_name
FROM customers
WHERE country = 'USA'
  AND signup_date >= '2024-01-01'
  AND signup_date <  '2025-01-01';
Key interview points:

Use half‑open date ranges to avoid off‑by‑one errors.

WHERE filters rows before any aggregation; HAVING filters groups after GROUP BY.

Sorting and limiting (ORDER BY, LIMIT / TOP)
Concept
ORDER BY sorts the final result; LIMIT (MySQL/Postgres) or TOP (SQL Server) restricts row count.
​

Questions
“Fetch the latest 3 orders.”

“Why is ORDER BY important with LIMIT?”

Example
sql
-- Last 3 orders by date
SELECT order_id, customer_id, order_date
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 3;
Key points:

Always specify ORDER BY when you use LIMIT in interviews; result order is undefined without it.

Add a tiebreaker column (order_id) for deterministic ordering.

Aggregations and GROUP BY
Concept
Aggregate functions: COUNT, SUM, AVG, MIN, MAX.
GROUP BY collects rows into groups; each non‑aggregated column in SELECT must appear in GROUP BY (standard SQL).
​

Questions
“Total revenue per customer.”

“How many orders per status?”

“Difference between COUNT(*) and COUNT(column)?”

Examples
sql
-- Revenue per customer (sum of quantity * unit_price)
SELECT
    o.customer_id,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id;
sql
-- Orders per status
SELECT status, COUNT(*) AS order_count
FROM orders
GROUP BY status;
Interview talking points:

COUNT(*) counts rows, including NULLs; COUNT(column) ignores rows where column is NULL.

Filters that should impact aggregation go in WHERE; filters on aggregated results go in HAVING:

sql
-- Customers with revenue > 500
SELECT
    o.customer_id,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id
HAVING SUM(oi.quantity * oi.unit_price) > 500;
Joins (inner, left, right, full, cross)
Your screenshots mention joins multiple times, so this section is critical.
​

Core concepts
INNER JOIN: keep rows that match in both tables.

LEFT JOIN: keep all rows from left; NULLs when there is no match on the right.

RIGHT JOIN: opposite of left.

FULL OUTER JOIN: keep rows from both sides, matched or not.

CROSS JOIN: Cartesian product (every combination).

Typical interview questions
“Show each order with the customer name.”

“Show customers who have never placed an order.”

“Difference between INNER JOIN and LEFT JOIN?”

“How to find orders with items that have no matching product?” (data quality check).

INNER JOIN example
sql
-- Orders with customer name
SELECT
    o.order_id,
    c.first_name,
    c.last_name,
    o.order_date,
    o.status
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id;
Explanation:

Only customers that have orders appear. If a row is in orders with a customer_id that doesn’t exist in customers, it is excluded.

LEFT JOIN example (customers without orders)
sql
-- Customers and their total order count (including 0)
SELECT
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name;
Points to mention:

LEFT JOIN preserves all customers.

Customers with no orders will have NULL in o.order_id; COUNT(o.order_id) would ignore NULLs, so COUNT(*) could be used if you need row count per customer including ones with no orders but you adjust logic accordingly.

RIGHT / FULL OUTER JOIN example (concept)
Some databases don’t support FULL OUTER JOIN directly (e.g., MySQL). You can emulate with UNION.

Interview answer:

sql
-- Full outer join using UNION (MySQL-style)
SELECT *
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id

UNION

SELECT *
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;
Explain:

First query gets all customers plus matching orders.

Second gets all orders plus customers that might not exist (data anomaly).

UNION removes duplicates (we want that where matches exist).

CROSS JOIN example
sql
-- All combinations of country and category (for possible reports)
SELECT DISTINCT c.country, p.category
FROM customers c
CROSS JOIN products p;
Explain:

Use rarely; can explode row count; mention that in interview to show awareness of data explosion / skew.

Multi-table joins and derived metrics
Question
“Find total revenue per customer per product category.”

Query
sql
SELECT
    c.customer_id,
    c.first_name,
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM customers c
JOIN orders o
    ON c.customer_id = o.customer_id
JOIN order_items oi
    ON o.order_id = oi.order_id
JOIN products p
    ON oi.product_id = p.product_id
GROUP BY c.customer_id, c.first_name, p.category;
Interview notes:

Show you can chain multiple joins.

Mention that join order logically doesn’t matter, but execution plan may reorder for performance.

For large data, you’d use partitioning, broadcast joins, or salting in systems like Databricks / Spark (related to your images).

Filtering on joined data
Question
“List all shipped orders for USA customers.”

sql
SELECT
    o.order_id,
    o.order_date,
    c.country,
    o.status
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
WHERE c.country = 'USA'
  AND o.status = 'shipped';
Explain:

Filters in WHERE apply after the join but before grouping.

If using LEFT JOIN and you filter only on columns from the right table in WHERE, you turn it effectively into an inner join—this is often an interview trap.

Example of the trap:

sql
-- BUGGY: this behaves like INNER JOIN
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'shipped';
Correct version (if you really want customers with zero shipped orders too):

sql
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
  AND o.status = 'shipped';
Basic subqueries (to connect later with CTEs)
Concept
A subquery is a query inside another query; used in WHERE, FROM, or SELECT.
​

Question
“Find customers whose total revenue is above the average customer revenue.”

sql
-- Subquery in WHERE
SELECT customer_id, total_revenue
FROM (
    SELECT
        o.customer_id,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.customer_id
) t
WHERE total_revenue >
      (SELECT AVG(total_revenue) FROM (
           SELECT
               o.customer_id,
               SUM(oi.quantity * oi.unit_price) AS total_revenue
           FROM orders o
           JOIN order_items oi ON o.order_id = oi.order_id
           GROUP BY o.customer_id
      ) x);
