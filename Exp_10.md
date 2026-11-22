# Experiment 10 – Query Optimization Using Indexing and Query Rewriting

## Aim

To generate a dataset containing at least 10,000 rows, identify five slow SQL queries, inspect their execution plans using `EXPLAIN`, optimize them with suitable indexes and query rewriting, and compare execution time before and after optimization.

## Software / Tools Required

- MySQL 8.0 or later
- MySQL Workbench, phpMyAdmin, or MySQL Command Line Client
- InnoDB storage engine

> **Benchmark integrity:** Query durations depend on hardware, MySQL configuration and cache state. Run the statements on your system and enter your measured results; this experiment does not invent timings.

---

## 1. Theory

**Query optimization** selects an efficient way to execute an SQL statement. MySQL's optimizer considers indexes, estimated row counts, join order and operation costs.

An **index** helps MySQL locate rows without scanning every row. Indexes can improve filtering, joining and sorting, but consume storage and add overhead to inserts, updates and deletes.

**Query rewriting** expresses the same required result in a form that may execute more efficiently. Examples include avoiding functions on filtered columns, selecting only required columns, and replacing repeated correlated work with an equivalent grouped query.

`EXPLAIN` displays the execution plan selected by MySQL.

| Field | Meaning |
|---|---|
| `type` | Access method; `ALL` generally means a full table scan, while `ref` and `range` commonly indicate index access |
| `possible_keys` | Indexes MySQL considers usable |
| `key` | Index actually selected |
| `rows` | Estimated number of rows examined |
| `Extra` | Additional details, such as filtering or filesort |

`EXPLAIN ANALYZE` (MySQL 8.0.18+) executes a statement and reports actual iterator timing and row counts. Use it only with the read-only SELECT queries in this experiment.

## 2. Dataset Design

This experiment uses a synthetic e-commerce dataset containing **10,000 customers and 50,000 orders**. The data is generated deterministically using recursive CTEs.

## 3. Create Database and Tables

```sql
CREATE DATABASE IF NOT EXISTS QueryOptimizationDB;
USE QueryOptimizationDB;

DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS Customers;

CREATE TABLE Customers (
    customer_id INT NOT NULL,
    customer_name VARCHAR(100) NOT NULL,
    city VARCHAR(50) NOT NULL,
    email VARCHAR(120) NOT NULL,
    signup_date DATE NOT NULL,
    PRIMARY KEY (customer_id)
) ENGINE = InnoDB;

CREATE TABLE Orders (
    order_id INT NOT NULL,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    order_status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES Customers(customer_id)
) ENGINE = InnoDB;
```

## 4. Generate at Least 10,000 Rows

```sql
USE QueryOptimizationDB;
SET SESSION cte_max_recursion_depth = 60000;

INSERT INTO Customers (
    customer_id, customer_name, city, email, signup_date
)
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10000
)
SELECT
    n,
    CONCAT(
        ELT((n MOD 10) + 1,
            'Aarav', 'Diya', 'Kabir', 'Meera', 'Rohan',
            'Ananya', 'Ishaan', 'Sana', 'Vivaan', 'Aditi'),
        ' Customer'
    ),
    ELT((n MOD 10) + 1,
        'Delhi', 'Mumbai', 'Chandigarh', 'Kanpur', 'Jaipur',
        'Lucknow', 'Pune', 'Bengaluru', 'Hyderabad', 'Chennai'),
    CONCAT('customer', n, '@example.com'),
    DATE_ADD('2024-01-01', INTERVAL (n MOD 900) DAY)
FROM nums;

INSERT INTO Orders (
    order_id, customer_id, order_date,
    order_status, total_amount, payment_method
)
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 50000
)
SELECT
    n,
    1 + (n MOD 10000),
    DATE_ADD('2025-01-01', INTERVAL (n MOD 600) DAY),
    ELT((n MOD 4) + 1, 'Pending', 'Shipped', 'Delivered', 'Cancelled'),
    100.00 + (n MOD 200) * 25.00,
    ELT((n MOD 4) + 1, 'UPI', 'Card', 'NetBanking', 'Cash')
FROM nums;

SELECT COUNT(*) AS total_customers FROM Customers;
SELECT COUNT(*) AS total_orders FROM Orders;
```

**Expected counts:** 10,000 customers and 50,000 orders.

The primary keys and foreign key are present from the beginning. Secondary indexes used for optimization are created later. InnoDB creates an index to support the foreign key if a suitable index does not already exist.

## 5. Five Slow Queries – Before Optimization

Run each query and its `EXPLAIN` before creating the secondary indexes in Section 6. Record the duration shown by your SQL client.

### Query 1 – Filter Orders by Status

```sql
SELECT *
FROM Orders
WHERE order_status = 'Delivered';

EXPLAIN
SELECT *
FROM Orders
WHERE order_status = 'Delivered';
```

**Problem:** `order_status` initially has no dedicated secondary index, so MySQL will generally scan the Orders table.

### Query 2 – Filter by Date Using a Function

```sql
SELECT *
FROM Orders
WHERE YEAR(order_date) = 2025;

EXPLAIN
SELECT *
FROM Orders
WHERE YEAR(order_date) = 2025;
```

**Problem:** Applying `YEAR()` to the column makes a normal B-tree index on `order_date` difficult to use for a direct range lookup.

### Query 3 – Find Customers by City

```sql
SELECT *
FROM Customers
WHERE city = 'Delhi';

EXPLAIN
SELECT *
FROM Customers
WHERE city = 'Delhi';
```

**Problem:** `city` initially has no dedicated secondary index.

### Query 4 – Join and Filter Orders by Status and Date

```sql
SELECT
    o.order_id, c.customer_name, c.city,
    o.order_date, o.total_amount
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'Shipped'
  AND o.order_date >= '2025-06-01'
  AND o.order_date < '2025-07-01';

EXPLAIN
SELECT
    o.order_id, c.customer_name, c.city,
    o.order_date, o.total_amount
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'Shipped'
  AND o.order_date >= '2025-06-01'
  AND o.order_date < '2025-07-01';
```

**Problem:** Without a suitable composite index, the optimizer may examine many order rows before joining customers.

### Query 5 – Correlated Subquery for Above-Average Customer Orders

```sql
SELECT c.customer_id, c.customer_name
FROM Customers c
WHERE (
    SELECT AVG(o.total_amount)
    FROM Orders o
    WHERE o.customer_id = c.customer_id
) > 2500;

EXPLAIN
SELECT c.customer_id, c.customer_name
FROM Customers c
WHERE (
    SELECT AVG(o.total_amount)
    FROM Orders o
    WHERE o.customer_id = c.customer_id
) > 2500;
```

**Problem:** The correlated subquery can repeat aggregate work for each customer.

## 6. Create Indexes for Optimization

```sql
CREATE INDEX idx_orders_status ON Orders (order_status);
CREATE INDEX idx_orders_date ON Orders (order_date);
CREATE INDEX idx_customers_city ON Customers (city);
CREATE INDEX idx_orders_status_date ON Orders (order_status, order_date);
CREATE INDEX idx_orders_customer_amount ON Orders (customer_id, total_amount);

SHOW INDEX FROM Orders;
SHOW INDEX FROM Customers;
```

| Index | Purpose |
|---|---|
| `idx_orders_status` | Filtering by order status |
| `idx_orders_date` | Date range searches |
| `idx_customers_city` | Filtering customers by city |
| `idx_orders_status_date` | Combined status and date filtering |
| `idx_orders_customer_amount` | Customer grouping and reading order amounts |

The composite index `(order_status, order_date)` supports searches on `order_status` alone and on both columns, following the leftmost-prefix rule.

## 7. Optimize the Five Queries

### Optimized Query 1 – Status Filter

```sql
SELECT order_id, customer_id, order_date, total_amount
FROM Orders
WHERE order_status = 'Delivered';

EXPLAIN
SELECT order_id, customer_id, order_date, total_amount
FROM Orders
WHERE order_status = 'Delivered';
```

**Optimization:** Added `idx_orders_status` and selected only required columns instead of `SELECT *`.

### Optimized Query 2 – Rewrite the Date Condition

```sql
SELECT order_id, customer_id, order_date, total_amount
FROM Orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2026-01-01';

EXPLAIN
SELECT order_id, customer_id, order_date, total_amount
FROM Orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2026-01-01';
```

**Optimization:** Rewrote the condition as a sargable range predicate and added `idx_orders_date`. The half-open range also handles date-time values correctly if the column is later changed to DATETIME.

### Optimized Query 3 – City Filter

```sql
SELECT customer_id, customer_name, email
FROM Customers
WHERE city = 'Delhi';

EXPLAIN
SELECT customer_id, customer_name, email
FROM Customers
WHERE city = 'Delhi';
```

**Optimization:** Added `idx_customers_city` and selected only required columns.

### Optimized Query 4 – Composite Index

```sql
SELECT
    o.order_id, o.customer_id, o.order_date, o.total_amount,
    c.customer_name, c.city
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'Shipped'
  AND o.order_date >= '2025-06-01'
  AND o.order_date < '2025-07-01';

EXPLAIN
SELECT
    o.order_id, o.customer_id, o.order_date, o.total_amount,
    c.customer_name, c.city
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'Shipped'
  AND o.order_date >= '2025-06-01'
  AND o.order_date < '2025-07-01';
```

**Optimization:** Added `idx_orders_status_date` for equality on status followed by the date range. The Customers primary key supports matching `customer_id`.

### Optimized Query 5 – Rewrite the Correlated Subquery

```sql
SELECT c.customer_id, c.customer_name
FROM Customers c
JOIN (
    SELECT customer_id, AVG(total_amount) AS avg_amount
    FROM Orders
    GROUP BY customer_id
    HAVING AVG(total_amount) > 2500
) AS customer_totals
    ON customer_totals.customer_id = c.customer_id;

EXPLAIN
SELECT c.customer_id, c.customer_name
FROM Customers c
JOIN (
    SELECT customer_id, AVG(total_amount) AS avg_amount
    FROM Orders
    GROUP BY customer_id
    HAVING AVG(total_amount) > 2500
) AS customer_totals
    ON customer_totals.customer_id = c.customer_id;
```

**Optimization:** Replaced the correlated scalar subquery with a grouped derived table and added `idx_orders_customer_amount`. MySQL may still choose a scan or another plan based on its cost estimates.

Both forms return customers whose average order amount is greater than 2500. Customers with no orders are excluded by both forms.

## 8. Compare Execution Plans

After each optimization, compare its `EXPLAIN` output with the corresponding output from Section 5.

For detailed runtime information on MySQL 8.0.18 or later, run:

```sql
EXPLAIN ANALYZE
SELECT order_id, customer_id, order_date, total_amount
FROM Orders
WHERE order_status = 'Delivered';
```

Repeat `EXPLAIN ANALYZE` for each optimized SELECT query. It executes the query, so use it only with read-only SELECT statements.

Record access type, selected key, estimated rows and actual execution time before and after optimization. Do not assume an index must always be selected: a table scan can be the best plan for low-selectivity filters or queries returning many rows.

## 9. Execution Time Benchmark

Use MySQL Workbench's execution duration or another consistent SQL client timer.

1. Run each query once before measurement to warm up the cache.
2. Execute each query at least five times and record each duration.
3. Use the median duration as the before value.
4. Apply the indexes and rewrites.
5. Repeat the same warm-up and five-run procedure.
6. Compare medians using the same machine, dataset and client.

| Query | Before median (ms) | After median (ms) | Improvement (%) |
|---|---:|---:|---:|
| 1. Status filter | | | |
| 2. Year/date filter | | | |
| 3. City filter | | | |
| 4. Status + date join | | | |
| 5. Customer average | | | |

Calculate percentage improvement as:

```text
Improvement (%) = ((Before Time - After Time) / Before Time) * 100
```

A negative value means the measured query became slower. This can happen because of cache variation, optimizer choices or the cost of returning many rows.

## 10. EXPLAIN Plan Comparison

Fill in this table from your actual execution plans.

| Query | Before key | After key | Before access type | After access type | Estimated rows before | Estimated rows after |
|---|---|---|---|---|---:|---:|
| 1. Status filter | | | | | | |
| 2. Year/date filter | | | | | | |
| 3. City filter | | | | | | |
| 4. Status + date join | | | | | | |
| 5. Customer average | | | | | | |

## 11. Important Observations

1. Indexes can reduce the number of rows examined for selective conditions.
2. Applying a function to an indexed column can prevent a normal index range lookup; rewriting it as a range condition is often more efficient.
3. Composite index column order matters. `(order_status, order_date)` is not equivalent to `(order_date, order_status)`.
4. Verify that a rewritten query returns the same result as the original.
5. Indexes use extra disk space and can make data modification operations more expensive.
6. The optimizer chooses a plan based on estimated cost. A table scan can be best for low-selectivity filters or queries returning many rows.
7. `EXPLAIN` shows an estimated plan; `EXPLAIN ANALYZE` provides actual measurements on supported MySQL versions.

## Result

A synthetic dataset containing 10,000 customer records and 50,000 order records was generated. Five queries were examined using `EXPLAIN`, then optimized using single-column and composite indexes as well as query rewriting. Execution plans and measured query durations were compared before and after optimization. Enter actual benchmark values and plan details from the MySQL environment used to perform the experiment.
