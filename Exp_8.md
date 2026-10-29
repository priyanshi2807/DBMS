# Experiment 8 – Banking Transactions, Savepoints and Isolation Levels

## Aim

To implement banking-style database transactions using `START TRANSACTION` / `BEGIN`, `COMMIT`, `ROLLBACK` and `SAVEPOINT`; simulate partial failures; and demonstrate transaction isolation levels and concurrency anomalies using two database sessions.

## Software / Tools Required

- **MySQL 8.0 or later**
- MySQL Workbench, phpMyAdmin, or MySQL Command Line Client
- Two query tabs / connections (**Session A** and **Session B**) to the same MySQL database

> **Note:** Run the concurrency examples in two separate sessions connected to the same database. These examples use InnoDB tables. Execute the statements in the order shown and do not run both sessions' scripts as one uninterrupted script.

---

## 1. Theory

### 1.1 Transaction

A **transaction** is a group of database operations treated as one logical unit. A banking transfer should debit one account and credit another account together. Either both updates succeed, or neither should be permanently saved.

Example: transfer ₹500 from account 101 to account 102.

### 1.2 ACID Properties

| Property | Meaning | Banking example |
|---|---|---|
| **Atomicity** | All operations happen, or none happen | A transfer must not debit an account without crediting the receiver |
| **Consistency** | A transaction preserves database rules | Account balances must remain valid |
| **Isolation** | Concurrent transactions should not interfere incorrectly | Two withdrawals must not both spend the same available balance |
| **Durability** | Committed changes persist | A successful transfer remains saved after the transaction completes |

### 1.3 Transaction Control Commands

| Command | Purpose |
|---|---|
| `START TRANSACTION` / `BEGIN` | Starts a transaction |
| `COMMIT` | Saves all changes made in the current transaction |
| `ROLLBACK` | Undoes all uncommitted changes in the current transaction |
| `SAVEPOINT name` | Creates a point to which a transaction can partially roll back |
| `ROLLBACK TO SAVEPOINT name` | Undoes changes made after that savepoint without cancelling earlier changes |
| `SET TRANSACTION ISOLATION LEVEL ...` | Chooses the isolation level for a transaction |

### 1.4 Isolation Levels

| Isolation level | General behaviour |
|---|---|
| **READ UNCOMMITTED** | May allow dirty reads (seeing uncommitted changes) |
| **READ COMMITTED** | Reads only committed data; repeated reads may return different values |
| **REPEATABLE READ** | Consistent repeated reads for a transaction; this is MySQL InnoDB's default |
| **SERIALIZABLE** | Strongest standard isolation; conflicting operations may block or fail rather than behave as if interleaved |

The exact behaviour also depends on the database engine. The examples below are intended for **MySQL with InnoDB**.

### 1.5 Concurrency Anomalies

| Anomaly | Meaning | Example |
|---|---|---|
| **Dirty read** | One transaction reads changes that another transaction has not committed | Session B sees a temporary balance update from Session A, which later rolls back |
| **Non-repeatable read** | A transaction reads the same row twice and gets different committed values | An account balance changes between two reads in Session A |
| **Lost update** | One transaction overwrites another transaction's update | Two sessions read the same balance, independently calculate a new value and the later write overwrites the earlier change |
| **Phantom read** | Repeating a range query returns a different set of rows | A new account matching a balance filter is committed by another session between reads |

---

## 2. Procedure

1. Create a database and an InnoDB `Accounts` table.
2. Insert sample account records and verify the starting balances.
3. Perform a transfer using `START TRANSACTION` and `COMMIT`.
4. Attempt a transfer and use `ROLLBACK` to cancel it.
5. Use `SAVEPOINT` and `ROLLBACK TO SAVEPOINT` to undo only part of a transaction.
6. Simulate a partial failure by deliberately stopping after the debit and rolling back the transaction.
7. Open two independent sessions and follow the concurrency schedules below.
8. Compare results across isolation levels and record the observed output.

---

## 3. Database and Sample Table

Run this setup once in either session. If you already have a table named `Accounts`, back it up before dropping or recreating it.

```sql
CREATE DATABASE IF NOT EXISTS BankingDB;
USE BankingDB;

DROP TABLE IF EXISTS Accounts;

CREATE TABLE Accounts (
    account_id INT PRIMARY KEY,
    account_holder VARCHAR(100) NOT NULL,
    balance DECIMAL(12, 2) NOT NULL,
    CONSTRAINT chk_balance_nonnegative CHECK (balance >= 0)
) ENGINE = InnoDB;

INSERT INTO Accounts (account_id, account_holder, balance) VALUES
(101, 'Aarav Sharma', 5000.00),
(102, 'Diya Verma',   3000.00),
(103, 'Kabir Singh',  2000.00);

SELECT * FROM Accounts ORDER BY account_id;
```

### Initial Output

| account_id | account_holder | balance |
|---:|---|---:|
| 101 | Aarav Sharma | 5000.00 |
| 102 | Diya Verma | 3000.00 |
| 103 | Kabir Singh | 2000.00 |

---

## 4. Banking Transaction Using COMMIT

Transfer ₹500 from account 101 to account 102. Both updates are committed as one transaction.

```sql
USE BankingDB;

START TRANSACTION;

UPDATE Accounts
SET balance = balance - 500
WHERE account_id = 101 AND balance >= 500;

-- Confirm that exactly one source account row was updated.
SELECT ROW_COUNT() AS source_rows_updated;

UPDATE Accounts
SET balance = balance + 500
WHERE account_id = 102;

SELECT account_id, account_holder, balance
FROM Accounts
WHERE account_id IN (101, 102)
ORDER BY account_id;

COMMIT;

SELECT account_id, account_holder, balance
FROM Accounts
ORDER BY account_id;
```

**Expected result:** Account 101 becomes ₹4,500 and account 102 becomes ₹3,500. The changes are permanent after `COMMIT`.

> In application code, check that the debit and credit each affect exactly one row; if either step fails, issue `ROLLBACK` instead of `COMMIT`. The standalone SQL example shows the core transaction commands.

---

## 5. Transaction Using ROLLBACK

Try a transfer, inspect the temporary values in the same session, and cancel the transaction.

```sql
START TRANSACTION;

UPDATE Accounts
SET balance = balance - 200
WHERE account_id = 101;

UPDATE Accounts
SET balance = balance + 200
WHERE account_id = 103;

-- These values are visible to this session before rollback.
SELECT account_id, balance
FROM Accounts
WHERE account_id IN (101, 103)
ORDER BY account_id;

ROLLBACK;

-- Original committed values are restored.
SELECT account_id, balance
FROM Accounts
WHERE account_id IN (101, 103)
ORDER BY account_id;
```

**Expected result:** The ₹200 transfer is cancelled. Balances return to their values before this transaction started.

---

## 6. Partial Rollback Using SAVEPOINT

A savepoint lets the user undo only later work and retain earlier changes. This example adds ₹100 to account 103, creates a savepoint, then attempts another change and rolls back only to the savepoint.

```sql
START TRANSACTION;

UPDATE Accounts
SET balance = balance + 100
WHERE account_id = 103;

SAVEPOINT after_first_change;

UPDATE Accounts
SET balance = balance - 50
WHERE account_id = 102;

-- Undo the update to account 102 only.
ROLLBACK TO SAVEPOINT after_first_change;

-- Account 103's +100 remains pending; account 102's -50 is undone.
SELECT account_id, balance
FROM Accounts
WHERE account_id IN (102, 103)
ORDER BY account_id;

COMMIT;

SELECT account_id, balance
FROM Accounts
WHERE account_id IN (102, 103)
ORDER BY account_id;
```

**Expected result:** Account 103's ₹100 increase is committed, while account 102's ₹50 decrease is undone. A savepoint rollback does not end the transaction.

---

## 7. Simulating a Partial Failure

A banking transfer can fail after the debit but before the credit. Use `ROLLBACK` to ensure the debit is not retained.

```sql
START TRANSACTION;

UPDATE Accounts
SET balance = balance - 700
WHERE account_id = 101 AND balance >= 700;

SELECT account_id, balance
FROM Accounts
WHERE account_id = 101;

-- Simulated failure: the credit operation is intentionally not executed.
-- In a real program, error handling would issue ROLLBACK here.
ROLLBACK;

SELECT account_id, balance
FROM Accounts
WHERE account_id = 101;
```

**Expected result:** The attempted ₹700 debit is cancelled. The account retains its starting balance for this section, demonstrating atomicity.

---

## 8. Two-Session Concurrency Experiments

### Important Setup

- Open **Session A** and **Session B** in separate query tabs or separate client connections.
- Run `USE BankingDB;` in both sessions.
- Before **each** anomaly experiment, restore any modified test rows to the stated starting values and make sure no previous transaction is still open.
- Follow the actions in the exact order shown. A statement in one session may wait until the other session commits or rolls back; that is part of the demonstration.
- Do not use real banking data. These are test records only.

### 8.1 Dirty Read — READ UNCOMMITTED

**Objective:** Demonstrate that Session B may read an uncommitted value from Session A.

Restore account 101 to ₹5,000 before starting:

```sql
UPDATE Accounts SET balance = 5000.00 WHERE account_id = 101;
COMMIT;
```

**Session A — step A1:**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
UPDATE Accounts SET balance = 4000.00 WHERE account_id = 101;
-- Do not commit or roll back yet.
```

**Session B — step B1 (run while A's transaction remains open):**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM Accounts WHERE account_id = 101;
```

**Session B — step B2:** If the engine exposes the uncommitted value, it may show ₹4,000. Record the value, then end B's read transaction:

```sql
COMMIT;
```

**Session A — step A2:**

```sql
ROLLBACK;
SELECT balance FROM Accounts WHERE account_id = 101;
```

**Observation:** Account 101 returns to ₹5,000 in Session A. If Session B saw ₹4,000 before the rollback, it read a dirty value that was never committed. InnoDB can permit this read at `READ UNCOMMITTED`; behaviour should be verified on the installed engine/version.

### 8.2 Non-Repeatable Read — READ COMMITTED

Restore account 101 to ₹5,000 and commit before beginning.

**Session A — step A1:**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM Accounts WHERE account_id = 101;
-- Note the first value: 5000.00.
```

**Session B — step B1:**

```sql
START TRANSACTION;
UPDATE Accounts SET balance = 4500.00 WHERE account_id = 101;
COMMIT;
```

**Session A — step A2:**

```sql
SELECT balance FROM Accounts WHERE account_id = 101;
COMMIT;
```

**Observation:** In Session A, the first read is ₹5,000 and the second committed read can be ₹4,500. The same row produced different values within one transaction.

### 8.3 Repeatable Read — Compare with the Previous Experiment

Restore account 101 to ₹5,000 and commit before starting.

**Session A — step A1:**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM Accounts WHERE account_id = 101;
-- Note 5000.00.
```

**Session B — step B1:**

```sql
START TRANSACTION;
UPDATE Accounts SET balance = 4500.00 WHERE account_id = 101;
COMMIT;
```

**Session A — step A2:**

```sql
SELECT balance FROM Accounts WHERE account_id = 101;
COMMIT;
```

**Observation:** With InnoDB's consistent non-locking reads, Session A normally sees the original committed value for both reads while its repeatable-read transaction remains open. After Session A commits and starts a new transaction, it can see ₹4,500. Locking reads such as `SELECT ... FOR UPDATE` have different locking semantics.

### 8.4 Lost Update — Read, Calculate and Overwrite

This example deliberately uses application-style read–modify–write updates. Restore account 101 to ₹5,000 first.

**Session A — step A1:**

```sql
START TRANSACTION;
SELECT balance FROM Accounts WHERE account_id = 101;
-- Application reads 5000.00 and calculates 5000 - 500 = 4500.
```

**Session B — step B1:**

```sql
START TRANSACTION;
SELECT balance FROM Accounts WHERE account_id = 101;
-- Application also reads 5000.00 and calculates 5000 - 300 = 4700.
```

**Session A — step A2:**

```sql
UPDATE Accounts SET balance = 4500.00 WHERE account_id = 101;
COMMIT;
```

**Session B — step B2:**

```sql
UPDATE Accounts SET balance = 4700.00 WHERE account_id = 101;
COMMIT;
```

**Observation:** The final balance may be ₹4,700, so Session A's ₹500 withdrawal has effectively been overwritten by Session B's stale calculated value. This is an unsafe application pattern, not the correct way to perform a debit.

**Safer approach:** Perform the arithmetic in the database and validate the balance in a single update:

```sql
UPDATE Accounts
SET balance = balance - 300
WHERE account_id = 101 AND balance >= 300;
```

Check that exactly one row changed. For a multi-step operation, use an appropriate transaction and locking strategy, for example selecting the account with `SELECT ... FOR UPDATE` inside a transaction.

### 8.5 Phantom Read — REPEATABLE READ vs READ COMMITTED

This example uses a range query. Restore the test data before starting; other sample balances do not matter for the predicate shown.

**Session A — step A1:**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT account_id, balance
FROM Accounts
WHERE balance BETWEEN 1000 AND 6000
ORDER BY account_id;
```

**Session B — step B1:**

```sql
START TRANSACTION;
INSERT INTO Accounts (account_id, account_holder, balance)
VALUES (104, 'Meera Joshi', 2500.00);
COMMIT;
```

**Session A — step A2:**

```sql
SELECT account_id, balance
FROM Accounts
WHERE balance BETWEEN 1000 AND 6000
ORDER BY account_id;
COMMIT;
```

**Observation:** Under `READ COMMITTED`, the second query can include account 104, even though it was absent from the first result. The extra row is a phantom. Delete the test row after the experiment if it is not needed:

```sql
DELETE FROM Accounts WHERE account_id = 104;
COMMIT;
```

**Optional comparison:** Repeat with Session A using `REPEATABLE READ`. In InnoDB, consistent snapshot reads normally keep the same snapshot for repeated plain SELECT statements. Locking range reads and writes can use next-key or gap locks; their precise effects depend on the query and index.

---

## 9. Isolation Level Syntax Reference

Set a session's default isolation level before beginning its transaction:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

To check the current session and global values in MySQL 8.0:

```sql
SELECT @@SESSION.transaction_isolation,
       @@GLOBAL.transaction_isolation;
```

Use one isolation setting at a time for each experiment. Start the transaction after changing the session setting.

---

## 10. Result / Observation Table

Fill the observed results after running the queries in MySQL.

| Experiment | Expected observation |
|---|---|
| COMMIT | Both sides of the transfer persist |
| ROLLBACK | All uncommitted changes from the transaction are cancelled |
| SAVEPOINT | Only changes after the savepoint are undone |
| Partial failure | Debit is not retained when the transfer aborts |
| READ UNCOMMITTED | A dirty value may be visible before the writer rolls back |
| READ COMMITTED | Repeating a read may return a newer committed value |
| REPEATABLE READ | Repeated consistent reads normally see the same snapshot in InnoDB |
| Lost update | A stale application-calculated value can overwrite a later update |
| Phantom read | A repeated range query can return an additional committed row under READ COMMITTED |

---

## 11. Important Notes

- `ROLLBACK` reverses changes in the current uncommitted transaction; it cannot undo a transaction after it has been committed.
- `ROLLBACK TO SAVEPOINT` does not finish the transaction. Commit or roll back the remaining work afterward.
- MySQL DDL statements such as `CREATE TABLE` can cause implicit commits. Do not mix table creation into a transaction when testing rollback.
- Use InnoDB for the transactional behaviour described here. Do not assume every storage engine supports transactions.
- Reset modified test values before repeating the experiments, and close each transaction with `COMMIT` or `ROLLBACK`.
- Exact blocking and anomaly behaviour depends on isolation level, query type, indexes and engine implementation. Record what your own MySQL setup returns.

---

## Result

Banking-style transactions were implemented using `START TRANSACTION`, `COMMIT`, `ROLLBACK` and `SAVEPOINT`. A partial transfer failure was simulated to demonstrate atomicity. Two independent database sessions were used to study dirty reads, non-repeatable reads, lost updates and phantom reads, and to compare the behaviour of MySQL InnoDB isolation levels.

The experiment demonstrates how transaction control, isolation and safe update patterns help preserve the correctness of banking data.
