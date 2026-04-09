
### Part 1: Backup and Recovery - The Art of Resurrection

Hardware fails. Software corrupts. Humans type `DELETE FROM production_table;` without a `WHERE` clause. **Data loss is a certainty, not a possibility.** The measure of an engineer is not preventing the inevitable, but **Mean Time To Recovery (MTTR)** .

#### The Three Pillars of Recovery

| Pillar | Purpose | Mechanism |
| :--- | :--- | :--- |
| **Backup** | Physical copy of data at a point in time. | `pg_dump`, `pg_basebackup`, `mysqldump`. |
| **WAL Archiving** | Continuous stream of changes. | Write-Ahead Log (WAL) shipping. |
| **Transaction Log** | Ability to roll forward or backward. | Point-In-Time Recovery (PITR). |

#### 1. Types of Backups (The 10^10 IQ Strategy)

| Type | Description | Speed (Create) | Speed (Restore) | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Logical (pg_dump)** | SQL statements to recreate schema and data. | Slow | Very Slow | Migrating to new version, partial restore. |
| **Physical (File-Level)** | Copy of the data directory files. **Must be consistent snapshot.** | Fast | Fast | Disaster Recovery. |
| **Incremental** | Only changes since last backup. | Very Fast | Slower (replay chain) | High-frequency backups (hourly). |

**The 10^10 IQ Insight:** Never rely solely on Logical Backups for a 5TB production database. Restoring `pg_dump` for 5TB takes **days** due to index rebuilding and constraint validation. Use **Physical Backups** (`pg_basebackup` or filesystem snapshots) for the base, combined with **WAL Archiving** for point-in-time precision.

#### 2. Point-In-Time Recovery (PITR) - The Time Machine

**Scenario:** A developer runs a script that corrupts the `Orders` table at 10:32 AM. You need to restore the database to exactly **10:31:59 AM**.

**Mechanism:**
1. Restore the last **Physical Base Backup** (taken at 2:00 AM).
2. **Replay** the archived WAL files generated between 2:00 AM and 10:31:59 AM.
3. The database is now in the exact state just before the fat-finger moment.

**PostgreSQL Configuration (`postgresql.conf`):**
```ini
wal_level = replica           # Enables WAL archiving
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

**Recovery Configuration (`recovery.signal` file + `postgresql.auto.conf`):**
```ini
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-04-10 10:31:59'
```

#### 3. Backup and Restore Commands (Practical Examples)

**PostgreSQL Logical Backup:**
```bash
# Dump specific database
pg_dump -h localhost -U postgres -d mydb -Fc -f mydb_backup.dump

# Restore
pg_restore -h localhost -U postgres -d mydb -j 4 mydb_backup.dump
```

**PostgreSQL Physical Base Backup (using pg_basebackup):**
```bash
# Take a base backup
pg_basebackup -h localhost -U replication_user -D /backup/base -Ft -z -P

# Recovery: Extract base backup, configure recovery settings, start PostgreSQL.
```

**MySQL / MariaDB:**
```bash
# Logical dump
mysqldump --single-transaction --routines --triggers --all-databases > backup.sql

# Physical (Percona XtraBackup for hot backups)
xtrabackup --backup --target-dir=/backup/base
```

#### 4. The 3-2-1 Backup Rule (The Senior Engineer's Mantra)

- **3** copies of the data.
- **2** different storage media (e.g., local NAS + cloud object storage).
- **1** copy offsite (geographically separate).

---

### Part 2: PL/SQL (and PL/pgSQL) - Embedding Logic in the Engine

**PL/SQL** (Procedural Language/Structured Query Language) is Oracle's extension. **PL/pgSQL** is PostgreSQL's equivalent. They are **imperative languages** that live **inside the database**.

**Why not just do logic in Python/Java?**
Because moving **millions of rows over the network to the app server, processing them, and sending them back** is architectural madness. PL/SQL allows you to push **loops, conditions, and complex calculations directly to the data**.

#### 1. Basic Block Structure (Anonymous Block)

```sql
-- PostgreSQL PL/pgSQL Example
DO $$ 
DECLARE
    v_customer_name TEXT;
    v_order_count INT;
BEGIN
    -- Select into variables
    SELECT name INTO v_customer_name 
    FROM Customers 
    WHERE customer_id = 1;
    
    -- Get count
    SELECT COUNT(*) INTO v_order_count 
    FROM Orders 
    WHERE customer_id = 1;
    
    -- Conditional Logic
    IF v_order_count > 10 THEN
        RAISE NOTICE 'VIP Customer: % has % orders', v_customer_name, v_order_count;
    ELSE
        RAISE NOTICE 'Regular Customer: %', v_customer_name;
    END IF;
END $$;
```

#### 2. Stored Procedures vs. Functions

| Aspect | Procedure | Function |
| :--- | :--- | :--- |
| **Return Value** | Can return multiple via `OUT` params, or none. | **Must return a single value or table.** |
| **Usage in SQL** | Called with `CALL proc_name();` | Used in `SELECT func_name();` |
| **Transaction Control** | Can `COMMIT` / `ROLLBACK` internally. | Cannot control transactions (runs in caller's context). |
| **Primary Use** | Data modification batches, complex workflows. | Computations, data transformations. |

**Example: Function to Calculate Discount**
```sql
CREATE OR REPLACE FUNCTION calculate_discount(p_customer_id INT, p_order_total DECIMAL)
RETURNS DECIMAL AS $$
DECLARE
    v_loyalty_years INT;
BEGIN
    SELECT EXTRACT(YEAR FROM age(NOW(), signup_date)) INTO v_loyalty_years
    FROM Customers WHERE customer_id = p_customer_id;
    
    IF v_loyalty_years > 5 THEN
        RETURN p_order_total * 0.15; -- 15% discount
    ELSE
        RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Usage in SELECT
SELECT order_id, total, calculate_discount(customer_id, total) AS discount
FROM Orders WHERE order_id = 123;
```

#### 3. Cursors - Row-by-Row Processing (Use Sparingly)

**Warning:** Cursors break the set-based paradigm of SQL. Use only when set-based operations are impossible (e.g., calling external API per row, or complex iterative logic).

```sql
DO $$
DECLARE
    cur CURSOR FOR SELECT customer_id, email FROM Customers WHERE is_active = TRUE;
    rec RECORD;
BEGIN
    OPEN cur;
    LOOP
        FETCH cur INTO rec;
        EXIT WHEN NOT FOUND;
        
        -- Send email to rec.email (via external function or notification)
        RAISE NOTICE 'Processing customer %', rec.customer_id;
    END LOOP;
    CLOSE cur;
END $$;
```

#### 4. Triggers - Automatic Execution on Events

A **Trigger** is a procedure that fires automatically in response to `INSERT`, `UPDATE`, or `DELETE` events.

**Example: Automatic `updated_at` Timestamp**
```sql
-- Function to update timestamp
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger on Products table
CREATE TRIGGER update_product_modtime
    BEFORE UPDATE ON Products
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_column();
```

#### 5. Exception Handling

```sql
DO $$
BEGIN
    INSERT INTO Orders (order_id, customer_id) VALUES (1, 999);
EXCEPTION
    WHEN foreign_key_violation THEN
        RAISE NOTICE 'Customer does not exist. Aborting.';
        ROLLBACK;
    WHEN OTHERS THEN
        RAISE NOTICE 'Unexpected error: %', SQLERRM;
END $$;
```

---

### Part 3: Advanced SQL Features - The Esoteric and Powerful

These are the features that separate the query writers from the data shapers.

#### 1. `MERGE` / `UPSERT` (Idempotent Writes)

**Scenario:** Insert a row. If it already exists (conflict on PK), update it instead.

**PostgreSQL (`INSERT ... ON CONFLICT`):**
```sql
INSERT INTO Inventory (product_id, quantity)
VALUES (101, 50)
ON CONFLICT (product_id) DO UPDATE
SET quantity = Inventory.quantity + EXCLUDED.quantity;
```

**Standard SQL `MERGE` (Oracle, SQL Server, PostgreSQL 15+):**
```sql
MERGE INTO Inventory AS target
USING (VALUES (101, 50)) AS source (product_id, quantity)
ON target.product_id = source.product_id
WHEN MATCHED THEN
    UPDATE SET quantity = target.quantity + source.quantity
WHEN NOT MATCHED THEN
    INSERT (product_id, quantity) VALUES (source.product_id, source.quantity);
```

#### 2. Lateral Joins - Correlated Subqueries in `FROM`

**Scenario:** For each department, find the top 3 highest-paid employees. Without `LATERAL`, this is painful.

```sql
SELECT d.dept_name, e.name, e.salary
FROM Departments d
CROSS JOIN LATERAL (
    SELECT name, salary
    FROM Employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
    LIMIT 3
) e;
```
*`LATERAL` allows the subquery to reference columns from preceding `FROM` items.*

#### 3. `WITH ORDINALITY` - Numbering Rows from Set-Returning Functions

**Scenario:** You have a function that returns a set of values, and you want to preserve the order index.

```sql
SELECT *
FROM json_array_elements_text('["apple", "banana", "cherry"]') WITH ORDINALITY AS elem(value, idx);
-- Result: ('apple', 1), ('banana', 2), ('cherry', 3)
```

#### 4. `FILTER` Clause for Aggregates

Instead of complex `CASE` statements inside aggregates, use `FILTER`.

```sql
SELECT
    department,
    COUNT(*) AS total_employees,
    COUNT(*) FILTER (WHERE salary > 100000) AS high_earners,
    AVG(salary) FILTER (WHERE hire_date > '2020-01-01') AS avg_recent_salary
FROM Employees
GROUP BY department;
```

#### 5. JSON/JSONB - The Best of Both Worlds

PostgreSQL's **JSONB** is a binary representation that allows indexing and fast operators, giving you document-store flexibility within a relational database.

```sql
-- Table with JSONB column
CREATE TABLE Products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes JSONB
);

INSERT INTO Products (name, attributes) VALUES 
('Laptop', '{"brand": "Dell", "ram": 16, "ssd": true}');

-- Query inside JSON
SELECT * FROM Products WHERE attributes @> '{"ram": 16}';
SELECT attributes->>'brand' AS brand FROM Products;

-- Index for fast JSON queries
CREATE INDEX idx_products_attributes ON Products USING GIN (attributes);
```

#### 6. `LISTAGG` / `STRING_AGG` - String Concatenation Across Rows

**Scenario:** For each order, list all product names in a single comma-separated string.

```sql
SELECT
    o.order_id,
    STRING_AGG(p.name, ', ' ORDER BY p.name) AS products_list
FROM Orders o
JOIN Order_Items oi ON o.order_id = oi.order_id
JOIN Products p ON oi.product_id = p.product_id
GROUP BY o.order_id;
```

#### 7. Recursive CTEs (Already Covered Briefly, Deeper Example)

**Example: Exploding a Bill of Materials (BOM)**
```sql
WITH RECURSIVE bom_tree (part_id, component_id, quantity, level) AS (
    -- Anchor: Top-level assembly
    SELECT part_id, component_id, quantity, 1
    FROM BillOfMaterials
    WHERE part_id = 'FinalProduct'
    
    UNION ALL
    
    -- Recursive: Join to find subcomponents
    SELECT b.part_id, b.component_id, b.quantity, bt.level + 1
    FROM BillOfMaterials b
    JOIN bom_tree bt ON b.part_id = bt.component_id
)
SELECT * FROM bom_tree ORDER BY level;
```

### Summary: The Senior Engineer's Trinity

| Domain | Core Principle | Ultimate Goal |
| :--- | :--- | :--- |
| **Backup & Recovery** | Redundancy and Replayability | **Zero Data Loss** within Recovery Point Objective (RPO). |
| **PL/SQL** | Logic at the Data Layer | **Minimize network round trips** and ensure atomic complex operations. |
| **Advanced SQL** | Expressive Power | **Push computation to the database engine** instead of application memory. |

