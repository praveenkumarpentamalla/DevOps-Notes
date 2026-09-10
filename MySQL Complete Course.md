# MySQL Complete Course — Full Explanations (Reference Edition)

> **How to use this document:** This is the *complete* written version of every module — for quick reference. But real mastery still needs practice. After reading each module, close the file and try the practice prompts listed at the end of that module *before* checking back. Come back to me any time and say "quiz me on Module X" or "give me practice on JOINs" and I'll run you through active-recall practice interactively.

---

# THE COURSE DATABASE (used in every module)

We use one evolving **service marketplace** database throughout.

```sql
CREATE DATABASE marketplace;
USE marketplace;

CREATE TABLE users (
    user_id       INT AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(150) NOT NULL UNIQUE,
    role          ENUM('customer','provider','employee') NOT NULL,
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE locations (
    location_id   INT AUTO_INCREMENT PRIMARY KEY,
    city          VARCHAR(100),
    state         VARCHAR(100)
);

CREATE TABLE categories (
    category_id   INT AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(100) NOT NULL
);

CREATE TABLE providers (
    provider_id   INT AUTO_INCREMENT PRIMARY KEY,
    user_id       INT NOT NULL,
    business_name VARCHAR(150),
    rating        DECIMAL(2,1) DEFAULT 0.0,
    location_id   INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (location_id) REFERENCES locations(location_id)
);

CREATE TABLE customers (
    customer_id   INT AUTO_INCREMENT PRIMARY KEY,
    user_id       INT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE services (
    service_id    INT AUTO_INCREMENT PRIMARY KEY,
    provider_id   INT NOT NULL,
    category_id   INT NOT NULL,
    title         VARCHAR(150),
    price         DECIMAL(10,2),
    FOREIGN KEY (provider_id) REFERENCES providers(provider_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE orders (
    order_id      INT AUTO_INCREMENT PRIMARY KEY,
    customer_id   INT NOT NULL,
    service_id    INT NOT NULL,
    status        ENUM('pending','confirmed','completed','cancelled') DEFAULT 'pending',
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (service_id) REFERENCES services(service_id)
);

CREATE TABLE bookings (
    booking_id     INT AUTO_INCREMENT PRIMARY KEY,
    order_id       INT NOT NULL,
    scheduled_date DATETIME,
    status         ENUM('scheduled','done','no_show') DEFAULT 'scheduled',
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE payments (
    payment_id    INT AUTO_INCREMENT PRIMARY KEY,
    order_id      INT NOT NULL,
    amount        DECIMAL(10,2),
    status        ENUM('pending','paid','failed','refunded') DEFAULT 'pending',
    paid_at       DATETIME,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE subscriptions (
    subscription_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT NOT NULL,
    plan            ENUM('basic','pro','premium'),
    start_date      DATE,
    end_date        DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE reviews (
    review_id     INT AUTO_INCREMENT PRIMARY KEY,
    order_id      INT NOT NULL,
    rating        TINYINT CHECK (rating BETWEEN 1 AND 5),
    comment       VARCHAR(500),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE employees (
    employee_id   INT AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(100),
    manager_id    INT,
    department    VARCHAR(100),
    salary        DECIMAL(10,2),
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);
```

**Relationship map:**
- `users` 1—1 `providers`/`customers` (a user *becomes* one of these)
- `providers` 1—many `services`
- `customers` 1—many `orders`
- `orders` 1—1 `bookings`, 1—1 `payments`, 1—1 `reviews`
- `categories` 1—many `services`
- `employees` self-referencing (manager_id → employee_id) — for org-chart/recursive CTE examples later


---

# MODULE 1 — Database Fundamentals 🟢

### What is a database?
A database is an organized collection of data stored so it can be easily accessed, managed, and updated. A **DBMS** (Database Management System) is the software that manages databases (MySQL, PostgreSQL, Oracle). An **RDBMS** (Relational DBMS) organizes data into tables with rows and columns and enforces relationships between them — MySQL is an RDBMS.

### Why does it exist?
Before databases, applications stored data in flat files. Flat files can't easily enforce rules ("every order must belong to a real customer"), can't be searched efficiently at scale, and break easily when multiple people write at once. RDBMSs solve all three: structure, integrity, and concurrency.

### Analogy
- **Database** → a warehouse
- **Table** → a labeled shelf/category in the warehouse (e.g., "Orders" shelf)
- **Row** → one item on that shelf (one specific order)
- **Column** → a property every item on that shelf has (order_id, date, amount)

### Core vocabulary
| Term | Meaning |
|---|---|
| Schema | The structure/blueprint of a database (tables + relationships) — in MySQL, "schema" and "database" are the same thing |
| Primary Key (PK) | A column (or set of columns) that uniquely identifies each row. Cannot be NULL, cannot repeat. |
| Foreign Key (FK) | A column that points to a Primary Key in another table, creating a relationship |
| Candidate Key | Any column that *could* have been chosen as the primary key (e.g., email is a candidate key on `users`) |
| Composite Key | A primary key made of more than one column together |
| Unique Key | Like a primary key (no duplicates) but a table can have several, and it *can* allow one NULL |
| Constraint | A rule enforced by the database (NOT NULL, UNIQUE, CHECK, FOREIGN KEY, DEFAULT) |
| NULL | "Unknown" or "not applicable" — NOT the same as zero or empty string |

### Relationships
- **One-to-one:** `users` ↔ `providers` — one user is at most one provider profile.
- **One-to-many:** `providers` → `services` — one provider offers many services.
- **Many-to-many:** e.g., customers and categories they've bought from — needs a **junction table** in between (we'll build one in Module 14).

### Common mistakes
- Confusing "schema" with "table" (in MySQL they're not the same thing — schema = database).
- Assuming NULL equals 0 or "" — it doesn't; `NULL = NULL` is not true, it's *unknown*.
- Forgetting a table needs a primary key at all.

### How to remember it
**"PK identifies. FK connects."** Every table should have one column whose only job is to be a unique fingerprint (PK); relationships happen when one table's column points at another table's PK (FK).

### Interview angle
- What's the difference between a candidate key and a primary key?
- Why can a foreign key reference a non-primary-key column? (It must reference a UNIQUE or PRIMARY key.)
- What does NULL mean in a database, and why isn't `column = NULL` valid syntax?

### Production angle
Every production table should have: a primary key, explicit data types (not everything as VARCHAR), and foreign keys enforced (or at minimum, application-level equivalent logic) so that orphaned rows (e.g., a payment for a deleted order) can't happen.

**Practice before moving on:** Sketch (on paper) the PK and FK for `orders`, `payments`, and `bookings`. Ask yourself: "if I deleted a customer, what happens to their orders?" — that question is the seed for Module 14 (design).

---

# MODULE 2 — SQL Fundamentals (DDL & DML) 🟢

### What is it?
SQL (Structured Query Language) is how you talk to the database. It splits into:
- **DDL** (Data Definition Language): CREATE, ALTER, DROP, TRUNCATE — defines structure
- **DML** (Data Manipulation Language): INSERT, SELECT, UPDATE, DELETE — manipulates data

### Syntax patterns (memorize the shape, not the example)

**CREATE TABLE**
```sql
CREATE TABLE table_name (
    column1 datatype constraints,
    column2 datatype constraints
);
```

**INSERT**
```sql
INSERT INTO table_name (col1, col2) VALUES (val1, val2);
```

**SELECT**
```sql
SELECT columns
FROM table
WHERE condition
ORDER BY column
LIMIT n;
```

**UPDATE**
```sql
UPDATE table_name
SET column = value
WHERE condition;
```

**DELETE**
```sql
DELETE FROM table_name
WHERE condition;
```

### Break down the keywords
- `WHERE` filters **rows** before grouping.
- `ORDER BY` sorts the final result (ASC default, DESC for descending).
- `LIMIT n` caps how many rows come back — always pair with ORDER BY or your "top N" is meaningless (rows aren't guaranteed in any order without it).
- `DISTINCT` removes duplicate rows from the result.

### Real-world examples (marketplace)
```sql
-- All confirmed orders
SELECT * FROM orders WHERE status = 'confirmed';

-- Top 5 highest-priced services
SELECT title, price FROM services ORDER BY price DESC LIMIT 5;

-- Update a payment to paid
UPDATE payments SET status = 'paid', paid_at = NOW() WHERE order_id = 12;

-- Remove a cancelled order older than a year
DELETE FROM orders WHERE status = 'cancelled' AND created_at < '2025-01-01';
```

### Common mistakes
- Running `UPDATE`/`DELETE` **without WHERE** — this hits every row in the table. Always write the `SELECT` with the same WHERE first, check the rows, *then* change SELECT to UPDATE/DELETE.
- Forgetting `ORDER BY` before `LIMIT` when "top N" is required.
- Using `=` to compare with NULL instead of `IS NULL`.

### DELETE vs TRUNCATE vs DROP (comparison)
| | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| What | Removes rows (can filter with WHERE) | Removes **all** rows, resets AUTO_INCREMENT | Removes the whole table structure |
| Speed | Slower (logged row by row) | Fast (deallocates pages) | Instant |
| Rollback | Can be rolled back in a transaction | Cannot be rolled back in MySQL/InnoDB in most cases | Cannot be rolled back |
| Use when | You need to remove specific rows | You want to empty a table completely | You want the table gone entirely |

### How to remember it
**"SELECT reads. INSERT adds. UPDATE changes. DELETE removes."** — say it in that order, every time, before touching a table.

### Interview angle
- Why is TRUNCATE faster than DELETE?
- What happens if you run UPDATE without a WHERE clause?
- Difference between DROP TABLE and DROP DATABASE?

### Production angle
Production teams often **disable autocommit** and wrap DELETE/UPDATE in a transaction with a `SELECT` sanity check first. Many outages have been caused by an unfiltered DELETE — this is one of the most repeated real-world incidents in the industry.

**Practice before moving on:** Write (don't run yet) a query to mark all `pending` orders older than 30 days as `cancelled`. Then write the SELECT you'd run *first* to verify you're touching the right rows.

---

# MODULE 3 — SQL Execution Order (why queries "feel wrong" sometimes) 🟢🟡

### What is it?
SQL is written in one order but **executed** in a different logical order:

```
FROM  →  JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

### Why does it matter?
This explains real errors:
- You **cannot** use a column alias defined in SELECT inside a WHERE clause — because WHERE runs *before* SELECT.
- You **can** use a SELECT alias in ORDER BY — because ORDER BY runs *after* SELECT.
- HAVING filters *after* grouping (on aggregated results); WHERE filters *before* grouping (on raw rows).

### Example that breaks without knowing this
```sql
-- ERROR: alias not recognized in WHERE
SELECT price * 1.1 AS price_with_tax
FROM services
WHERE price_with_tax > 100;   -- ❌ fails

-- WORKS: same alias is fine in ORDER BY
SELECT price * 1.1 AS price_with_tax
FROM services
ORDER BY price_with_tax;      -- ✅ works
```

### How to remember it
**"Find it, join it, filter it, group it, filter the groups, pick columns, dedupe, sort, cut."** Say this sentence — it's literally the execution order in plain English.

### Interview angle
- Why can't you reference a SELECT alias in a WHERE clause?
- What's the real difference between WHERE and HAVING, tied to execution order?

**Practice before moving on:** Predict whether this fails or works, and why:
```sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING order_count > 3
ORDER BY order_count DESC;
```
(Try to reason it out before checking — HAVING referencing a SELECT alias works in MySQL specifically, which is a MySQL-specific relaxation of the standard order — a good thing to know.)


---

# MODULE 4 — Aggregation: GROUP BY, HAVING, Aggregate Functions 🟢🟡

### What is it?
Aggregation collapses many rows into a summary — "how many," "total," "average" — per group.

### Why does it exist?
Raw rows answer "what happened." Aggregation answers "what happened *overall* or *per category*" — which is what dashboards, reports, and business decisions actually need.

### Analogy
Think of GROUP BY as sorting items into buckets first (e.g., one bucket per customer), then asking a question about the contents of each bucket (COUNT, SUM, AVG) rather than about each individual item.

### Syntax pattern
```sql
SELECT group_column, AGG_FUNC(column)
FROM table
WHERE row_filter
GROUP BY group_column
HAVING group_filter
ORDER BY ...;
```

### Aggregate functions
`COUNT(*)`, `COUNT(column)` (ignores NULLs), `SUM()`, `AVG()`, `MIN()`, `MAX()`

### Real-world examples
```sql
-- Number of orders per customer
SELECT customer_id, COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id;

-- Average service price per category
SELECT category_id, AVG(price) AS avg_price
FROM services
GROUP BY category_id;

-- Customers with more than 3 orders (filter on the AGGREGATE)
SELECT customer_id, COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 3;

-- Total revenue from paid payments only, per month
SELECT DATE_FORMAT(paid_at, '%Y-%m') AS month, SUM(amount) AS revenue
FROM payments
WHERE status = 'paid'
GROUP BY month
ORDER BY month;
```

### WHERE vs HAVING (comparison)
| | WHERE | HAVING |
|---|---|---|
| What | Filters raw rows | Filters grouped/aggregated results |
| When it runs | Before GROUP BY | After GROUP BY |
| Can use aggregate functions? | No | Yes |
| Example | `WHERE status = 'paid'` | `HAVING COUNT(*) > 3` |

### Common mistakes
- Selecting a column that's neither aggregated nor in GROUP BY (MySQL allows this loosely by default, but it produces **arbitrary** values — a classic silent bug).
- Using HAVING for a filter that WHERE could do more efficiently (WHERE reduces rows *before* the expensive grouping work).
- Forgetting `COUNT(column)` skips NULLs while `COUNT(*)` doesn't.

### How to remember it
**"WHERE picks players before the game. HAVING picks winners after the game."**

### Interview angle
- Why can't you use an aggregate function in WHERE?
- What's the difference between COUNT(*) and COUNT(column_name)?
- Write a query to find categories with average service price above $100.

### Production angle
Poorly filtered aggregation queries (e.g., `GROUP BY` on an unindexed column across millions of rows) are one of the most common sources of slow dashboards. This connects directly to Module 11 (indexing).

**Practice before moving on:**
1. Find the number of services each provider offers.
2. Find providers who have more than 5 services.
3. Find the total amount paid per customer (needs a JOIN — try it now, we'll clean it up in Module 5).

---

# MODULE 5 — JOINs 🟡

### What is it?
A JOIN combines rows from two or more tables based on a related column, usually a foreign key pointing to a primary key.

### Why does it exist?
Normalized databases split data across tables to avoid duplication (a customer's name shouldn't be copy-pasted into every order row). JOINs let you reassemble that data for a single query result.

### Analogy
Think of two spreadsheets — `orders` and `customers` — each on a separate sheet of paper. A JOIN is physically taping matching rows together side-by-side wherever `orders.customer_id` matches `customers.customer_id`.

### Syntax pattern
```sql
SELECT columns
FROM table1
[INNER|LEFT|RIGHT] JOIN table2
    ON table1.key = table2.key;
```

### Types of JOIN
- **INNER JOIN**: only rows that match in both tables
- **LEFT JOIN**: all rows from the left table, matched data from the right (NULL if no match)
- **RIGHT JOIN**: mirror of LEFT JOIN
- **FULL JOIN**: MySQL has no native FULL JOIN — simulate with `LEFT JOIN UNION RIGHT JOIN`
- **SELF JOIN**: a table joined to itself (e.g., employees to their managers)
- **CROSS JOIN**: every row of table1 with every row of table2 (Cartesian product — rare, usually a mistake if unintended)

### Real-world examples
```sql
-- Every order with the customer's name (INNER JOIN — only matched rows)
SELECT o.order_id, u.name, o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN users u ON c.user_id = u.user_id;

-- Every provider, even ones with NO services yet (LEFT JOIN)
SELECT p.business_name, s.title
FROM providers p
LEFT JOIN services s ON p.provider_id = s.provider_id;

-- Self join: employee with their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;

-- Total paid per customer (answers last module's practice question)
SELECT u.name, SUM(p.amount) AS total_paid
FROM payments p
JOIN orders o ON p.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN users u ON c.user_id = u.user_id
WHERE p.status = 'paid'
GROUP BY u.name;
```

### Output example
For the LEFT JOIN example: a provider with zero services still appears once, with `s.title` shown as `NULL` — this is the signature of a LEFT JOIN and the #1 way to spot one in someone else's query.

### Common mistakes
- Using INNER JOIN when you actually need LEFT JOIN (silently dropping rows with no match — e.g., "show all providers" quietly excludes providers with no services).
- **Fan-out**: joining one-to-many relationships (e.g., orders to payments to reviews all at once) multiplies rows and inflates SUM()/COUNT() results. Fix with separate aggregated subqueries or careful GROUP BY.
- Forgetting the `ON` condition (accidental CROSS JOIN — row count explodes).
- Ambiguous column names when two joined tables share a column name (always alias tables: `o.status` not `status`).

### INNER JOIN vs LEFT JOIN (comparison)
| | INNER JOIN | LEFT JOIN |
|---|---|---|
| What | Only matching rows | All left rows + matches (or NULL) |
| Use when | You only want records that exist in both | You want to keep all "primary" records regardless of match |
| Mistake risk | Silently drops unmatched rows | Can silently duplicate rows on fan-out |

### How to remember it
**"INNER = both must show up. LEFT = left never gets left out."**

### Interview angle
- Explain the difference between INNER and LEFT JOIN with an example.
- What causes duplicate rows in a JOIN, and how do you fix it?
- Write a self-join query to find employees who earn more than their manager.

### Production angle
JOIN performance is where most production slowness starts. An unindexed foreign key column turns a JOIN into a full scan of one table for every row of the other — this connects directly into Module 11.

**Practice before moving on:**
1. List every booking with the customer's name and service title (3-table join).
2. List every provider and their average rating from reviews (needs JOIN + GROUP BY together — combine Module 4 + 5).
3. Find employees who have no manager (self-join, filter on NULL).


---

# MODULE 6 — Subqueries 🟡

### What is it?
A subquery is a query nested inside another query. It can appear in SELECT, FROM, WHERE, or HAVING.

### Why does it exist?
Some questions need an intermediate answer first. "Find customers who spent more than the average customer" requires knowing the average *before* you can filter — a subquery computes that intermediate value.

### Types
- **Scalar subquery**: returns one value
- **Column subquery**: returns one column, many rows (use with IN)
- **Correlated subquery**: references the outer query's row — runs once *per outer row*

### Syntax patterns
```sql
-- Scalar subquery in WHERE
SELECT * FROM services WHERE price > (SELECT AVG(price) FROM services);

-- Column subquery with IN
SELECT * FROM customers WHERE customer_id IN (SELECT customer_id FROM orders WHERE status = 'completed');

-- Correlated subquery
SELECT * FROM providers p
WHERE price_of_highest_service > (
    SELECT AVG(s2.price) FROM services s2 WHERE s2.provider_id = p.provider_id
);

-- EXISTS
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

### Real-world example
```sql
-- Providers whose average service price is above the marketplace-wide average
SELECT p.business_name,
       (SELECT AVG(s.price) FROM services s WHERE s.provider_id = p.provider_id) AS avg_price
FROM providers p
WHERE (SELECT AVG(s.price) FROM services s WHERE s.provider_id = p.provider_id)
      > (SELECT AVG(price) FROM services);

-- Customers who have never placed an order (NOT EXISTS)
SELECT u.name
FROM users u
JOIN customers c ON u.user_id = c.user_id
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

### IN vs EXISTS vs JOIN (comparison)
| | IN | EXISTS | JOIN |
|---|---|---|---|
| What | Checks membership in a list | Checks if any matching row exists | Combines rows from both tables |
| Best when | Subquery result is small, no NULLs | Subquery involves a correlated check, especially with large tables | You need columns from both tables in the result |
| NULL danger | `NOT IN` breaks silently if the subquery returns any NULL | Not affected by NULLs the same way | N/A |
| Speed | Can be slow if subquery result is huge | Often optimized well since MySQL can stop at first match | Usually fastest for combining data |

### Common mistakes
- Using `NOT IN` with a subquery that can return NULL — this silently returns **zero rows**, a very common and hard-to-spot bug.
- Writing a correlated subquery when a JOIN would be simpler and faster.
- Forgetting a correlated subquery re-runs for every outer row — fine for small tables, dangerous for large ones (Module 11 territory).

### How to remember it
**"Subquery answers a question the main query needs before it can even ask its own question."**

### Interview angle
- Why is `NOT IN` risky with subqueries?
- What's the difference between a correlated and non-correlated subquery?
- Rewrite a correlated subquery as a JOIN (and explain when you would/wouldn't).

### Production angle
Correlated subqueries scale poorly (O(n×m) style execution) — production teams usually convert them to JOINs or window functions once tables grow. EXPLAIN (Module 11) will show a `DEPENDENT SUBQUERY` warning sign.

**Practice before moving on:**
1. Find customers who have spent more than customer #5.
2. Find services that have never been ordered (NOT EXISTS or LEFT JOIN + IS NULL — try both).
3. Find providers with above-average ratings.

---

# MODULE 7 — CTEs (Common Table Expressions) 🟡🔴

### What is it?
A CTE is a named, temporary result set defined with `WITH`, used inside a bigger query. Think of it as giving a subquery a name and using it like a real table.

### Why does it exist?
Nested subqueries become unreadable fast. CTEs let you build a query in readable steps, and they let you do something subqueries alone cannot: **recursion** (self-referencing queries, like traversing a management hierarchy).

### Syntax pattern
```sql
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name WHERE ...;
```

Multiple CTEs:
```sql
WITH cte1 AS (...), cte2 AS (...)
SELECT * FROM cte1 JOIN cte2 ON ...;
```

Recursive CTE:
```sql
WITH RECURSIVE cte_name AS (
    -- anchor member (base case)
    SELECT ...
    UNION ALL
    -- recursive member (references cte_name itself)
    SELECT ... FROM table JOIN cte_name ON ...
)
SELECT * FROM cte_name;
```

### Real-world examples
```sql
-- Readable multi-step query: top-spending customers this year
WITH yearly_spend AS (
    SELECT c.customer_id, SUM(p.amount) AS total_spent
    FROM payments p
    JOIN orders o ON p.order_id = o.order_id
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE p.status = 'paid' AND YEAR(p.paid_at) = 2026
    GROUP BY c.customer_id
)
SELECT * FROM yearly_spend WHERE total_spent > 500;

-- Recursive CTE: full management chain under a given employee
WITH RECURSIVE org_chart AS (
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees WHERE employee_id = 1  -- top of the tree
    UNION ALL
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY level;
```

### Output example
The recursive CTE above returns every employee under employee #1, with a `level` column showing how many steps down the hierarchy they are — level 1 is the root, level 2 their direct reports, and so on.

### CTE vs subquery vs temp table (comparison)
| | CTE | Subquery | Temp Table |
|---|---|---|---|
| Readability | High (named, step-by-step) | Lower when nested deeply | High but heavier to set up |
| Reusable in same query? | Yes, multiple times | No, must repeat it | Yes, across multiple statements |
| Can be recursive? | Yes | No | No (needs a loop in application code or procedure) |
| Persists after query? | No | No | Yes, until dropped or session ends |

### Common mistakes
- Forgetting `RECURSIVE` keyword when writing a self-referencing CTE.
- Writing a recursive CTE with no terminating condition — infinite loop (MySQL has a recursion depth limit that will error out, which is a safety net, not a fix).
- Treating a CTE like it's cached/materialized for performance — MySQL may re-run it multiple times depending on the query plan.

### How to remember it
**"CTE = a subquery with a name tag, that can also loop back on itself."**

### Interview angle
- When would you use a CTE instead of a subquery?
- Explain how a recursive CTE works, step by step, using an org chart.
- What's the anchor member and recursive member in a recursive CTE?

### Production angle
Recursive CTEs are the standard modern way to handle hierarchical data (categories with subcategories, org charts, threaded comments) without needing a separate nested-set or adjacency-list workaround.

**Practice before moving on:**
1. Write a CTE that computes total revenue per provider, then filter to providers earning more than $1,000.
2. Write a recursive CTE to list all subcategories under a given parent category (assume `categories` gets a `parent_category_id` column — we'll add this properly in Module 14).


---

# MODULE 8 — Window Functions 🔴

### What is it?
A window function performs a calculation across a set of rows related to the current row — **without** collapsing them like GROUP BY does. You get the detail rows *and* the aggregate/ranking info together.

### Why does it exist?
GROUP BY forces you to choose: either row-level detail, or aggregated summary — not both. Window functions give you "this row's value, plus its rank/running total/comparison to neighbors" in the same row.

### Analogy
GROUP BY is like putting people into rooms and only reporting room totals. A window function is like giving every person a badge showing their rank *within their room*, while everyone still stays visible individually.

### Syntax pattern
```sql
SELECT columns,
       WINDOW_FUNC(column) OVER (
           PARTITION BY grouping_column
           ORDER BY sort_column
       ) AS result
FROM table;
```

### Key functions
- `ROW_NUMBER()` — unique sequential number per row, no ties
- `RANK()` — same rank for ties, but skips numbers after a tie (1,1,3)
- `DENSE_RANK()` — same rank for ties, no skipping (1,1,2)
- `LEAD(col, n)` / `LAG(col, n)` — value from a following/preceding row
- Aggregate functions used as window functions: `SUM() OVER (...)`, `AVG() OVER (...)` for running totals/moving averages

### Real-world examples
```sql
-- Rank services within each category by price (highest = rank 1)
SELECT title, category_id, price,
       RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS price_rank
FROM services;

-- Running total of payments per customer, ordered by date
SELECT customer_id, paid_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY paid_at) AS running_total
FROM payments
WHERE status = 'paid';

-- Top-2 highest-priced services PER provider (classic "top-N-per-group")
SELECT * FROM (
    SELECT title, provider_id, price,
           ROW_NUMBER() OVER (PARTITION BY provider_id ORDER BY price DESC) AS rn
    FROM services
) ranked
WHERE rn <= 2;

-- Compare each order's amount to the previous order from the same customer
SELECT customer_id, order_id, created_at,
       LAG(created_at) OVER (PARTITION BY customer_id ORDER BY created_at) AS previous_order_date
FROM orders;
```

### RANK vs DENSE_RANK vs ROW_NUMBER (comparison)
| Rows (by price) | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|
| 100 | 1 | 1 | 1 |
| 100 (tie) | 2 | 1 | 1 |
| 90 | 3 | 3 | 2 |

### Common mistakes
- Forgetting `PARTITION BY` — without it, the window function treats the *entire result set* as one group, not per-category/per-customer.
- Using a window function result directly in the same query's WHERE clause — not allowed (window functions run *after* WHERE in execution order); you must wrap it in a subquery/CTE and filter the outer layer, as shown in the top-N example.
- Confusing RANK (skips numbers on ties) with DENSE_RANK (doesn't skip).

### How to remember it
**"PARTITION BY makes separate rooms. ORDER BY inside OVER() decides the order within each room."**

### Interview angle
- Explain the difference between RANK, DENSE_RANK, and ROW_NUMBER with an example.
- How would you find the top 3 highest-paid employees per department?
- Why can't you filter directly on a window function in WHERE?

### Production angle
Window functions replace what used to require multiple self-joins or application-side loops — leaderboards, "top N per group" reports, and running totals are now single, index-friendly SQL queries.

**Practice before moving on:**
1. Find the top 3 highest-rated reviews per service.
2. Compute a 3-order moving average of payment amounts per customer.
3. Find the difference in days between each customer's consecutive orders (LAG + DATEDIFF).

---

# MODULE 9 — JSON, Dates, Strings & Generated Columns 🔴

### What is it?
Modern MySQL (8.x) supports storing and querying semi-structured data (JSON), rich date/time math, string manipulation, and computed columns that derive their value from other columns automatically.

### JSON
```sql
-- Example: services.metadata JSON column storing flexible attributes
SELECT title, JSON_EXTRACT(metadata, '$.duration_minutes') AS duration
FROM services;

-- Shorthand arrow syntax
SELECT title, metadata->>'$.duration_minutes' AS duration FROM services;

-- Filtering on a JSON field
SELECT * FROM services WHERE metadata->>'$.remote' = 'true';
```

### Generated columns
```sql
ALTER TABLE orders
ADD COLUMN order_year INT GENERATED ALWAYS AS (YEAR(created_at)) STORED;
```
A generated column's value is computed automatically from an expression — `STORED` means it's saved on disk (and can be indexed), `VIRTUAL` means it's computed on read.

### Date/time functions
```sql
SELECT NOW(), CURDATE(), DATE_ADD(NOW(), INTERVAL 7 DAY), DATEDIFF('2026-09-10','2026-01-01');
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month FROM orders;
```

### String functions
```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;
SELECT UPPER(name), LOWER(email), TRIM(name), SUBSTRING(name, 1, 3) FROM users;
SELECT * FROM services WHERE title LIKE '%repair%';
SELECT * FROM services WHERE title REGEXP '^[A-Z]';
```

### Common mistakes
- Storing structured, frequently-queried data as JSON just because it's convenient — if you regularly filter/join on a field, it usually belongs as a real column with an index.
- Comparing dates as strings without considering time zones or truncation issues (comparing DATETIME to DATE truncates time — be explicit).
- Forgetting `LIKE '%text%'` can't use a standard index efficiently (a leading wildcard forces a full scan) — see Module 11.

### How to remember it
**"JSON is for shape that changes. Real columns are for data you filter, join, or index on."**

### Interview angle
- When would you use a JSON column instead of a normal column?
- What's the difference between a STORED and VIRTUAL generated column?
- Why is `LIKE '%something'` slow on a large table?

### Production angle
JSON columns are common for things like user preferences or API payload snapshots, but overusing them defeats the relational model's biggest strength: enforced structure and fast indexed lookups.

**Practice before moving on:** Add a JSON `metadata` column to `services` for optional attributes (like `duration_minutes`, `remote`), and write a query filtering on one of those attributes.


---

# MODULE 10 — Views 🟡

### What is it?
A view is a saved, named SELECT query that behaves like a virtual table — you query it just like a table, but it always reflects live underlying data.

### Why does it exist?
Views hide complexity (a 4-table JOIN becomes `SELECT * FROM provider_summary`) and can restrict access (show only certain columns to certain users).

### Syntax
```sql
CREATE VIEW provider_summary AS
SELECT p.provider_id, p.business_name, COUNT(s.service_id) AS total_services, AVG(s.price) AS avg_price
FROM providers p
LEFT JOIN services s ON p.provider_id = s.provider_id
GROUP BY p.provider_id, p.business_name;

SELECT * FROM provider_summary WHERE avg_price > 100;
```

### View vs Table (comparison)
| | View | Table |
|---|---|---|
| Stores data? | No (usually) — runs the underlying query each time | Yes |
| Updatable? | Only simple views (single table, no GROUP BY/JOIN in most cases) | Always |
| Use for | Simplifying complex queries, access control | Actual data storage |

### Common mistakes
- Expecting a view to be fast just because it "looks like a table" — it re-runs the underlying query (and its performance cost) every time, unless materialized manually.
- Trying to UPDATE/INSERT into a view built from a JOIN or aggregation — usually not allowed.

### How to remember it
**"A view is a nickname for a query, not a copy of the data."**

### Interview angle
- Can you update data through a view? When?
- What's the performance implication of stacking views on top of views?

### Production angle
Views are widely used to give reporting tools or junior developers a simplified, safe interface to complex schemas without exposing raw tables or requiring them to rewrite JOIN logic every time.

**Practice before moving on:** Create a view `customer_spending` showing each customer's name and total amount paid, then query it filtered to customers who've spent over $200.

---

# MODULE 11 — Indexes & Query Optimization 🔴⚫

### What is it?
An index is a separate, sorted data structure (a B-tree in InnoDB) that lets MySQL find rows without scanning the entire table — like a book's index lets you jump to a page instead of reading cover to cover.

### Why does it exist?
Without an index, `WHERE email = 'x@example.com'` on a million-row table means checking all million rows (a **full table scan**). An index on `email` turns that into a handful of comparisons.

### Syntax
```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_services_provider_category ON services(provider_id, category_id); -- composite
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### Composite indexes & the leftmost-prefix rule
An index on `(provider_id, category_id)` can be used for:
- `WHERE provider_id = 5` ✅
- `WHERE provider_id = 5 AND category_id = 2` ✅
- `WHERE category_id = 2` ❌ (can't skip the leftmost column)

**Memory trick:** think of a phone book sorted by (last name, first name) — you can jump straight to "Smith," or "Smith, John," but you can't efficiently find "everyone named John" without scanning the whole book.

### Covering index
An index that contains *every* column the query needs means MySQL never has to touch the actual table row — it reads the index and stops. Fastest possible read.

### EXPLAIN and EXPLAIN ANALYZE
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```
Key things to look for in EXPLAIN output:
- `type: ALL` = full table scan (bad on large tables)
- `type: ref`/`range`/`const` = index being used (good)
- `key: NULL` = no index used
- `Extra: Using filesort` = MySQL had to sort results manually (can be slow)
- `Extra: Using temporary` = a temp table was needed (often from GROUP BY/DISTINCT without a helpful index)

### Real diagnostic example
```sql
-- Slow (no index on status, and SELECT * pulls everything)
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20;

-- Better: composite index (status, created_at) supports both the filter and sort
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
```

### Pagination: OFFSET problem & keyset pagination
```sql
-- Gets slower as OFFSET grows — MySQL still scans/discards all skipped rows
SELECT * FROM orders ORDER BY order_id LIMIT 20 OFFSET 100000;

-- Keyset (seek) pagination — fast at any depth, uses the index directly
SELECT * FROM orders WHERE order_id > 100000 ORDER BY order_id LIMIT 20;
```

### Common mistakes
- Indexing every column "just in case" — indexes speed up reads but slow down INSERT/UPDATE/DELETE (they must be maintained) and use disk space.
- Wrapping an indexed column in a function in WHERE (`WHERE YEAR(created_at) = 2026`) — this disables the index. Rewrite as a range: `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'`.
- Using `SELECT *` when only 2 columns are needed — prevents covering-index optimization and wastes I/O.
- Assuming more indexes always means faster — on write-heavy tables, over-indexing slows everything down.

### INDEX vs Composite Index (comparison)
| | Single-column index | Composite index |
|---|---|---|
| Use when | Filtering/sorting by one column often | Filtering/sorting by a fixed combination of columns |
| Leftmost-prefix rule | N/A | Only the leftmost column(s) can be used alone |

### How to remember it
**"Index = a shortcut. Every shortcut costs a bit of write-time to maintain."**

### Interview angle
- What does EXPLAIN's `type: ALL` mean, and how do you fix it?
- Explain the leftmost-prefix rule with an example.
- Why does OFFSET-based pagination get slower on later pages, and what's the fix?

### Production angle
Index tuning is one of the highest-leverage skills in production MySQL — a single missing index is the most common root cause behind "the API suddenly got slow" incidents (this connects directly into Module 17).

**Practice before moving on:**
1. Run `EXPLAIN` (mentally, or on a real MySQL instance) for `SELECT * FROM services WHERE category_id = 3 ORDER BY price DESC;` — what index would help, and why?
2. Rewrite a `WHERE DATE(created_at) = '2026-09-10'` filter to be index-friendly.

---

# MODULE 12 — Transactions & Concurrency 🔴⚫

### What is it?
A transaction groups multiple statements into one all-or-nothing unit. Either every statement succeeds and is saved (`COMMIT`), or if something fails, none of it is saved (`ROLLBACK`).

### Why does it exist?
Consider a bank transfer: subtract from account A, add to account B. If the server crashes between those two statements, without a transaction you'd lose money. A transaction guarantees both happen, or neither does.

### Syntax
```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;   -- or ROLLBACK; if something went wrong

-- Partial rollback point
START TRANSACTION;
UPDATE orders SET status = 'confirmed' WHERE order_id = 10;
SAVEPOINT before_payment;
UPDATE payments SET status = 'paid' WHERE order_id = 10;
ROLLBACK TO before_payment;  -- undoes only the payment update
COMMIT;
```

### ACID
- **Atomicity** — all-or-nothing (the transfer example above)
- **Consistency** — the database moves from one valid state to another (constraints always hold)
- **Isolation** — concurrent transactions don't interfere with each other's intermediate state
- **Durability** — once committed, data survives even a crash

### Isolation levels & anomalies
| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ (MySQL default) | Prevented | Prevented | Mostly prevented (InnoDB uses gap locks) |
| SERIALIZABLE | Prevented | Prevented | Prevented |

- **Dirty read**: reading another transaction's *uncommitted* change
- **Non-repeatable read**: re-reading the same row twice in one transaction gives different results because another transaction committed a change in between
- **Phantom read**: re-running the same query returns a *different set of rows* because rows were inserted/deleted by another transaction

### Real scenario: order placement (walkthrough)
1. Transaction starts.
2. Check inventory/service availability.
3. Insert the order row.
4. Insert the payment row.
5. If payment fails → ROLLBACK (order never existed as far as the database is concerned).
6. If all succeed → COMMIT.

This prevents the classic bug: an order existing with no successful payment, or a payment existing with no order.

### Common mistakes
- Leaving a transaction open for a long time (a "long-running transaction") — this holds locks and blocks other transactions, and is a very common real production incident.
- Forgetting that autocommit is ON by default in MySQL — every single statement is its own transaction unless you explicitly `START TRANSACTION`.
- Not handling deadlocks in application code — MySQL will kill one of the two deadlocked transactions automatically; your app must catch that error and retry.

### How to remember it
**"Transactions = a promise. Either the whole promise is kept (COMMIT), or none of it happened (ROLLBACK)."**

### Interview angle
- Explain ACID with a real example.
- What's the difference between a dirty read and a non-repeatable read?
- How does MySQL handle a deadlock, and what should your application do about it?
- Why is REPEATABLE READ the default isolation level in MySQL/InnoDB (rather than READ COMMITTED, which is more common elsewhere)?

### Production angle
Every payment, inventory, or multi-step write operation in a real system should be wrapped in a transaction. Long-running transactions and unindexed lookups inside transactions are among the top causes of production lock contention and deadlocks.

**Practice before moving on:** Write the full transaction for placing an order end-to-end (order → booking → payment), including where you'd put a SAVEPOINT and why.


---

# MODULE 13 — Stored Procedures, Functions, Triggers, Events 🔴

### Stored Procedures
Reusable, saved SQL routines that can take parameters and run multiple statements.
```sql
DELIMITER //
CREATE PROCEDURE PlaceOrder(IN p_customer_id INT, IN p_service_id INT)
BEGIN
    INSERT INTO orders (customer_id, service_id, status) VALUES (p_customer_id, p_service_id, 'pending');
END //
DELIMITER ;

CALL PlaceOrder(3, 7);
```

### Functions
Like procedures, but must return a single value and can be used inline in a SELECT.
```sql
DELIMITER //
CREATE FUNCTION GetCustomerTotalSpend(p_customer_id INT) RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE total DECIMAL(10,2);
    SELECT SUM(p.amount) INTO total
    FROM payments p JOIN orders o ON p.order_id = o.order_id
    WHERE o.customer_id = p_customer_id AND p.status = 'paid';
    RETURN IFNULL(total, 0);
END //
DELIMITER ;

SELECT name, GetCustomerTotalSpend(customer_id) FROM customers;
```

### Procedure vs Function (comparison)
| | Procedure | Function |
|---|---|---|
| Returns | Zero, one, or multiple result sets/OUT params | Exactly one value |
| Callable from SELECT? | No | Yes |
| Use when | Multi-step workflows (place order, process refund) | A reusable calculation embedded in queries |

### Triggers
Code that runs automatically before/after an INSERT, UPDATE, or DELETE.
```sql
DELIMITER //
CREATE TRIGGER after_payment_paid
AFTER UPDATE ON payments
FOR EACH ROW
BEGIN
    IF NEW.status = 'paid' AND OLD.status <> 'paid' THEN
        UPDATE orders SET status = 'confirmed' WHERE order_id = NEW.order_id;
    END IF;
END //
DELIMITER ;
```

### Events
Scheduled tasks, like a cron job inside MySQL.
```sql
CREATE EVENT cancel_stale_orders
ON SCHEDULE EVERY 1 DAY
DO
  UPDATE orders SET status = 'cancelled'
  WHERE status = 'pending' AND created_at < NOW() - INTERVAL 7 DAY;
```

### Common mistakes
- Putting complex business logic in triggers that's invisible to developers reading application code — hard-to-debug "magic" behavior.
- Forgetting `DETERMINISTIC` on functions when required, or using non-deterministic functions inside `STORED` generated columns.
- Not testing trigger cascades — one trigger firing another trigger can create unexpected chains.

### How to remember it
**"Procedure = a saved multi-step action. Function = a saved single-value formula. Trigger = an automatic reaction. Event = a schedule."**

### Interview angle
- When would you use a trigger instead of application-level code?
- What's the risk of putting business logic in the database layer?

### Production angle
Many teams intentionally minimize triggers/procedures because they're harder to version-control, test, and observe compared to application code — but they're valuable for guarantees that must hold no matter what application touches the database (e.g., audit logging).

**Practice before moving on:** Write a trigger that automatically creates a `reviews` placeholder row set to NULL rating whenever an order's status changes to `completed`.

---

# MODULE 14 — Database Design & Normalization 🟡🔴

### What is it?
Database design is the process of turning real-world requirements into tables, columns, keys, and relationships — done *before* writing much SQL.

### The process
1. **Requirements analysis** — what does the system need to track?
2. **Entities** — the "nouns" (Customer, Order, Service)
3. **Attributes** — properties of each entity
4. **Relationships & cardinality** — how entities connect, and how many of each side
5. **Keys** — pick primary keys, define foreign keys
6. **Normalize** — remove duplication and update anomalies

### Normalization forms
- **1NF**: every column holds a single, atomic value (no comma-separated lists in one cell)
- **2NF**: 1NF + every non-key column depends on the *whole* primary key (matters for composite keys)
- **3NF**: 2NF + no non-key column depends on another non-key column (no "transitive" dependency)
- **BCNF**: a stricter version of 3NF for edge cases with multiple overlapping candidate keys

### Many-to-many & junction tables
Customers can favorite multiple categories; categories have multiple customers who favorite them → needs a junction table:
```sql
CREATE TABLE customer_favorite_categories (
    customer_id  INT NOT NULL,
    category_id  INT NOT NULL,
    PRIMARY KEY (customer_id, category_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);
```

### Denormalization
Sometimes you deliberately duplicate data (e.g., storing `provider_name` directly on `orders` as a snapshot) to avoid expensive JOINs on read-heavy systems, or to preserve historical accuracy (a provider's name might change later, but the order should show what it was called *at the time*).

### Practical production columns
```sql
ALTER TABLE orders
    ADD COLUMN created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ADD COLUMN updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    ADD COLUMN deleted_at DATETIME NULL;  -- soft delete: NULL = active, timestamp = "deleted"
```

### UUID vs AUTO_INCREMENT (comparison)
| | AUTO_INCREMENT | UUID |
|---|---|---|
| Size/speed | Small integer, fast index | Larger, slightly slower index |
| Predictable? | Yes (sequential — can leak info like order volume) | No, effectively random |
| Good for | Single-database systems | Distributed systems, merging data from multiple sources without collisions |

### Common mistakes
- Storing a list of values in one column (e.g., `"tag1,tag2,tag3"`) instead of a separate table — breaks 1NF and makes querying painful.
- Skipping normalization entirely, causing update anomalies (change a provider's name in one row but not others).
- Over-normalizing to the point every query needs 8 JOINs, hurting both readability and performance — real systems balance strict normalization with practical denormalization.

### How to remember it
**"1NF: no lists in a cell. 2NF: everything depends on the WHOLE key. 3NF: nothing depends on another non-key column."**

### Interview angle
- Design the tables for a many-to-many relationship between two entities.
- What's a real example of a 2NF violation?
- When would you deliberately denormalize a production schema?

### Production angle
Soft deletes, audit timestamps, and status fields (rather than physically deleting rows) are close to universal in production systems — they preserve history and let you undo mistakes.

**Practice before moving on (design exercise):** Design a schema for a "service marketplace" from scratch, on paper, *before* looking at ours: list every table, its columns, its primary key, and its foreign keys. Then compare to the schema at the top of this document.


---

# MODULE 15 — Performance Tuning ⚫

### What is it?
The practical process of diagnosing *why* a query is slow and fixing it — building on indexing (Module 11).

### Diagnostic workflow
1. Run `EXPLAIN` (or `EXPLAIN ANALYZE`) on the slow query.
2. Look for `type: ALL` (full scan), `Using filesort`, `Using temporary`.
3. Check if a relevant index exists; check the leftmost-prefix rule if it's composite.
4. Check if the query wraps an indexed column in a function.
5. Check row counts involved — sometimes the "slow" query is legitimately touching millions of rows and needs pagination or pre-aggregation instead of a quick index fix.
6. Rewrite, re-run EXPLAIN, compare.

### Example: diagnose and fix
```sql
-- SLOW: full scan, filesort
SELECT * FROM orders
WHERE YEAR(created_at) = 2026
ORDER BY created_at DESC;

-- FIX 1: avoid wrapping the column in a function
-- FIX 2: add a supporting index
CREATE INDEX idx_orders_created ON orders(created_at);

SELECT order_id, customer_id, status, created_at FROM orders
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
ORDER BY created_at DESC;
```

### Large UPDATE/DELETE operations
Doing a single massive UPDATE/DELETE on millions of rows locks a lot of the table for a long time. Production approach: batch it.
```sql
-- Batch delete in chunks of 1000 to avoid long locks
DELETE FROM orders WHERE status = 'cancelled' AND created_at < '2024-01-01' LIMIT 1000;
-- repeat until 0 rows affected
```

### Common mistakes
- Optimizing a query in isolation without checking if the underlying data volume itself is the real problem (sometimes the fix is archiving old data, not indexing).
- Adding an index but never confirming with EXPLAIN that it's actually being used.
- Running large single-transaction UPDATE/DELETE statements on live production tables during peak hours.

### How to remember it
**"Diagnose with EXPLAIN before you guess at a fix."**

### Interview angle
- Walk through how you'd diagnose a query that suddenly became slow.
- Why is a single giant DELETE risky in production, and what's the alternative?

### Production angle
This module *is* the day-to-day job for anyone running a production MySQL database — being systematic (EXPLAIN first, guesswork never) is what separates a junior from a senior here.

**Practice before moving on:** Take the pagination query from Module 11 and the aggregation query from Module 4 — write out, in words, what you'd check first if each one suddenly took 10 seconds instead of 100ms.

---

# MODULE 16 — MySQL Administration & Architecture ⚫

### Storage engines
InnoDB is the default and standard choice — supports transactions, row-level locking, foreign keys. (MyISAM is legacy — no transactions, table-level locking; avoid for new projects.)

### Key architecture pieces
- **Buffer pool**: InnoDB's in-memory cache of table/index data — the single biggest lever for read performance; bigger buffer pool = more data served from memory instead of disk.
- **Connections**: each client connection uses server resources; too many idle connections can exhaust `max_connections`.
- **Slow query log**: logs queries exceeding a configured time threshold — the first place to look for optimization targets.
- **Binary log (binlog)**: records every data-changing statement — used for replication and point-in-time recovery.

### Users, roles, privileges
```sql
CREATE USER 'app_reader'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT ON marketplace.* TO 'app_reader'@'%';

CREATE USER 'app_writer'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON marketplace.* TO 'app_writer'@'%';

REVOKE DELETE ON marketplace.* FROM 'app_writer'@'%';
FLUSH PRIVILEGES;
```
**Principle of least privilege**: give each application/user only the permissions it actually needs — a reporting service should never have DELETE rights.

### Replication & high availability
- **Replication**: a primary server streams its binlog to one or more **read replicas**, which apply the same changes.
- **Read replicas**: offload read traffic (reports, dashboards) from the primary, which handles writes.
- **Failover**: if the primary fails, a replica is promoted — critical for uptime.
- **Replication lag**: the delay between a write on the primary and its appearance on a replica — an app reading from a lagging replica right after a write can see stale data.

### Monitoring basics
Watch: CPU, memory, disk I/O and free space, active connections vs `max_connections`, replication lag, and lock waits/deadlock count. Tools: `SHOW PROCESSLIST;`, `SHOW ENGINE INNODB STATUS;`, `performance_schema` tables.

### Common mistakes
- Running the application with a root/superuser database account instead of a scoped one.
- No monitoring on replication lag — leads to silently serving stale data from replicas.
- Not sizing the buffer pool appropriately for the dataset (too small = constant disk reads).

### How to remember it
**"Least privilege for users. Buffer pool for speed. Replicas for reads and safety."**

### Interview angle
- What's the difference between InnoDB and MyISAM, and why does it matter?
- Explain replication lag and its practical impact on an application.
- Why should application database users follow least privilege?

### Production angle
This is core DevOps/DBA territory — knowing *what* to check (not necessarily running every command yourself) is what lets a backend engineer speak the same language as infrastructure teams during an incident.

**Practice before moving on:** Design the privilege scheme for three service accounts: a reporting dashboard (read-only), the main application (read/write, no DROP/TRUNCATE), and a migration tool (full DDL access, used rarely).

---

# MODULE 17 — Backup, Recovery & Incident Response ⚫

### Backups
```bash
# Full logical backup
mysqldump -u root -p marketplace > marketplace_backup.sql

# Single table
mysqldump -u root -p marketplace orders > orders_backup.sql

# Restore
mysql -u root -p marketplace < marketplace_backup.sql
```
**What it does:** `mysqldump` produces a text file of SQL statements that can recreate the schema and data. **Why:** simple, portable, human-readable. **What can go wrong:** locking on large tables during dump (use `--single-transaction` for InnoDB to avoid long locks), and restore time can be slow for huge databases. **How to verify:** restore to a *separate* test database and run row-count/spot checks before trusting it.

### Point-in-time recovery
Restore the last full backup, then replay binary logs from that point up to just before the incident (e.g., right before an accidental mass DELETE) — this is why binary logging matters even outside of replication.

### Disaster recovery scenarios (walkthroughs)
- **Accidental DROP TABLE**: restore from backup + binlog replay up to the moment before the DROP.
- **Deadlock**: MySQL auto-kills one transaction; the application should catch the deadlock error and retry the transaction.
- **Missing index causing timeout**: identify via slow query log/EXPLAIN, add index, verify with EXPLAIN again.
- **Connection exhaustion**: check `SHOW PROCESSLIST`, identify long-idle or runaway connections, fix connection pooling on the application side, consider raising `max_connections` as a stopgap only.
- **Disk full**: often caused by binlogs or unbounded table growth — check `SHOW BINARY LOGS`, purge old logs, check for old unarchived data.
- **Replication lag spike**: check for a single huge write on the primary (large batch job) blocking replica apply threads; consider throttling large writes.

### Common mistakes
- Never testing a restore — a backup you've never restored is not a verified backup.
- Keeping backups on the same server/disk as the live database (single point of failure).
- No retention policy — either running out of disk space from too many, or not having enough history for recovery needs.

### How to remember it
**"A backup is only as good as its last successful restore test."**

### Interview angle
- Walk through recovering data after an accidental production DROP TABLE.
- What's point-in-time recovery, and why do binary logs matter for it?
- How would you diagnose sudden connection exhaustion?

### Production angle
This module is what "on-call" actually looks like — the goal isn't memorizing every command, but having a calm, repeatable process: identify → contain → fix → verify → document.

**Practice before moving on:** Write out, step-by-step, your full recovery plan if someone ran `DELETE FROM payments;` with no WHERE clause in production five minutes ago.


---

# MODULE 18 — MySQL with Application Code ⚫

### The flow: Application → SQL → Database
```
User clicks "Book Service"
   → App code builds a query/ORM call
   → Connection pool hands out a DB connection
   → SQL runs inside a transaction
   → Result returned to app
   → Connection returned to the pool
```

### ORM vs raw SQL (comparison)
| | ORM (e.g., Sequelize, Django ORM, Eloquent) | Raw SQL |
|---|---|---|
| Speed of development | Faster for simple CRUD | Slower to write, but fully explicit |
| Performance control | Can hide inefficient queries (see N+1 below) | Full control |
| Best for | Standard CRUD, rapid development | Complex reports, performance-critical paths |

### The N+1 query problem
```
-- ORM naive pattern: 1 query for orders, then 1 query PER order for its customer
SELECT * FROM orders;                 -- 1 query
SELECT * FROM customers WHERE id = 1; -- N queries, one per order!
SELECT * FROM customers WHERE id = 2;
...
```
**Fix:** eager-load / JOIN, or a single `WHERE customer_id IN (...)` batch query, instead of one query per row.

### Migrations
Version-controlled, incremental schema changes (e.g., `add_status_column_to_orders.sql`) applied in order across environments — this is how teams keep dev, staging, and production schemas in sync safely.

### Connection pooling & caching
A pool of reusable database connections avoids the overhead of opening/closing a TCP+auth handshake per request. Caching (e.g., Redis) sits in front of frequent, expensive, rarely-changing reads (like a category list) to avoid hitting MySQL at all for every request.

### Common mistakes
- Letting an ORM silently generate N+1 queries — invisible until the app is under real load.
- No connection pool limit — the app opens unbounded connections under load, exhausting MySQL's `max_connections` (ties back to Module 17).
- Running unindexed queries generated automatically by an ORM without ever checking EXPLAIN.

### How to remember it
**"An ORM writes SQL for you — it doesn't remove the need to understand the SQL it wrote."**

### Interview angle
- What's the N+1 query problem, and how do you fix it?
- When would you bypass the ORM and write raw SQL?

### Production angle
Nearly every "why did our API get slow after we added a new feature" investigation eventually traces back to either N+1 queries or a missing index the ORM didn't know to suggest.

**Practice before moving on:** Given "list all orders with their customer's name," write both the N+1 naive version (in pseudocode) and the single-query JOIN fix.

---

# MODULE 19 — Interview Preparation (Sample Set) 🟡🔴⚫

*(Full bank is 110 questions across 4 levels — sample set below; ask me any time for more from a specific level/topic.)*

### Beginner (sample of 20)
1. What is the difference between a primary key and a unique key?
2. Write a query to find all services priced above $50.
3. What does `DISTINCT` do?
4. Difference between `CHAR` and `VARCHAR`?
5. What is a foreign key, and why is it useful?

### Intermediate (sample of 30)
6. Difference between `INNER JOIN` and `LEFT JOIN`, with an example.
7. Write a query to find customers with more than 5 orders.
8. What is a correlated subquery?
9. Explain `WHERE` vs `HAVING`.
10. Difference between `UNION` and `UNION ALL`.

### Advanced (sample of 30)
11. Write a query using a window function to find the top 3 highest-paid employees per department.
12. Explain a recursive CTE with a real example.
13. What causes a full table scan, and how do you detect it?
14. Explain the leftmost-prefix rule for composite indexes.
15. Write a query to detect duplicate emails in the `users` table.

### Senior/Production (sample of 30)
16. Walk through diagnosing a sudden production slowdown.
17. Explain isolation levels and which anomalies each one prevents.
18. How would you safely delete 10 million stale rows from a live production table?
19. Explain point-in-time recovery.
20. Design a least-privilege access scheme for three types of database users.

**How we'll use this:** In interactive sessions, I'll give you these one at a time (or in themed batches), have you answer first, then evaluate and correct.

---

# MODULE 20 — Final Projects & Mastery Test 🔴⚫

### Project 1 — Employee Management Database
Requirements: track employees, departments, managers (hierarchy), salaries, and salary history over time. *(Design it yourself first — tables, PKs, FKs — before checking against a reference design.)*

### Project 2 — E-commerce Database
Requirements: products, categories, customers, orders, order line items (an order can contain multiple products — this needs its own junction-style table), payments, and inventory tracking.

### Project 3 — Booking System
Requirements: resources (rooms/equipment), time slots, customers, bookings that must never overlap for the same resource (a real-world constraint problem, not just a schema problem).

### Project 4 — Subscription System
Requirements: plans, customers, subscription periods, renewals, cancellations, and prorated billing history.

### Project 5 — Capstone: Full Service Marketplace
Requirements: everything in this course's running schema, PLUS: multi-category services, reviews with moderation status, provider payouts, and a reporting layer (views/CTEs) for admin dashboards.

### Final Mastery Test — 10 Parts
1. Theory (definitions, comparisons)
2. Syntax recall (write from memory, no lookup)
3. SQL writing (given a requirement in English, write the query)
4. Query debugging (given broken SQL, fix it)
5. Database design (given requirements, design the schema)
6. Advanced SQL (window functions, recursive CTEs)
7. Index optimization (given a slow query + EXPLAIN output, fix it)
8. Transactions (design a multi-step transaction with rollback logic)
9. Production troubleshooting (given symptoms, diagnose and respond)
10. Real-world case study (design + query + optimize + troubleshoot, combined)

**How we'll use this:** When you're ready, tell me and I'll run each part as an interactive test — you attempt it, then I evaluate and show the reference solution with explanation.

---

# Quick-Reference: Comparisons Cheat Sheet

| Comparison | Key distinction |
|---|---|
| DELETE vs TRUNCATE vs DROP | Filtered rows vs all rows (fast) vs whole table gone |
| WHERE vs HAVING | Before grouping vs after grouping |
| UNION vs UNION ALL | Removes duplicates vs keeps everything (faster) |
| INNER vs LEFT JOIN | Only matches vs all-left-plus-matches |
| PRIMARY KEY vs UNIQUE KEY | One per table, no NULLs vs multiple allowed, one NULL allowed |
| CHAR vs VARCHAR | Fixed-length, padded vs variable-length |
| DATETIME vs TIMESTAMP | No timezone conversion, wider range vs timezone-aware, narrower range |
| IN vs EXISTS | List membership vs existence check (EXISTS often faster for correlated checks) |
| JOIN vs subquery | Combines columns from both tables vs answers an intermediate question |
| CTE vs subquery | Named & reusable, can recurse vs inline, single-use |
| RANK vs DENSE_RANK vs ROW_NUMBER | Skips on ties vs no skipping vs no ties at all |
| VIEW vs TABLE | Virtual/live query vs actual stored data |
| PROCEDURE vs FUNCTION | Multi-step action vs single returnable value usable in SELECT |
| READ COMMITTED vs REPEATABLE READ | Allows non-repeatable reads vs prevents them (MySQL default) |

---

# What to Do Next

This document gives you the *what and why* for the entire course. The parts that actually build long-term memory — practice problems, mistake correction, spaced repetition, active recall — only work interactively. Whenever you're ready, tell me things like:

- "Quiz me on Module 5 (JOINs)"
- "Give me 5 practice problems combining GROUP BY and JOIN"
- "Test me on Module 11 with an EXPLAIN scenario"
- "Start the Module 20 capstone project — give me requirements only"

I'll track which topics you're solid on and which need revisiting, and keep mixing old concepts into new practice the way the original plan called for.
