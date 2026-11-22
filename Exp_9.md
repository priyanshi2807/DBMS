# Experiment 9 – Railway Booking Indexing and Query Performance Benchmark

## Aim

To create a Railway Booking table with at least 5,000 records, benchmark query performance before and after creating single-column and composite indexes, and compare query execution plans using `EXPLAIN`.

## Software / Tools Required

- MySQL 8.0 or later
- MySQL Workbench, phpMyAdmin, or MySQL Command Line Client
- InnoDB storage engine

> **Note:** This experiment generates 5,000 sample bookings. Query timings depend on your computer, MySQL version and cache state. Record actual timings from your own installation; do not use invented benchmark values.

---

## 1. Theory

### 1.1 Indexing

An **index** is a data structure that helps MySQL find rows without scanning the complete table. It can improve searching, filtering, sorting and joining. Indexes use extra storage and add overhead to `INSERT`, `UPDATE` and `DELETE` operations.

### 1.2 Types of Indexes

| Index | Meaning | Example |
|---|---|---|
| Single-column | Index created on one column | `INDEX (train_no)` |
| Composite | Index created on multiple columns in a defined order | `INDEX (train_no, journey_date)` |
| Primary key | Unique index identifying each row | `booking_id` |

A composite index on `(train_no, journey_date)` can support searches on `train_no` and on both columns together. It is generally not as useful for a search on `journey_date` alone because of the **leftmost-prefix rule**.

### 1.3 EXPLAIN

`EXPLAIN` displays the execution plan selected by MySQL.

| Field | Meaning |
|---|---|
| `type` | Access method; `ALL` usually means a full table scan, while `range` and `ref` are index-based access methods |
| `possible_keys` | Indexes the optimizer considers usable |
| `key` | Index actually selected |
| `rows` | Estimated rows to examine |
| `Extra` | Additional details such as filtering or filesort |

An index is not always faster. MySQL may choose a table scan when a query returns a large portion of the table.

## 2. Procedure

1. Create the `RailwayDB` database and `Railway_Booking` table.
2. Insert 5,000 generated booking records and verify the count.
3. Run search queries and `EXPLAIN` before secondary indexes exist.
4. Record query duration and execution-plan details.
5. Create single-column indexes on `train_no` and `passenger_name`.
6. Repeat the queries and compare plans and timings.
7. Create a composite index on `(train_no, journey_date)`.
8. Repeat the train/date query and compare the results.

## 3. Create Database and Table

```sql
CREATE DATABASE IF NOT EXISTS RailwayDB;
USE RailwayDB;

DROP TABLE IF EXISTS Railway_Booking;

CREATE TABLE Railway_Booking (
    booking_id INT NOT NULL,
    passenger_name VARCHAR(100) NOT NULL,
    train_no INT NOT NULL,
    train_name VARCHAR(100) NOT NULL,
    source_station VARCHAR(60) NOT NULL,
    destination_station VARCHAR(60) NOT NULL,
    journey_date DATE NOT NULL,
    seat_no VARCHAR(10) NOT NULL,
    coach VARCHAR(10) NOT NULL,
    class_type VARCHAR(20) NOT NULL,
    booking_status VARCHAR(20) NOT NULL,
    fare DECIMAL(10, 2) NOT NULL,
    booking_date DATE NOT NULL,
    PRIMARY KEY (booking_id)
) ENGINE = InnoDB;
```

## 4. Insert 5,000 Records

The recursive CTE generates booking IDs from 1 to 5,000. Values are deterministic so the dataset can be recreated.

```sql
USE RailwayDB;
SET SESSION cte_max_recursion_depth = 6000;

INSERT INTO Railway_Booking (
    booking_id, passenger_name, train_no, train_name,
    source_station, destination_station, journey_date,
    seat_no, coach, class_type, booking_status, fare, booking_date
)
WITH RECURSIVE booking_numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM booking_numbers WHERE n < 5000
)
SELECT
    n,
    CONCAT(
        ELT((n MOD 10) + 1,
            'Aarav', 'Diya', 'Kabir', 'Meera', 'Rohan',
            'Ananya', 'Ishaan', 'Sana', 'Vivaan', 'Aditi'),
        ' ',
        ELT(((n DIV 10) MOD 10) + 1,
            'Sharma', 'Verma', 'Singh', 'Joshi', 'Gupta',
            'Khan', 'Mehta', 'Reddy', 'Kapoor', 'Mishra')
    ),
    12000 + (n MOD 20),
    ELT((n MOD 10) + 1,
        'Rajdhani Express', 'Shatabdi Express', 'Duronto Express',
        'Vande Bharat', 'Garib Rath', 'Sampark Kranti',
        'Humsafar Express', 'Jan Shatabdi', 'Mail Express',
        'Superfast Express'),
    ELT((n MOD 8) + 1,
        'New Delhi', 'Mumbai Central', 'Chennai Central', 'Howrah',
        'Kanpur Central', 'Lucknow', 'Bhopal', 'Jaipur'),
    ELT(((n + 3) MOD 8) + 1,
        'New Delhi', 'Mumbai Central', 'Chennai Central', 'Howrah',
        'Kanpur Central', 'Lucknow', 'Bhopal', 'Jaipur'),
    DATE_ADD('2026-12-01', INTERVAL (n MOD 90) DAY),
    CAST(1 + (n MOD 72) AS CHAR),
    CONCAT('S', 1 + (n MOD 10)),
    ELT((n MOD 4) + 1, 'Sleeper', 'AC 3 Tier', 'AC 2 Tier', 'Chair Car'),
    ELT((n MOD 4) + 1, 'Confirmed', 'Waiting', 'RAC', 'Cancelled'),
    250.00 + (n MOD 20) * 75.00,
    DATE_ADD('2026-10-01', INTERVAL (n MOD 60) DAY)
FROM booking_numbers;

SELECT COUNT(*) AS total_bookings FROM Railway_Booking;
SELECT * FROM Railway_Booking ORDER BY booking_id LIMIT 10;
```

**Expected count:** 5,000 records, with booking IDs from 1 through 5,000.

## 5. Benchmark Before Creating Secondary Indexes

At this stage, only the primary key index exists. Run each query and note its duration using the execution-time display in MySQL Workbench or your SQL client.

### Query 1 – Search by Train Number

```sql
SELECT *
FROM Railway_Booking
WHERE train_no = 12005;

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE train_no = 12005;
```

### Query 2 – Search by Passenger Name

```sql
SELECT *
FROM Railway_Booking
WHERE passenger_name = 'Aarav Sharma';

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE passenger_name = 'Aarav Sharma';
```

### Query 3 – Search by Train Number and Journey Date

```sql
SELECT *
FROM Railway_Booking
WHERE train_no = 12005
  AND journey_date = '2026-12-06';

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE train_no = 12005
  AND journey_date = '2026-12-06';
```

**Expected observation:** Since these filter columns have no secondary indexes yet, MySQL will generally scan the table (`type = ALL`). Check the actual plan returned by your server.

## 6. Create Single-Column Indexes

```sql
CREATE INDEX idx_train_no
ON Railway_Booking (train_no);

CREATE INDEX idx_passenger_name
ON Railway_Booking (passenger_name);

SHOW INDEX FROM Railway_Booking;
```

### 6.1 Repeat Train Number Query

```sql
SELECT *
FROM Railway_Booking
WHERE train_no = 12005;

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE train_no = 12005;
```

**Expected observation:** MySQL can use `idx_train_no`. Confirm the selected index in the `key` column.

### 6.2 Repeat Passenger Name Query

```sql
SELECT *
FROM Railway_Booking
WHERE passenger_name = 'Aarav Sharma';

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE passenger_name = 'Aarav Sharma';
```

**Expected observation:** MySQL can use `idx_passenger_name` to find matching passenger names.

## 7. Create and Test a Composite Index

```sql
CREATE INDEX idx_train_date
ON Railway_Booking (train_no, journey_date);

SHOW INDEX FROM Railway_Booking;
```

### 7.1 Repeat Train Number and Journey Date Query

```sql
SELECT *
FROM Railway_Booking
WHERE train_no = 12005
  AND journey_date = '2026-12-06';

EXPLAIN
SELECT *
FROM Railway_Booking
WHERE train_no = 12005
  AND journey_date = '2026-12-06';
```

**Expected observation:** MySQL can use `idx_train_date` for both conditions. Check `key`, `type` and `rows`; the optimizer may choose another index if its cost estimate favours it.

### 7.2 Demonstrate the Leftmost-Prefix Rule

```sql
-- The leading column of the composite index is present.
EXPLAIN
SELECT * FROM Railway_Booking
WHERE train_no = 12005;

-- Only the second column is present.
EXPLAIN
SELECT * FROM Railway_Booking
WHERE journey_date = '2026-12-06';
```

**Observation:** The composite index is designed for searches beginning with `train_no`. It is generally not a direct lookup index for `journey_date` by itself.

## 8. Timing and Benchmark Method

Use the duration shown by your SQL client for each `SELECT`. To make the comparison fair:

1. Run each query once to warm up the cache; do not record that first run.
2. Run each query at least five more times and record the durations.
3. Compare the median duration, not just the fastest run.
4. Use the same query text and data before and after index creation.
5. Record the `EXPLAIN` output, especially `type`, `key` and `rows`.

In MySQL Workbench, use the execution duration shown for each query. Client-only timing commands should be entered in the MySQL command-line client, not inside a SQL script.

## 9. Result Comparison Table

Fill in actual measurements from your MySQL run. Times are intentionally left blank because they depend on the machine and database environment.

| Query | Index state | EXPLAIN key | Access type | Estimated rows | Median time |
|---|---|---|---|---:|---:|
| Train number | Before indexes | | | | |
| Train number | Single index | | | | |
| Passenger name | Before indexes | | | | |
| Passenger name | Single index | | | | |
| Train + journey date | Before indexes | | | | |
| Train + journey date | Composite index | | | | |

## 10. Important Notes

- The primary key index exists throughout the experiment; the comparison concerns secondary indexes.
- Composite index column order matters. `(train_no, journey_date)` is different from `(journey_date, train_no)`.
- Indexes can improve read queries but consume storage and can slow down writes.
- `EXPLAIN` reports optimizer estimates, not measured elapsed time. Use both the plan and actual timings for comparison.
- If the optimizer does not select an index, check data selectivity and the `rows` estimate. Do not force an index just to make the plan appear different.
- The sample data is synthetic and must not be treated as real railway passenger information.

## Result

A Railway Booking table containing 5,000 generated records was created. Query plans and execution durations were examined before and after single-column indexes on `train_no` and `passenger_name`, followed by a composite index on `(train_no, journey_date)`. The `EXPLAIN` command was used to compare index selection, access type and estimated rows. Actual timings should be entered from the MySQL environment used to perform the experiment.
