# 🧪 Experiment 3 – Employee–Department–Project Schema and SQL Queries

## Aim

To create an **Employee–Department–Project** schema in MySQL, insert at least **30 employees across 5 departments and 8 projects**, and write SQL queries demonstrating **selection, projection, aggregate functions, GROUP BY, HAVING, CASE expressions and ORDER BY**.

## Software Required

- MySQL Server 8.0 or later
- MySQL Workbench or the MySQL command-line client

## Files

| File | Description |
|---|---|
| [`company_queries.sql`](company_queries.sql) | Complete script: schema, sample data and all 36 queries |
| `README.md` | This lab record |

**How to run:**

```bash
mysql -u root -p -t < company_queries.sql
```

---

## 1. Theory

### SQL Clauses and Relational Algebra

| Concept | SQL | Relational Algebra | Purpose |
|---|---|---|---|
| Selection | `WHERE` | σ (sigma) | Choose rows that satisfy a condition |
| Projection | `SELECT col1, col2` / `DISTINCT` | π (pi) | Choose columns |
| Aggregation | `COUNT, SUM, AVG, MIN, MAX` | 𝒢 (gamma) | Summarise many rows into one value |
| Grouping | `GROUP BY` | 𝒢 with grouping attributes | Aggregate per group |
| Group filter | `HAVING` | σ applied after 𝒢 | Keep only groups that satisfy a condition |
| Conditional logic | `CASE … WHEN … THEN … END` | – | If-then-else inside a query |
| Sorting | `ORDER BY … ASC / DESC` | τ (tau) | Order the result |

### Logical Order of Execution

SQL is **written** in the order `SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY`, but it is **evaluated** in this order:

```
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

That is why:
- `WHERE` cannot use aggregate functions, because the groups don't exist yet. Use `HAVING` for that.
- `ORDER BY` can use a column alias defined in `SELECT`, because `SELECT` has already run.

### WHERE vs HAVING

| WHERE | HAVING |
|---|---|
| Filters individual **rows** | Filters **groups** |
| Applied **before** `GROUP BY` | Applied **after** `GROUP BY` |
| Cannot contain aggregate functions | Usually contains aggregate functions |

---

## 2. ER Diagram

```mermaid
erDiagram
    DEPARTMENT ||--|{ EMPLOYEE : "employs"
    DEPARTMENT |o--|| EMPLOYEE : "managed by"
    EMPLOYEE |o--o{ EMPLOYEE : "supervises"
    DEPARTMENT ||--o{ PROJECT : "controls"
    EMPLOYEE ||--o{ WORKS_ON : "assigned"
    PROJECT ||--|{ WORKS_ON : "has"

    DEPARTMENT {
        char dept_id PK
        varchar dept_name UK
        varchar location
        int manager_id FK
        year established
    }
    EMPLOYEE {
        int emp_id PK
        varchar first_name
        varchar last_name
        enum gender
        date dob
        date hire_date
        varchar email UK
        varchar city
        varchar designation
        decimal salary
        decimal commission "nullable"
        int manager_id FK "self-reference"
        char dept_id FK
    }
    PROJECT {
        char proj_id PK
        varchar proj_name UK
        char dept_id FK
        varchar location
        decimal budget
        date start_date
        date end_date "nullable"
        enum status
    }
    WORKS_ON {
        int emp_id PK, FK
        char proj_id PK, FK
        decimal hours_per_week
        varchar role
    }
```

## 3. Relational Schema

```
DEPARTMENT (dept_id, dept_name, location, manager_id → EMPLOYEE, established)
EMPLOYEE   (emp_id, first_name, last_name, gender, dob, hire_date, email, city, designation,
            salary, commission, manager_id → EMPLOYEE, dept_id → DEPARTMENT)
PROJECT    (proj_id, proj_name, dept_id → DEPARTMENT, location, budget, start_date, end_date, status)
WORKS_ON   (emp_id → EMPLOYEE, proj_id → PROJECT, hours_per_week, role)
```

| Table | Primary Key | Foreign Keys | Other Constraints |
|---|---|---|---|
| department | `dept_id` | `manager_id` → employee (SET NULL) | `dept_name` UNIQUE |
| employee | `emp_id` | `dept_id` → department (RESTRICT); `manager_id` → employee (SET NULL) | `email` UNIQUE, `salary > 0` |
| project | `proj_id` | `dept_id` → department (RESTRICT) | `proj_name` UNIQUE, `budget > 0` |
| works_on | (`emp_id`, `proj_id`) | `emp_id` → employee (CASCADE); `proj_id` → project (CASCADE) | `hours_per_week` between 1 and 48 |

> `department.manager_id` and `employee.dept_id` refer to each other. So `department` is created first, and its foreign key to `employee` is added afterwards with `ALTER TABLE`.

---

## 4. Implementation

### 4.1 Create Database and Tables

```sql
DROP DATABASE IF EXISTS company_db;
CREATE DATABASE company_db;
USE company_db;

CREATE TABLE department (
    dept_id       CHAR(3)      PRIMARY KEY,
    dept_name     VARCHAR(50)  NOT NULL UNIQUE,
    location      VARCHAR(50)  NOT NULL,
    manager_id    INT          NULL,               -- FK added after EMPLOYEE exists
    established   YEAR         NOT NULL
);

CREATE TABLE employee (
    emp_id        INT           PRIMARY KEY,
    first_name    VARCHAR(30)   NOT NULL,
    last_name     VARCHAR(30)   NOT NULL,
    gender        ENUM('M','F') NOT NULL,
    dob           DATE          NOT NULL,
    hire_date     DATE          NOT NULL,
    email         VARCHAR(100)  NOT NULL UNIQUE,
    city          VARCHAR(30)   NOT NULL,
    designation   VARCHAR(50)   NOT NULL,
    salary        DECIMAL(10,2) NOT NULL,          -- monthly salary in INR
    commission    DECIMAL(10,2) NULL,              -- only sales/marketing staff
    manager_id    INT           NULL,
    dept_id       CHAR(3)       NOT NULL,
    CONSTRAINT chk_salary CHECK (salary > 0),
    CONSTRAINT fk_emp_manager FOREIGN KEY (manager_id)
        REFERENCES employee(emp_id) ON DELETE SET NULL,
    CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id)
        REFERENCES department(dept_id) ON DELETE RESTRICT ON UPDATE CASCADE
);

ALTER TABLE department
    ADD CONSTRAINT fk_dept_manager FOREIGN KEY (manager_id)
        REFERENCES employee(emp_id) ON DELETE SET NULL;

CREATE TABLE project (
    proj_id       CHAR(3)       PRIMARY KEY,
    proj_name     VARCHAR(60)   NOT NULL UNIQUE,
    dept_id       CHAR(3)       NOT NULL,          -- controlling department
    location      VARCHAR(50)   NOT NULL,
    budget        DECIMAL(12,2) NOT NULL,          -- INR
    start_date    DATE          NOT NULL,
    end_date      DATE          NULL,
    status        ENUM('Planned','Ongoing','Completed') NOT NULL DEFAULT 'Planned',
    CONSTRAINT chk_budget CHECK (budget > 0),
    CONSTRAINT fk_proj_dept FOREIGN KEY (dept_id)
        REFERENCES department(dept_id) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- M:N relationship EMPLOYEE <-> PROJECT
CREATE TABLE works_on (
    emp_id          INT          NOT NULL,
    proj_id         CHAR(3)      NOT NULL,
    hours_per_week  DECIMAL(4,1) NOT NULL,
    role            VARCHAR(30)  NOT NULL,
    PRIMARY KEY (emp_id, proj_id),
    CONSTRAINT chk_hours CHECK (hours_per_week BETWEEN 1 AND 48),
    CONSTRAINT fk_wo_emp  FOREIGN KEY (emp_id)  REFERENCES employee(emp_id) ON DELETE CASCADE,
    CONSTRAINT fk_wo_proj FOREIGN KEY (proj_id) REFERENCES project(proj_id) ON DELETE CASCADE
);
```

### 4.2 Insert Sample Data

| Department | Location | Employees | Projects |
|---|---|---|---|
| D01 – Engineering | Bengaluru | 10 | P01 UPI Payment Gateway, P02 Mobile App Revamp, P03 AI Chatbot (Hindi-English) |
| D02 – Human Resources | Mumbai | 4 | P04 Campus Hiring Drive 2026 |
| D03 – Finance | Mumbai | 5 | P05 GST Compliance Automation |
| D04 – Marketing | New Delhi | 6 | P06 Diwali Festive Campaign |
| D05 – Operations | Chennai | 7 | P07 Warehouse Automation, P08 Tier-2 City Expansion |
| **Total** | | **32** | **8** |

Salaries are monthly, in INR. Employees 114 and 131 are not assigned to any project.

<details>
<summary><b>Click to view the INSERT statements</b></summary>

```sql
INSERT INTO department (dept_id, dept_name, location, established) VALUES
('D01', 'Engineering',     'Bengaluru', 2010),
('D02', 'Human Resources', 'Mumbai',    2010),
('D03', 'Finance',         'Mumbai',    2011),
('D04', 'Marketing',       'New Delhi', 2013),
('D05', 'Operations',      'Chennai',   2012);

INSERT INTO employee VALUES
-- Engineering (10)
(101,'Rajesh','Kumar','M','1980-03-15','2012-06-01','rajesh.kumar@company.in','Bengaluru','Engineering Manager',210000,NULL,NULL,'D01'),
(102,'Ananya','Reddy','F','1988-07-22','2015-08-17','ananya.reddy@company.in','Hyderabad','Tech Lead',165000,NULL,101,'D01'),
(103,'Vikram','Singh','M','1990-11-05','2016-01-11','vikram.singh@company.in','Bengaluru','Senior Software Engineer',135000,NULL,102,'D01'),
(104,'Sneha','Kulkarni','F','1993-02-14','2018-07-02','sneha.kulkarni@company.in','Pune','Senior Software Engineer',128000,NULL,102,'D01'),
(105,'Arjun','Nair','M','1995-09-30','2019-03-18','arjun.nair@company.in','Kochi','Software Engineer',95000,NULL,102,'D01'),
(106,'Pooja','Desai','F','1996-12-08','2020-09-07','pooja.desai@company.in','Ahmedabad','Software Engineer',88000,NULL,102,'D01'),
(107,'Karthik','Iyer','M','1997-05-19','2021-07-12','karthik.iyer@company.in','Chennai','Software Engineer',82000,NULL,103,'D01'),
(108,'Divya','Menon','F','1998-08-25','2022-08-01','divya.menon@company.in','Bengaluru','Associate Engineer',62000,NULL,103,'D01'),
(109,'Rahul','Verma','M','1999-01-10','2023-07-17','rahul.verma@company.in','Lucknow','Associate Engineer',58000,NULL,104,'D01'),
(110,'Neha','Joshi','F','2000-04-03','2024-01-08','neha.joshi@company.in','Jaipur','Associate Engineer',55000,NULL,104,'D01'),
-- Human Resources (4)
(111,'Meenakshi','Pillai','F','1982-06-11','2010-04-05','meenakshi.pillai@company.in','Mumbai','HR Manager',150000,NULL,NULL,'D02'),
(112,'Sanjay','Gupta','M','1989-10-02','2016-11-14','sanjay.gupta@company.in','Mumbai','HR Business Partner',90000,NULL,111,'D02'),
(113,'Ritu','Saxena','F','1994-03-27','2019-06-03','ritu.saxena@company.in','Delhi','Recruiter',60000,NULL,111,'D02'),
(114,'Aditya','Bhatt','M','1998-12-15','2023-02-20','aditya.bhatt@company.in','Mumbai','HR Executive',42000,NULL,111,'D02'),
-- Finance (5)
(115,'Suresh','Agarwal','M','1978-09-09','2009-01-12','suresh.agarwal@company.in','Kolkata','Finance Manager',185000,NULL,NULL,'D03'),
(116,'Kavita','Mehta','F','1987-05-17','2014-05-26','kavita.mehta@company.in','Mumbai','Senior Accountant',105000,NULL,115,'D03'),
(117,'Manoj','Tiwari','M','1991-08-21','2017-10-09','manoj.tiwari@company.in','Varanasi','Accountant',72000,NULL,116,'D03'),
(118,'Lakshmi','Rao','F','1995-01-29','2020-02-10','lakshmi.rao@company.in','Hyderabad','Financial Analyst',85000,NULL,115,'D03'),
(119,'Imran','Khan','M','1997-07-07','2022-04-18','imran.khan@company.in','Bhopal','Accounts Executive',48000,NULL,116,'D03'),
-- Marketing (6)
(120,'Nisha','Malhotra','F','1984-04-18','2013-03-04','nisha.malhotra@company.in','Delhi','Marketing Manager',160000,20000,NULL,'D04'),
(121,'Rohit','Bansal','M','1990-02-28','2017-05-15','rohit.bansal@company.in','Delhi','Brand Manager',110000,15000,120,'D04'),
(122,'Shreya','Ghosh','F','1993-09-12','2019-08-19','shreya.ghosh@company.in','Kolkata','Digital Marketing Specialist',78000,8000,121,'D04'),
(123,'Aman','Chauhan','M','1996-06-06','2021-01-25','aman.chauhan@company.in','Chandigarh','SEO Analyst',56000,NULL,121,'D04'),
(124,'Tanvi','Kapoor','F','1998-11-20','2022-11-07','tanvi.kapoor@company.in','Noida','Content Writer',45000,NULL,121,'D04'),
(125,'Farhan','Siddiqui','M','1995-03-03','2020-10-12','farhan.siddiqui@company.in','Lucknow','Sales Executive',52000,12000,120,'D04'),
-- Operations (7)
(126,'Venkatesh','Subramanian','M','1981-12-24','2011-09-19','venkatesh.s@company.in','Chennai','Operations Manager',175000,NULL,NULL,'D05'),
(127,'Deepa','Krishnan','F','1988-01-16','2015-02-02','deepa.krishnan@company.in','Chennai','Logistics Lead',98000,NULL,126,'D05'),
(128,'Harish','Gowda','M','1992-07-30','2018-04-16','harish.gowda@company.in','Mysuru','Supply Chain Analyst',76000,NULL,127,'D05'),
(129,'Swati','Mishra','F','1994-10-10','2019-12-02','swati.mishra@company.in','Patna','Operations Executive',54000,NULL,127,'D05'),
(130,'Gaurav','Yadav','M','1997-02-22','2021-06-14','gaurav.yadav@company.in','Kanpur','Operations Executive',50000,NULL,127,'D05'),
(131,'Bhavna','Thakur','F','1999-05-05','2023-09-04','bhavna.thakur@company.in','Shimla','Operations Trainee',35000,NULL,127,'D05'),
(132,'Naveen','Prasad','M','1986-08-08','2014-07-21','naveen.prasad@company.in','Chennai','Quality Manager',120000,NULL,126,'D05');

UPDATE department SET manager_id = 101 WHERE dept_id = 'D01';
UPDATE department SET manager_id = 111 WHERE dept_id = 'D02';
UPDATE department SET manager_id = 115 WHERE dept_id = 'D03';
UPDATE department SET manager_id = 120 WHERE dept_id = 'D04';
UPDATE department SET manager_id = 126 WHERE dept_id = 'D05';

INSERT INTO project VALUES
('P01','UPI Payment Gateway',       'D01','Bengaluru', 4500000,'2025-01-15','2025-12-31','Completed'),
('P02','Mobile App Revamp',         'D01','Bengaluru', 3200000,'2025-06-01','2026-12-31','Ongoing'),
('P03','AI Chatbot (Hindi-English)','D01','Hyderabad', 2800000,'2026-02-01','2027-03-31','Ongoing'),
('P04','Campus Hiring Drive 2026',  'D02','Mumbai',     600000,'2026-01-10','2026-08-31','Completed'),
('P05','GST Compliance Automation', 'D03','Mumbai',    1500000,'2025-09-01','2026-11-30','Ongoing'),
('P06','Diwali Festive Campaign',   'D04','New Delhi', 2200000,'2026-08-01','2026-11-15','Ongoing'),
('P07','Warehouse Automation',      'D05','Chennai',   5000000,'2026-04-01','2027-09-30','Ongoing'),
('P08','Tier-2 City Expansion',     'D05','Pune',      3500000,'2026-12-01',NULL,        'Planned');

INSERT INTO works_on VALUES
(101,'P01', 5,'Sponsor'),   (102,'P01',20,'Lead'),      (103,'P01',30,'Developer'),
(105,'P01',35,'Developer'), (118,'P01',10,'Analyst'),
(102,'P02',15,'Architect'), (104,'P02',30,'Lead'),      (106,'P02',40,'Developer'),
(108,'P02',35,'Developer'), (122,'P02',10,'Consultant'),
(101,'P03', 5,'Sponsor'),   (103,'P03',10,'Developer'), (107,'P03',40,'Developer'),
(109,'P03',40,'Developer'), (110,'P03',40,'Developer'), (124,'P03',15,'Content'),
(111,'P04',10,'Sponsor'),   (112,'P04',25,'Lead'),      (113,'P04',40,'Recruiter'),
(108,'P04', 5,'Panelist'),  (104,'P04', 5,'Panelist'),
(115,'P05',10,'Sponsor'),   (116,'P05',30,'Lead'),      (117,'P05',40,'Analyst'),
(119,'P05',40,'Analyst'),   (105,'P05', 5,'Developer'),
(120,'P06',15,'Sponsor'),   (121,'P06',35,'Lead'),      (122,'P06',30,'Specialist'),
(123,'P06',40,'SEO'),       (125,'P06',30,'Sales'),
(126,'P07',10,'Sponsor'),   (127,'P07',35,'Lead'),      (128,'P07',40,'Analyst'),
(129,'P07',40,'Executive'), (130,'P07',40,'Executive'), (132,'P07',15,'Quality'),
(126,'P08', 5,'Sponsor'),   (132,'P08',10,'Planning'),  (118,'P08',10,'Analyst');
```

</details>

**Row counts after inserting:**

```
+------------+-----------+
| table_name | row_count |
+------------+-----------+
| department |         5 |
| employee   |        32 |
| project    |         8 |
| works_on   |        40 |
+------------+-----------+
```

---

## 5. Queries and Output

| Concept | Queries |
|---|---|
| Selection (σ) | Q1, Q2, Q3, Q4, Q5, Q6, Q7, Q8, Q9 |
| Projection (π) | Q10, Q11, Q12, Q13 |
| Aggregate Functions | Q14, Q15 |
| GROUP BY | Q16, Q17, Q18, Q19, Q20 |
| HAVING | Q21, Q22, Q23, Q24, Q25 |
| CASE Expressions | Q26, Q27, Q28, Q29, Q30 |
| ORDER BY | Q31, Q32, Q33, Q34, Q35 |
| All Concepts Together | Q36 |

> Queries without `ORDER BY` may return rows in a different order on your machine. SQL does not guarantee row order unless you ask for it. Q13 uses `CURDATE()`, so ages and experience depend on the day you run it.

### 5.1 Selection (σ) – filtering rows with WHERE

Selection picks the **rows** that satisfy a condition. Relational algebra: σ<sub>condition</sub>(R).

#### Q1. Display all departments.

```sql
SELECT * FROM department;
```

**Output:**

```
+---------+-----------------+-----------+------------+-------------+
| dept_id | dept_name       | location  | manager_id | established |
+---------+-----------------+-----------+------------+-------------+
| D01     | Engineering     | Bengaluru |        101 |        2010 |
| D02     | Human Resources | Mumbai    |        111 |        2010 |
| D03     | Finance         | Mumbai    |        115 |        2011 |
| D04     | Marketing       | New Delhi |        120 |        2013 |
| D05     | Operations      | Chennai   |        126 |        2012 |
+---------+-----------------+-----------+------------+-------------+
```

#### Q2. Engineering employees earning more than Rs 1,00,000 per month (WHERE with AND).

```sql
SELECT emp_id, first_name, last_name, designation, salary
FROM employee
WHERE dept_id = 'D01' AND salary > 100000;
```

**Output:**

```
+--------+------------+-----------+--------------------------+-----------+
| emp_id | first_name | last_name | designation              | salary    |
+--------+------------+-----------+--------------------------+-----------+
|    101 | Rajesh     | Kumar     | Engineering Manager      | 210000.00 |
|    102 | Ananya     | Reddy     | Tech Lead                | 165000.00 |
|    103 | Vikram     | Singh     | Senior Software Engineer | 135000.00 |
|    104 | Sneha      | Kulkarni  | Senior Software Engineer | 128000.00 |
+--------+------------+-----------+--------------------------+-----------+
```

#### Q3. Employees with salary between Rs 50,000 and Rs 80,000 (BETWEEN).

```sql
SELECT emp_id, first_name, designation, salary
FROM employee
WHERE salary BETWEEN 50000 AND 80000;
```

**Output:**

```
+--------+------------+------------------------------+----------+
| emp_id | first_name | designation                  | salary   |
+--------+------------+------------------------------+----------+
|    108 | Divya      | Associate Engineer           | 62000.00 |
|    109 | Rahul      | Associate Engineer           | 58000.00 |
|    110 | Neha       | Associate Engineer           | 55000.00 |
|    113 | Ritu       | Recruiter                    | 60000.00 |
|    117 | Manoj      | Accountant                   | 72000.00 |
|    122 | Shreya     | Digital Marketing Specialist | 78000.00 |
|    123 | Aman       | SEO Analyst                  | 56000.00 |
|    125 | Farhan     | Sales Executive              | 52000.00 |
|    128 | Harish     | Supply Chain Analyst         | 76000.00 |
|    129 | Swati      | Operations Executive         | 54000.00 |
|    130 | Gaurav     | Operations Executive         | 50000.00 |
+--------+------------+------------------------------+----------+
```

#### Q4. Employees based in Mumbai, Chennai or Delhi (IN).

```sql
SELECT emp_id, first_name, last_name, city
FROM employee
WHERE city IN ('Mumbai', 'Chennai', 'Delhi');
```

**Output:**

```
+--------+------------+-------------+---------+
| emp_id | first_name | last_name   | city    |
+--------+------------+-------------+---------+
|    107 | Karthik    | Iyer        | Chennai |
|    111 | Meenakshi  | Pillai      | Mumbai  |
|    112 | Sanjay     | Gupta       | Mumbai  |
|    113 | Ritu       | Saxena      | Delhi   |
|    114 | Aditya     | Bhatt       | Mumbai  |
|    116 | Kavita     | Mehta       | Mumbai  |
|    120 | Nisha      | Malhotra    | Delhi   |
|    121 | Rohit      | Bansal      | Delhi   |
|    126 | Venkatesh  | Subramanian | Chennai |
|    127 | Deepa      | Krishnan    | Chennai |
|    132 | Naveen     | Prasad      | Chennai |
+--------+------------+-------------+---------+
```

#### Q5. Employees whose designation contains 'Manager' (LIKE).

```sql
SELECT emp_id, first_name, designation
FROM employee
WHERE designation LIKE '%Manager%';
```

**Output:**

```
+--------+------------+---------------------+
| emp_id | first_name | designation         |
+--------+------------+---------------------+
|    101 | Rajesh     | Engineering Manager |
|    111 | Meenakshi  | HR Manager          |
|    115 | Suresh     | Finance Manager     |
|    120 | Nisha      | Marketing Manager   |
|    121 | Rohit      | Brand Manager       |
|    126 | Venkatesh  | Operations Manager  |
|    132 | Naveen     | Quality Manager     |
+--------+------------+---------------------+
```

#### Q6. Employees who report to no one, i.e. top-level managers (IS NULL).

```sql
SELECT emp_id, first_name, last_name, designation
FROM employee
WHERE manager_id IS NULL;
```

**Output:**

```
+--------+------------+-------------+---------------------+
| emp_id | first_name | last_name   | designation         |
+--------+------------+-------------+---------------------+
|    101 | Rajesh     | Kumar       | Engineering Manager |
|    111 | Meenakshi  | Pillai      | HR Manager          |
|    115 | Suresh     | Agarwal     | Finance Manager     |
|    120 | Nisha      | Malhotra    | Marketing Manager   |
|    126 | Venkatesh  | Subramanian | Operations Manager  |
+--------+------------+-------------+---------------------+
```

#### Q7. Employees who earn a commission (IS NOT NULL).

```sql
SELECT emp_id, first_name, salary, commission
FROM employee
WHERE commission IS NOT NULL;
```

**Output:**

```
+--------+------------+-----------+------------+
| emp_id | first_name | salary    | commission |
+--------+------------+-----------+------------+
|    120 | Nisha      | 160000.00 |   20000.00 |
|    121 | Rohit      | 110000.00 |   15000.00 |
|    122 | Shreya     |  78000.00 |    8000.00 |
|    125 | Farhan     |  52000.00 |   12000.00 |
+--------+------------+-----------+------------+
```

#### Q8. Female employees hired on or after 1 Jan 2020, or anyone from Finance (AND / OR / dates).

```sql
SELECT emp_id, first_name, gender, hire_date, dept_id
FROM employee
WHERE (gender = 'F' AND hire_date >= '2020-01-01') OR dept_id = 'D03';
```

**Output:**

```
+--------+------------+--------+------------+---------+
| emp_id | first_name | gender | hire_date  | dept_id |
+--------+------------+--------+------------+---------+
|    106 | Pooja      | F      | 2020-09-07 | D01     |
|    108 | Divya      | F      | 2022-08-01 | D01     |
|    110 | Neha       | F      | 2024-01-08 | D01     |
|    115 | Suresh     | M      | 2009-01-12 | D03     |
|    116 | Kavita     | F      | 2014-05-26 | D03     |
|    117 | Manoj      | M      | 2017-10-09 | D03     |
|    118 | Lakshmi    | F      | 2020-02-10 | D03     |
|    119 | Imran      | M      | 2022-04-18 | D03     |
|    124 | Tanvi      | F      | 2022-11-07 | D04     |
|    131 | Bhavna     | F      | 2023-09-04 | D05     |
+--------+------------+--------+------------+---------+
```

#### Q9. Projects that are NOT completed (NOT / <>).

```sql
SELECT proj_id, proj_name, status
FROM project
WHERE status <> 'Completed';
```
