

### Part 1: The Art of the Subquery and CTE (Thinking in Sets)

Novices write queries one line at a time. Masters write queries *inside* queries.

#### Subqueries (Nested SELECTs)
A subquery is a `SELECT` statement embedded within a clause of another SQL statement. It allows you to use the result of one query as input for another.

**Example: Find all employees who earn more than the *average* salary in the company.**

```sql
-- Without a subquery, you'd need two round trips to the DB or a local variable.
-- With a subquery, it's one atomic thought.
SELECT name, salary
FROM Employees
WHERE salary > (SELECT AVG(salary) FROM Employees);
```
**Key Types:**
- **Scalar:** Returns single value (`=`, `>`, `<`).
- **Row:** Returns single row (`= (col1, col2)`).
- **Table:** Returns a result set (used in `FROM` or `IN`).

**Correlated Subquery (The Dangerous Beauty):**
A subquery that references the *outer* query. Executed **once per row**.
*Use Case:* "Find employees whose salary is greater than the average salary of *their own department*."

```sql
SELECT e1.name, e1.salary, e1.dept_id
FROM Employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM Employees e2
    WHERE e2.dept_id = e1.dept_id  -- The Correlation: ties inner to outer
);
```
*10^10 IQ Warning:* Correlated subqueries can be **performance killers** on large datasets (O(n²) complexity). Often, a **Window Function** or **CTE** is a better choice.

#### Common Table Expressions (CTEs) - The `WITH` Clause
CTEs are **named temporary result sets**. They exist only for the duration of the query. They make complex logic **readable** and **maintainable**.

**Example: The same "Above Dept Average" query, but readable.**

```sql
WITH DeptAverages AS (
    -- This CTE calculates the average per department once.
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM Employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, e.dept_id, d.avg_sal
FROM Employees e
JOIN DeptAverages d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;
```
**Advantage:** The DBMS optimizes `DeptAverages` once and reuses it. The intent is crystal clear.

#### Recursive CTEs (Walking the Graph)
**This is the single most powerful feature in standard SQL for handling hierarchical data** (Org Charts, Bill of Materials, Threaded Comments).
Without it, you need a `WHILE` loop in your application code. With it, the database engine traverses the tree natively.

**Example: Employee Hierarchy (Who reports to whom?)**
```sql
WITH RECURSIVE OrgChart AS (
    -- Anchor Member: The starting point (The CEO)
    SELECT id, name, manager_id, 1 AS level
    FROM Employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive Member: Join the CTE to the table to find children
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM Employees e
    INNER JOIN OrgChart o ON e.manager_id = o.id
)
SELECT * FROM OrgChart ORDER BY level, name;
```
*Result:* You get the entire reporting structure in one query. No loops. No N+1 query problem.

---

### Part 2: Window Functions - The End of the `GROUP BY` Tyranny

`GROUP BY` collapses rows into aggregates. Window Functions **do not collapse rows**. They perform calculations *across a set of rows related to the current row* while leaving the individual row intact.

Think of it as adding a "calculated column" that sees the whole table or a specific window.

**Core Components:**
- `OVER()`: Defines the window.
- `PARTITION BY`: Like a `GROUP BY` but for the window calculation only.
- `ORDER BY`: Defines the order for ranking or running totals.

#### The Holy Trinity of Ranking
**Scenario:** You need to give a Gold, Silver, Bronze medal to the top 3 salespeople per region.

```sql
SELECT
    name,
    region,
    total_sales,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_sales DESC) AS rank_strict,
    RANK() OVER (PARTITION BY region ORDER BY total_sales DESC) AS rank_with_gaps,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY total_sales DESC) AS rank_no_gaps
FROM SalesPeople;
```
| Name | Region | Sales | ROW_NUMBER | RANK | DENSE_RANK |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Alice | West | 100 | 1 | 1 | 1 |
| Bob | West | 100 | 2 | 1 | 1 |
| Carol | West | 90 | 3 | 3 | 2 |

*10^10 IQ Distinction:*
- `ROW_NUMBER`: Unique sequential number (ties broken arbitrarily unless specified).
- `RANK`: Ties get same rank, next rank skips (1, 1, 3).
- `DENSE_RANK`: Ties get same rank, next rank is consecutive (1, 1, 2).

#### LAG and LEAD (Time Travel in a Row)
Need to compare a value to *yesterday's* value without a self-join? Use `LAG`.
**Scenario:** Calculate day-over-day change in stock price.

```sql
SELECT
    date,
    price,
    LAG(price, 1) OVER (ORDER BY date) AS prev_price,
    price - LAG(price, 1) OVER (ORDER BY date) AS change
FROM StockPrices
ORDER BY date;
```

#### Running Totals and Moving Averages
**Scenario:** Show the cumulative number of signups per day.

```sql
SELECT
    signup_date,
    COUNT(*) AS daily_signups,
    SUM(COUNT(*)) OVER (ORDER BY signup_date) AS cumulative_total
FROM Users
GROUP BY signup_date
ORDER BY signup_date;
```
*Note the interplay:* We aggregate with `GROUP BY`, then apply the window function to the *aggregated result*.

---

### Part 3: Transactions and Concurrency - The Acid Test Revisited

In the "Basics" we defined ACID. Now we manage it. When you have 10,000 users hitting the "Buy" button for the last PS5 in stock, how does the DBMS keep its sanity?

#### Isolation Levels (The Trade-Off)
You control the balance between **Speed (Concurrency)** and **Correctness (Isolation)** .

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance | Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | Yes | Yes | Yes | Fastest | *Almost never used. Only for rough estimates.* |
| **Read Committed** | No | Yes | Yes | Good | **PostgreSQL/MySQL Default.** Good for most web apps. |
| **Repeatable Read** | No | No | Yes | Slower | Financial reports that must see a static snapshot. |
| **Serializable** | No | No | No | Slowest | Bank transfers. Absolute consistency. |

**Example Problem: The Inventory Race Condition**
Two users click "Buy" simultaneously on Item ID 99 (Qty: 1).
- **Read Committed:** User A reads Qty=1. User B reads Qty=1. Both deduct. Both think they bought it. **You sold the same item twice.**
- **Serializable:** The DBMS detects the conflict and rolls back User B. User B sees "Sorry, out of stock."

**Code Snippet: Explicit Locking (The Heavy Hand)**
Sometimes you need to be sure.
```sql
BEGIN;
-- Lock the row for update. No one else can touch it until we commit.
SELECT quantity FROM Inventory WHERE product_id = 99 FOR UPDATE;
-- Check if quantity >= 1 in application code...
UPDATE Inventory SET quantity = quantity - 1 WHERE product_id = 99;
COMMIT;
```

---

### Part 4: Performance and Optimization (Thinking Like the Engine)

You can write perfect SQL that takes 3 hours to run. Advanced SQL means understanding the **Query Planner**.

#### The `EXPLAIN` Command
Before deploying a query, run `EXPLAIN ANALYZE`. This shows you the **Execution Plan**.
- **Seq Scan:** Reading every single row. (Bad for large tables, good for tiny tables).
- **Index Scan:** Using the phone book to find a name. (Good).
- **Nested Loop:** For each row in A, check B. (Good for small sets).
- **Hash Join:** Build a hash table of A, probe with B. (Good for large sets).

**The 10^10 IQ Optimization Mantra:**
**"Seek, don't Scan."**
If you see `Seq Scan` on a 10-million-row table for a query returning 1 row, **create an index**.

#### Advanced Indexing Strategies
```sql
-- 1. Partial Index (Index only rows you care about)
-- Only index orders that are 'ACTIVE' (which is 1% of the table)
CREATE INDEX idx_active_orders ON Orders (customer_id) WHERE status = 'ACTIVE';

-- 2. Covering Index (Include extra columns to avoid hitting the main table)
-- Allows an "Index Only Scan"
CREATE INDEX idx_user_email_name ON Users (email) INCLUDE (name, avatar_url);

-- 3. GIN Index for Full-Text Search or JSONB
-- "Find all products where description contains 'wireless'"
CREATE INDEX idx_product_desc ON Products USING GIN (to_tsvector('english', description));
```

#### Views vs. Materialized Views
- **View:** A saved **query**. Every time you `SELECT * FROM view`, it runs the underlying query. **Always fresh, sometimes slow.**
- **Materialized View:** A saved **result set** (a physical table). You must `REFRESH MATERIALIZED VIEW` to update it. **Potentially stale, blazingly fast.** Use this for complex dashboards and end-of-day reports.

```sql
CREATE MATERIALIZED VIEW monthly_sales_report AS
SELECT ... (complex join of 5 tables) ...

-- Run this via cron job every night
REFRESH MATERIALIZED VIEW monthly_sales_report;
```

### Part 5: Stored Procedures and Functions (Moving Logic to the Data)

For decades, developers argued: "Where should business logic live? App Server or Database?"
*10^10 IQ Answer:* **Where the data is, when the operation is data-intensive.**

If you need to loop through a million rows to apply interest to savings accounts, **do it inside the database**. Moving a million rows over the network to the App Server just to add 1% interest and send them back is madness.

**Example: Function to Calculate Compound Interest (PL/pgSQL)**
```sql
CREATE OR REPLACE FUNCTION apply_interest(rate DECIMAL)
RETURNS VOID AS $$
BEGIN
    -- This updates millions of rows instantly using set logic, not loops
    UPDATE Savings_Accounts
    SET balance = balance * (1 + rate)
    WHERE balance > 0;

    -- Optional: Log the action
    INSERT INTO Audit_Log (action, timestamp) VALUES ('Interest Applied', NOW());
END;
$$ LANGUAGE plpgsql;

-- Execute it
SELECT apply_interest(0.03);
```

### Summary: The Senior Engineer's Advanced Toolbox

| Problem | Advanced SQL Solution | Why It's Superior |
| :--- | :--- | :--- |
| Hierarchical Data (Org Chart) | **Recursive CTE** | Replaces hundreds of round-trip queries. |
| "Top 3 per Group" | **Window Function (ROW_NUMBER)** | Replaces complex self-joins and temporary tables. |
| Comparing Row to Previous Row | **LAG / LEAD** | Avoids slow correlated subqueries. |
| Complex Multi-Step Logic | **CTE (WITH Clause)** | Makes code readable and optimizer-friendly. |
| Long-Running Dashboard Queries | **Materialized View** | Trading slight staleness for millisecond response times. |
| Data-Heavy Batch Processing | **Stored Procedure** | Minimizes network traffic and leverages server power. |

Master these, and you cease to be a developer who merely *uses* a database. You become a **Data Engineer** who orchestrates the flow of information with precision and elegance.