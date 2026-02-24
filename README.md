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
```SELECT chooses columns; FROM chooses table; WHERE filters rows before grouping/aggregation.```
​

Common questions
“Select all USA customers who signed up in 2024.”

“What is the difference between WHERE and HAVING?”

Example
sql
```-- Q1: USA customers signed up in 2024
SELECT customer_id, first_name, last_name
FROM customers
WHERE country = 'USA'
  AND signup_date >= '2024-01-01'
  AND signup_date <  '2025-01-01';```
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
```-- Last 3 orders by date
SELECT order_id, customer_id, order_date
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 3;```
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
```SELECT
    o.customer_id,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id;
sql
-- Orders per status
SELECT status, COUNT(*) AS order_count
FROM orders
GROUP BY status;```
Interview talking points:

COUNT(*) counts rows, including NULLs; COUNT(column) ignores rows where column is NULL.

Filters that should impact aggregation go in WHERE; filters on aggregated results go in HAVING:

sql
-- Customers with revenue > 500
```SELECT
    o.customer_id,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id
HAVING SUM(oi.quantity * oi.unit_price) > 500;```
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
```sql
-- Orders with customer name
SELECT
    o.order_id,
    c.first_name,
    c.last_name,
    o.order_date,
    o.status
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id;```
Explanation:

Only customers that have orders appear. If a row is in orders with a customer_id that doesn’t exist in customers, it is excluded.

LEFT JOIN example (customers without orders)
```sql
-- Customers and their total order count (including 0)
SELECT
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name;```
Points to mention:

LEFT JOIN preserves all customers.

Customers with no orders will have NULL in o.order_id; COUNT(o.order_id) would ignore NULLs, so COUNT(*) could be used if you need row count per customer including ones with no orders but you adjust logic accordingly.

RIGHT / FULL OUTER JOIN example (concept)
Some databases don’t support FULL OUTER JOIN directly (e.g., MySQL). You can emulate with UNION.

Interview answer:

```sql
-- Full outer join using UNION (MySQL-style)
SELECT *
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id

UNION

SELECT *
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;```
Explain:

First query gets all customers plus matching orders.

Second gets all orders plus customers that might not exist (data anomaly).

UNION removes duplicates (we want that where matches exist).

CROSS JOIN example
```sql
-- All combinations of country and category (for possible reports)
SELECT DISTINCT c.country, p.category
FROM customers c
CROSS JOIN products p;```
Explain:

Use rarely; can explode row count; mention that in interview to show awareness of data explosion / skew.

Multi-table joins and derived metrics
Question
“Find total revenue per customer per product category.”

Query
```sql
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
GROUP BY c.customer_id, c.first_name, p.category;```
Interview notes:

Show you can chain multiple joins.

Mention that join order logically doesn’t matter, but execution plan may reorder for performance.

For large data, you’d use partitioning, broadcast joins, or salting in systems like Databricks / Spark (related to your images).

Filtering on joined data
Question
“List all shipped orders for USA customers.”

```sql
SELECT
    o.order_id,
    o.order_date,
    c.country,
    o.status
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
WHERE c.country = 'USA'
  AND o.status = 'shipped';```
Explain:

Filters in WHERE apply after the join but before grouping.

If using LEFT JOIN and you filter only on columns from the right table in WHERE, you turn it effectively into an inner join—this is often an interview trap.

Example of the trap:

```sql
-- BUGGY: this behaves like INNER JOIN
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'shipped';```
Correct version (if you really want customers with zero shipped orders too):

sql
```SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
  AND o.status = 'shipped';```
Basic subqueries (to connect later with CTEs)
Concept
A subquery is a query inside another query; used in WHERE, FROM, or SELECT.
​

Question
“Find customers whose total revenue is above the average customer revenue.”

sql
```-- Subquery in WHERE
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
      ) x);```



### Window functions basics
Window functions perform calculations across a set of related rows but keep each row.

Syntax pattern:
```
function_name(...) OVER (
    PARTITION BY <col(s)>   -- optional: define groups
    ORDER BY <col(s)>       -- optional: define order inside each group
    -- optional frame clause
)
```
**Ranking functions: ROW_NUMBER, RANK, DENSE_RANK**
Concepts
ROW_NUMBER(): unique sequence, no ties; 1,2,3,….

RANK(): same value ⇒ same rank; gaps after ties (1,1,3,…).

DENSE_RANK(): same value ⇒ same rank; no gaps (1,1,2,…).

Interview question 1
“Get each customer’s orders, with newest first, and assign a row number per customer.”
```
SELECT
    o.customer_id,
    o.order_id,
    o.order_date,
    ROW_NUMBER() OVER (
        PARTITION BY o.customer_id
        ORDER BY o.order_date DESC, o.order_id DESC
    ) AS order_seq
FROM orders o;
```
Use cases to mention:

Pick latest order per customer by wrapping in a subquery/CTE and filtering order_seq = 1.
```
WITH ordered AS (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY o.customer_id
            ORDER BY o.order_date DESC, o.order_id DESC
        ) AS rn
    FROM orders o
)
SELECT *
FROM ordered
WHERE rn = 1;
```
This pattern (ROW_NUMBER in CTE then filter) is extremely common in interviews.

Interview question 2
“Top 2 products per category by revenue, and explain the difference between RANK and DENSE_RANK.”

```
WITH product_revenue AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.product_name, p.category
)
SELECT
    category,
    product_name,
    revenue,
    RANK() OVER (
        PARTITION BY category
        ORDER BY revenue DESC
    ) AS rnk,
    DENSE_RANK() OVER (
        PARTITION BY category
        ORDER BY revenue DESC
    ) AS dense_rnk
FROM product_revenue
WHERE RANK() OVER (
        PARTITION BY category
        ORDER BY revenue DESC
     ) <= 2;
```
Explain for the interviewer:

If two products tie for highest revenue in a category, both get rank 1.

RANK then jumps to 3 for the next product; DENSE_RANK uses 2.

**LEAD and LAG**
Concepts
LAG(col, offset) looks at previous row in the window.

LEAD(col, offset) looks at next row.

“Show each customer’s orders and the gap in days since their previous order.”
```
SELECT
    o.customer_id,
    o.order_id,
    o.order_date,
    LAG(o.order_date) OVER (
        PARTITION BY o.customer_id
        ORDER BY o.order_date
    ) AS prev_order_date,
    DATEDIFF(
        DAY,
        LAG(o.order_date) OVER (
            PARTITION BY o.customer_id
            ORDER BY o.order_date
        ),
        o.order_date
    ) AS days_since_prev
FROM orders o
ORDER BY o.customer_id, o.order_date;
```
Key talking points:

LAG avoids self‑joins for “previous row” questions (stock prices, page views, customer events).

First row per partition has NULL previous value; you can provide a default: LAG(col,1,0).


**Aggregation as window function (running totals, partitioning)**
Running total per customer
“Compute running total revenue per customer, by date.”
```
SELECT
    o.customer_id,
    o.order_date,
    SUM(oi.quantity * oi.unit_price) AS daily_revenue,
    SUM(oi.quantity * oi.unit_price) OVER (
        PARTITION BY o.customer_id
        ORDER BY o.order_date
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS running_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.customer_id, o.order_date
ORDER BY o.customer_id, o.order_date;
```
Explain:

The GROUP BY aggregates to daily revenue; window SUM then builds a cumulative total per customer.​

Window aggregation keeps one row per date (not collapsed like normal GROUP BY).

### CTEs (Common Table Expressions)
Concept
CTE = named temporary result defined with WITH, used only for one statement.

Syntax:
```
WITH cte_name AS (
    -- some SELECT
)
SELECT ...
FROM cte_name;
```
WITH cte_name AS (
    -- some SELECT
)
SELECT ...
FROM cte_name;
```
Why interviews like CTEs
Improve readability of multi‑step logic.

Reusable inside the same query.

Required for recursive queries in many databases.
```
WITH customer_revenue AS (
    SELECT
        o.customer_id,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.customer_id
)
SELECT *
FROM customer_revenue
WHERE total_revenue > 500;
```
Explain:

CTE is just syntactic sugar; logically similar to derived table.

Advantage is clarity when you later add window functions or additional filters.

**Interview question 2 – multiple CTEs**
“Find, for each customer, their latest order and the number of days since signup.”

```
WITH latest_order AS (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY o.customer_id
            ORDER BY o.order_date DESC, o.order_id DESC
        ) AS rn
    FROM orders o
),
max_per_customer AS (
    SELECT *
    FROM latest_order
    WHERE rn = 1
)
SELECT
    c.customer_id,
    c.first_name,
    max_per_customer.order_id,
    max_per_customer.order_date,
    DATEDIFF(DAY, c.signup_date, max_per_customer.order_date) AS days_since_signup
FROM customers c
JOIN max_per_customer
  ON c.customer_id = max_per_customer.customer_id;
```
Talking points:

Multiple CTEs are evaluated top‑down; later CTEs can use previous ones.
​

Always mention complexity impact: some engines materialize CTEs; others inline them.

Anti‑joins: finding missing data
Pattern 1: LEFT JOIN ... WHERE right IS NULL
“Find order_items whose product_id does not exist in products (data quality issue).”

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```
Explain:

NOT EXISTS usually handles NULLs safely and can be more efficient than NOT IN in many engines.
​
### UNION vs UNION ALL
Frequently asked with joins.
​

UNION ALL: concatenates results, keeps duplicates, faster (no dedup sort).

UNION: deduplicates rows, may need sort/hash, slower.
```
SELECT customer_id FROM customers WHERE country = 'USA'
UNION ALL
SELECT customer_id FROM customers WHERE country = 'Mexico';
```
Mention: choose UNION only when you need uniqueness.

Constraints and keys
Primary key and foreign key
From our schema:
```
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    ...
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```
Interview points:

Primary key: unique + NOT NULL, identifies a row.

Foreign key: enforces referential integrity; order must link to existing customer.

You can have one primary key per table but multiple foreign keys.

Other constraints
UNIQUE – all values distinct.

CHECK – custom boolean expression (e.g., quantity > 0).

NOT NULL – disallow NULL.
```
ALTER TABLE order_items
ADD CONSTRAINT chk_quantity_positive
CHECK (quantity > 0);
```
### Normalization vs denormalization (high‑level)
Interviewers often test if you understand data modeling more than exact forms.

Normalization: structuring tables to reduce redundancy, avoid anomalies (1NF, 2NF, 3NF…).

Denormalization: intentionally duplicating data for read performance (e.g., adding customer_country into orders for reporting).

```
ALTER TABLE orders ADD customer_country VARCHAR(50);

UPDATE orders o
JOIN customers c ON o.customer_id = c.customer_id
SET o.customer_country = c.country;
```
“For OLTP systems I prefer normalized models; for analytics (star schema on Snowflake or Databricks) I may denormalize to simplify reporting and improve performance.”

### Indexing basics (very common interview topic)
What is an index?
An index is a separate data structure that lets the DB find rows faster, similar to a book index.
Key concepts to say:

Clustered index: defines physical order of rows; usually on primary key (order_id).

Non‑clustered index: separate structure pointing to rows; many per table allowed.
​
Example interview‑style answer:

```
-- Non-clustered index to speed up lookups by customer_id on orders
CREATE INDEX ix_orders_customer_id
ON orders(customer_id);
```
Talking points:

Helps queries like WHERE customer_id = ? or joins on customer_id.

Too many indexes slow down INSERT/UPDATE/DELETE because each index must be updated.

Putting it together – interview‑style question
Question: “For each customer, show their latest order, its revenue, their total lifetime revenue, their rank by total revenue among all customers, and number of days since previous order.”
```
WITH order_revenue AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_date,
        SUM(oi.quantity * oi.unit_price) AS order_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, o.customer_id, o.order_date
),
customer_totals AS (
    SELECT
        customer_id,
        SUM(order_revenue) AS total_revenue
    FROM order_revenue
    GROUP BY customer_id
),
latest_orders AS (
    SELECT
        orv.*,
        ROW_NUMBER() OVER (
            PARTITION BY orv.customer_id
            ORDER BY orv.order_date DESC, orv.order_id DESC
        ) AS rn,
        LAG(orv.order_date) OVER (
            PARTITION BY orv.customer_id
            ORDER BY orv.order_date
        ) AS prev_order_date
    FROM order_revenue orv
)
SELECT
    c.customer_id,
    c.first_name,
    lo.order_id,
    lo.order_date,
    lo.order_revenue,
    ct.total_revenue,
    RANK() OVER (ORDER BY ct.total_revenue DESC) AS revenue_rank,
    DATEDIFF(DAY, lo.prev_order_date, lo.order_date) AS days_since_prev_order
FROM latest_orders lo
JOIN customers c ON c.customer_id = lo.customer_id
JOIN customer_totals ct ON ct.customer_id = c.customer_id
WHERE lo.rn = 1
ORDER BY revenue_rank;
```
This single query showcases:

CTEs

Aggregations

Joins

Window functions (ROW_NUMBER, LAG, RANK)

Business‑style logic (lifetime vs latest metrics)

Exactly the style seen in advanced SQL interview guides.
