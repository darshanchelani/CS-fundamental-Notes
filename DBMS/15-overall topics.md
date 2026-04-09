

### Part 1: Joins – Stitching the Universe Together

A **JOIN** combines rows from two or more tables based on a related column. The type determines what happens when a match is **not** found.

#### 1. INNER JOIN (The Intersection)
Returns rows **only when there is a match in both tables**.

**Example: Customers who have placed orders.**
```sql
SELECT c.name, o.order_id
FROM Customers c
INNER JOIN Orders o ON c.cust_id = o.cust_id;
```
*10^10 IQ Insight:* This is the **most common** join. The optimizer can reorder tables freely because only matching rows survive.

#### 2. LEFT JOIN (Preserve the Left)
Returns **all rows** from the left table, plus matching rows from the right. If no match, right side columns are **NULL**.

**Example: All customers, with order IDs if they have any.**
```sql
SELECT c.name, o.order_id
FROM Customers c
LEFT JOIN Orders o ON c.cust_id = o.cust_id;
-- Grace Hopper (never ordered) appears with order_id = NULL
```

#### 3. RIGHT JOIN (Preserve the Right)
Returns all rows from the right table. **Rarely used** because you can always swap table order and use `LEFT JOIN`. Equivalent to `LEFT JOIN` with reversed table positions.

#### 4. SELF JOIN (Table Joins Itself)
Used to compare rows **within the same table**. Requires table aliases.

**Example: Find employees and their managers (both in `Employees` table).**
```sql
SELECT e.name AS employee, m.name AS manager
FROM Employees e
LEFT JOIN Employees m ON e.manager_id = m.emp_id;
```
*Note:* `LEFT JOIN` ensures the CEO (who has no manager) still appears.

---

### Part 2: Subqueries – Nested Logic

A **subquery** is a query inside another query. They can appear in `SELECT`, `FROM`, or `WHERE`.

| Type | Location | Execution |
| :--- | :--- | :--- |
| **Scalar** | `SELECT` / `WHERE` | Returns single value. |
| **Row Subquery** | `WHERE` | Returns single row. |
| **Table Subquery** | `FROM` / `IN` | Returns result set. |
| **Correlated** | `WHERE` | References outer query; runs **once per outer row**. |

**Example: Find employees earning above department average (Correlated).**
```sql
SELECT name, salary, dept_id
FROM Employees e1
WHERE salary > (SELECT AVG(salary) FROM Employees e2 WHERE e2.dept_id = e1.dept_id);
```
*10^10 IQ Warning:* Correlated subqueries can be **performance killers**. Often better to use a **Window Function** or **CTE**.

**Modern Alternative (CTE):**
```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM Employees GROUP BY dept_id
)
SELECT e.name, e.salary
FROM Employees e
JOIN dept_avg d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;
```

---

### Part 3: Window Functions – Analytics Without Collapsing

Window functions perform calculations across a set of rows **without grouping them into a single output row**. They are the swiss army knife for ranking, running totals, and time-series comparisons.

**Core Syntax:**
```sql
<function>() OVER (
    [PARTITION BY col1, col2]  -- Like GROUP BY, but rows remain separate
    [ORDER BY col3]            -- Defines order for rank/lead/lag
    [ROWS/RANGE frame]         -- Defines sliding window
)
```

**Essential Functions:**

| Function | Purpose | Example |
| :--- | :--- | :--- |
| `ROW_NUMBER()` | Unique sequential number per partition | Rank students by score (no ties) |
| `RANK()` | Rank with gaps for ties | 1, 2, 2, 4 |
| `DENSE_RANK()` | Rank without gaps | 1, 2, 2, 3 |
| `LAG(expr, offset)` | Value from previous row | Day-over-day change |
| `LEAD(expr, offset)` | Value from next row | Forecast comparison |
| `SUM() OVER (ORDER BY ...)` | Running total | Cumulative sales |

**Example: Running total of sales per customer.**
```sql
SELECT
    order_id,
    customer_id,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total
FROM Orders;
```

---

### Part 4: Complex Queries – Combining the Arsenal

Complex queries combine `JOINs`, subqueries, `CASE`, `GROUP BY`, `HAVING`, and window functions. The key is **readability** and **optimizer-friendliness**.

**Strategy: Use Common Table Expressions (CTEs) to build modular steps.**

**Example: Top 3 products per category based on revenue.**
```sql
WITH ProductRevenue AS (
    SELECT
        p.category_id,
        p.product_id,
        p.name,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM Products p
    JOIN Order_Items oi ON p.product_id = oi.product_id
    GROUP BY p.category_id, p.product_id, p.name
),
RankedProducts AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY revenue DESC) AS rank
    FROM ProductRevenue
)
SELECT category_id, product_id, name, revenue
FROM RankedProducts
WHERE rank <= 3
ORDER BY category_id, rank;
```
*10^10 IQ Tip:* Always filter **early** (in CTEs or subqueries) to reduce the data volume before expensive operations like window functions or joins.

---

### Part 5: Sharding vs. Replication – Scaling Dimensions

These are the two fundamental axes of distributed database scaling.

| Aspect | **Replication** | **Sharding (Partitioning)** |
| :--- | :--- | :--- |
| **Goal** | **Availability & Read Scalability** | **Write Scalability & Data Volume** |
| **Mechanism** | Copy the **same data** to multiple nodes. | Split **different data** across multiple nodes. |
| **Analogy** | Having multiple copies of the same book in different libraries. | Splitting a huge encyclopedia into volumes, each stored in a different library. |
| **Write Path** | Writes go to one **Leader** (or conflict resolution). | Writes go to the **specific shard** owning the key. |
| **Query Impact** | Reads can be served by any replica (stale possible). | Queries for a single key are fast; cross-shard joins are **expensive**. |
| **Complexity** | Low to Moderate (failover handling). | **High** (choice of shard key, resharding, distributed transactions). |

**The 10^10 IQ Rule:**
- **Start with Replication.** Most apps never outgrow a single write master with read replicas.
- **Shard only when write throughput or total storage exceeds a single machine's capacity.** And when you do, **choose the shard key based on the most frequent access pattern** (e.g., `tenant_id` for SaaS).

---

### Part 6: Caching (Redis Basics) – The Speed of Memory

**Redis** is an in-memory **data structure server**. It is the go-to solution for reducing database load by storing frequently accessed data in RAM.

**Why use Redis?**
- **Latency:** Redis responds in **sub-millisecond** time. PostgreSQL/MySQL takes **milliseconds to tens of milliseconds**.
- **Throughput:** Handles **100,000+ operations per second** on modest hardware.

**Core Data Types & Patterns:**

| Type | Use Case | Command Example |
| :--- | :--- | :--- |
| **String** | Caching HTML fragments, serialized objects, counters. | `SET user:1:profile "{...}" EX 3600` |
| **Hash** | Storing object fields (user attributes). | `HSET user:1 name "Ada" email "ada@example.com"` |
| **List** | Job queues, activity feeds. | `LPUSH queue:emails "welcome:user1"` |
| **Set** | Tags, unique visitors. | `SADD article:1:tags "redis" "database"` |
| **Sorted Set** | Leaderboards, rate limiting. | `ZADD leaderboard 1000 "player1"` |

**Caching Strategies (Patterns):**

1.  **Cache-Aside (Lazy Loading):** Application checks cache; if miss, loads from DB and writes to cache.
    ```python
    data = redis.get(key)
    if not data:
        data = db.query(...)
        redis.setex(key, 3600, data)
    return data
    ```
2.  **Write-Through:** Write to cache first, then to DB synchronously. (Slower writes, consistent cache).
3.  **Write-Behind:** Write to cache, acknowledge immediately, async flush to DB. (Fast, risk of data loss).

**The 10^10 IQ Redis Setup:**
- Always set **TTL (Expiration)** to prevent memory exhaustion.
- Use **Connection Pooling**.
- For high availability, use **Redis Sentinel** or **Redis Cluster**.

---

### Part 7: Read vs. Write Optimization – The Balancing Act

Optimizing for reads and writes often requires **opposite** strategies.

| Goal | Write-Optimized Strategies | Read-Optimized Strategies |
| :--- | :--- | :--- |
| **Indexes** | Minimize indexes (each index slows writes). | Create **covering indexes**, partial indexes for critical queries. |
| **Normalization** | Favor normalization (less duplication to update). | **Denormalize** (summary tables, materialized views). |
| **Data Structure** | Append-only logs, LSM Trees (Cassandra, RocksDB). | B-Trees with caching (PostgreSQL, MySQL). |
| **Caching** | Use **Write-Through/Behind** caches. | Aggressive use of **Read-Through** caches (Redis). |
| **Concurrency** | **Optimistic Locking** (version columns). | **MVCC** (readers don't block writers). |
| **Hardware** | Fast **sequential write** disks (SSD with high DWPD). | Large **RAM** for caching (Buffer Pool). |

**Practical Example:**
- **E-commerce Product Page:** Read-heavy. Use **materialized view** of product details + reviews count. Cache in Redis with TTL 5 minutes.
- **Analytics Event Ingestion:** Write-heavy. Use **Kafka** -> **ClickHouse** or **TimescaleDB** with minimal indexes during ingest, batch aggregate later.

---

### Part 8: Index Types (B-Tree vs. Hash) – The Access Paths

The query optimizer chooses the index. Understanding the differences prevents mis-design.

| Feature | **B-Tree (Default)** | **Hash Index** |
| :--- | :--- | :--- |
| **Supported Operations** | `=`, `>`, `<`, `>=`, `<=`, `BETWEEN`, `LIKE 'prefix%'`, `ORDER BY` | **Only** `=` (equality) |
| **Storage** | Sorted keys; leaf nodes linked for range scans. | Buckets with hash of key. |
| **Use Case** | General purpose, range queries, sorting. | Very fast point lookups for exact keys (e.g., `WHERE session_id = 'abc'`). |
| **Limitations** | Slightly slower for single `=` than hash. | Cannot handle range queries. Not WAL-logged in PostgreSQL (must rebuild after crash). |
| **Multi-Column** | **Composite keys** work (leftmost prefix). | Hash of combined columns, only works with exact match on **all** columns. |

**Example: Choosing Index Type**
```sql
-- Use B-Tree (Default)
CREATE INDEX idx_orders_date ON Orders (order_date);  -- range queries work

-- Use Hash (PostgreSQL)
CREATE INDEX idx_sessions_token ON UserSessions USING HASH (session_token);
-- Query: SELECT * FROM UserSessions WHERE session_token = 'xyz'; (Fast O(1))
```

*10^10 IQ Rule:* **Stick with B-Tree unless you have a very specific, high-volume equality lookup and no range needs.** B-Tree's versatility almost always outweighs the marginal speed gain of Hash.

---

### Part 9: Isolation Levels (with Examples) – The Consistency Slider

Isolation levels control the visibility of uncommitted changes. They trade off **correctness** vs. **concurrency**.

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Example Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | Yes | Yes | Yes | *Never use in production.* |
| **Read Committed** | No | Yes | Yes | **PostgreSQL Default.** T1 sees only committed data. Within T1, two reads of same row may differ if T2 committed. |
| **Repeatable Read** | No | No | Yes (in SQL Std) / **No in PG** | T1 sees a **snapshot** as of transaction start. Rows are stable. But new rows matching a condition can appear (phantoms). PG prevents phantoms too. |
| **Serializable** | No | No | No | Transactions appear to run **one after another**. Highest safety, highest abort rate. |

**Concrete Example (PostgreSQL):**

```sql
-- Session 1: Repeatable Read
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM Accounts WHERE id = 1; -- returns 100

-- Session 2:
BEGIN;
UPDATE Accounts SET balance = 200 WHERE id = 1;
COMMIT;

-- Session 1 again:
SELECT balance FROM Accounts WHERE id = 1; -- still returns 100 (snapshot isolation)
UPDATE Accounts SET balance = balance + 50 WHERE id = 1; -- uses old value (100+50=150)
COMMIT;
-- Result: Session 2's update is lost! (Write-Write conflict detection prevents this in Serializable)
```

*10^10 IQ Advice:* Use **Read Committed** for 90% of web apps. Use **Repeatable Read** for reports needing a consistent point-in-time view. Use **Serializable** only for financial transactions or critical inventory adjustments.

---

### Part 10: Deadlocks – The Embrace of Mutual Exclusion

A **deadlock** occurs when two transactions are each waiting for a lock held by the other.

**Classic Example:**
- T1: Locks Account A, wants Account B.
- T2: Locks Account B, wants Account A.

**Detection & Resolution:**
The DBMS maintains a **Wait-For Graph**. When a cycle is detected, it **aborts one transaction** (the "victim") and rolls it back, returning an error. The application **must retry** the transaction.

**Prevention Strategies (10^10 IQ):**
1.  **Consistent Lock Ordering:** Always access resources in the same order (e.g., always lock Account 1 before Account 2).
2.  **Keep Transactions Short:** Long transactions increase lock hold time.
3.  **Use Lower Isolation Levels:** Fewer locks, less contention.
4.  **Index Foreign Keys:** Without an index on a FK, locking the parent may lock the entire child table.

**Code: Handling Deadlock in Application (Python)**
```python
import psycopg2
from psycopg2 import errors

def transfer_funds(from_id, to_id, amount):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                cur.execute("BEGIN;")
                cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", (amount, from_id))
                cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", (amount, to_id))
                cur.execute("COMMIT;")
            return
        except errors.DeadlockDetected:
            conn.rollback()
            time.sleep(0.1 * (2 ** attempt))  # exponential backoff
    raise Exception("Transfer failed after retries")
```

---

### Part 11: Query Execution Basics – The Journey of a SQL Statement

Understanding the lifecycle demystifies performance.

```text
[1. Parser]       -> Lexical & Syntax Analysis -> Parse Tree
[2. Rewriter]     -> Apply Rules (Views, Row-Level Security)
[3. Planner/Optimizer] -> Generate multiple execution plans, choose cheapest using statistics.
[4. Executor]     -> Execute plan (scan tables, join, aggregate, sort).
```

**Key Components of Execution Plan (from `EXPLAIN`):**

| Node Type | Meaning | When It's Good | When It's Bad |
| :--- | :--- | :--- | :--- |
| **Seq Scan** | Read entire table sequentially. | Small tables (< few MB). | Large tables for selective queries. |
| **Index Scan** | Traverse index, then fetch heap rows. | Highly selective queries. | Many rows returned (random I/O). |
| **Index Only Scan** | Index covers all needed columns; never touches heap. | **Excellent.** Covering indexes. | Rarely bad. |
| **Bitmap Heap Scan** | Build bitmap of row locations from index, sort, then read heap sequentially. | Moderate selectivity (5-20% of table). | Overhead of bitmap creation. |
| **Nested Loop Join** | For each outer row, probe inner table. | Small outer table and index on inner. | Large outer table. |
| **Hash Join** | Build hash table of smaller table, probe with larger. | Joining two large tables. | Memory exhaustion if hash table too big. |
| **Merge Join** | Both inputs sorted; walk through in parallel. | Both tables pre-sorted (indexes). | Sorting cost if not sorted. |

**How to Use `EXPLAIN (ANALYZE, BUFFERS)`**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```
Focus on:
- **`actual time`** vs `estimated rows` (stale statistics?).
- **`Buffers: shared hit`** (cache hits) vs **`read`** (disk I/O).
- **Nested Loop with high row count** → Missing index on join column.

---

### Summary: The Integrated Mental Model

These topics are not isolated trivia; they form a coherent system:

- **Joins & Subqueries** express the **what**.
- **Window Functions** add analytics without losing detail.
- **Sharding & Replication** define the **where** (data placement).
- **Caching** and **Read/Write Optimization** manage the **when** (latency).
- **Index Types** and **Isolation Levels** control the **how** (access paths and safety).
- **Deadlocks** and **Query Execution** remind us that concurrency and performance are **emergent behaviors** of our design.

Master this interplay, and you will not just query data—you will command it.