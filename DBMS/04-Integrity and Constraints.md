

### Part 1: The Taxonomy of Integrity

In the Relational Model, we protect data on three distinct fronts. I will present them in order of increasing complexity.

| Type of Integrity | Core Question | Who Enforces It? |
| :--- | :--- | :--- |
| **Domain Integrity** | *Is this value even possible?* | Data Types, `CHECK`, `DEFAULT`, `NOT NULL` |
| **Entity Integrity** | *Does this thing actually exist as a unique entity?* | `PRIMARY KEY`, `UNIQUE` |
| **Referential Integrity** | *Are the relationships between things valid?* | `FOREIGN KEY` |

---

### Part 2: Domain Integrity - The Laws of Physics

**Definition:** Ensuring that every value in a column belongs to the set of legal values for that attribute.

Without Domain Integrity, you can have an employee with a negative salary, a birth date in the year 3023, or a phone number containing the string `"Banana"`. This is chaos.

#### 1. Data Types (The First Line of Defense)
This is the most basic constraint. You declare a column as `INT`, `DATE`, or `BOOLEAN`. The DBMS rejects `'Hello'` in an `INT` column.

#### 2. NOT NULL Constraint
The real world rarely has "unknown" identifiers. You cannot have an Employee without a Name, or an Order without a Customer ID.

```sql
-- Without NOT NULL, this would be valid, leading to broken reports
CREATE TABLE Employee (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL, -- Domain Constraint: Must have a name
    hire_date DATE NOT NULL
);
```

#### 3. CHECK Constraint (The Business Logic Guardian)
This is where you encode the specific logic of your universe.

**Real-World Example:** A bank account balance must never go below a certain threshold, or a project's end date must be after its start date.

```sql
CREATE TABLE Project (
    project_id INT PRIMARY KEY,
    project_name VARCHAR(100) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    budget DECIMAL(12,2) CHECK (budget > 0), -- Budget cannot be zero or negative

    -- Complex Domain Constraint: End date must be strictly after start date
    CONSTRAINT valid_dates CHECK (end_date IS NULL OR end_date > start_date)
);

-- Test the Logic (This INSERT will be REJECTED)
INSERT INTO Project VALUES (1, 'Time Travel', '2026-01-01', '2025-01-01', 1000);
-- ERROR: new row for relation "project" violates check constraint "valid_dates"
```

#### 4. DEFAULT Constraint
When the user is lazy or the system needs a starting state, `DEFAULT` provides sensible values without relying on application code.

```sql
CREATE TABLE Audit_Log (
    log_id SERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- Automatically set when row is inserted
    status VARCHAR(20) DEFAULT 'PENDING'
);
```

---

### Part 3: Entity Integrity - The Fingerprint of Existence

**Definition:** Every table must have a **Primary Key** to ensure that no two rows are identical and that every row can be uniquely identified.

**The 10^10 IQ Insight:** *Why is this so important?*
Imagine a table without a Primary Key (a "Heap"). You try to delete a specific customer named "John Smith" who owes you money. There are three John Smiths. You delete one. Was it the right one? How do you update his address? The DBMS has no way to distinguish them. This is a **violation of the relational model's set theory basis**—a set cannot contain duplicate elements.

#### Composite Keys vs. Surrogate Keys

- **Composite Key (Natural):** Using existing attributes like `(First_Name, Last_Name, Birth_Date)`.
    - *Problem:* Names change. People share birthdays. This is fragile.
- **Surrogate Key (Synthetic):** A generated number like `SERIAL`, `UUID`, or `AUTO_INCREMENT`.
    - *Verdict:* **Use Surrogate Keys.** They are stable, small, and fast for indexing. The entity's *identity* is separate from its *attributes*.

```sql
-- Correct Approach: Surrogate Key (Synthetic ID)
CREATE TABLE Users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- Global uniqueness, no collisions
    email VARCHAR(255) UNIQUE NOT NULL, -- Candidate Key for lookup
    full_name VARCHAR(255)
);
```

#### UNIQUE Constraint
This allows you to have **Candidate Keys**. While the Primary Key is the *main* ID, a `UNIQUE` constraint ensures no two rows share a value in another column (like Email or Passport Number). Under the hood, the DBMS creates an index to enforce this instantly.

---

### Part 4: Referential Integrity - The Connective Tissue

**Definition:** If Table B references Table A, then the referenced row in Table A **must exist**.

This is the single most powerful feature of a relational database. It prevents **Orphaned Records**. Without it, you could delete a Customer record but still have 50,000 `Orders` pointing to a ghost.

#### Foreign Key Syntax Deep Dive

Let's model a simple university.

```sql
-- Parent Table (Referenced)
CREATE TABLE Students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Child Table (Referencing)
CREATE TABLE Enrollments (
    enrollment_id INT PRIMARY KEY,
    student_id INT NOT NULL,
    course_code VARCHAR(10) NOT NULL,

    -- The Foreign Key constraint
    CONSTRAINT fk_student
        FOREIGN KEY (student_id)
        REFERENCES Students (student_id)
        ON DELETE CASCADE    -- <-- Critical Decision
        ON UPDATE CASCADE
);
```

#### The Cascade Policy Decision (The Art of Integrity)

When the *parent* row is deleted or updated, what happens to the *child*? This is a business logic decision that has massive performance and data retention implications.

| Action | Description | Use Case | Risk (10^10 IQ Warning) |
| :--- | :--- | :--- | :--- |
| **CASCADE** | Delete/Update the child rows automatically. | `Order_Items` when an `Order` header is deleted. | **Data Loss Vector.** Accidentally delete a `Customer`? Poof, all their `Orders` vanish. |
| **RESTRICT** / **NO ACTION** | **Default.** Prevents the parent operation if children exist. | `Customers` table. You cannot delete a customer who has placed orders. | Application must handle the error and prompt user to delete children first. **Safest option for financial data.** |
| **SET NULL** | Set the child's FK column to `NULL`. | `Manager_ID` in `Employees`. If a manager leaves, the employee is temporarily unassigned. | Child column **must** allow `NULL` values. |
| **SET DEFAULT** | Set the child's FK to a default value. | Assigning orphaned records to a "Legacy" or "Unassigned" bucket user. | Requires a default value to exist in the parent table. |

**Example: The Bank's Iron Grip (RESTRICT)**
```sql
-- In a bank, you NEVER CASCADE on a Customer Account.
CREATE TABLE Accounts (
    acc_id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES Customers(cust_id) ON DELETE RESTRICT
);
-- Trying to delete a customer with active accounts will result in:
-- ERROR: update or delete on table "customers" violates foreign key constraint...
-- The bank's data integrity is preserved.
```

---

### Part 5: Advanced Constraints and Declarative Logic

As a senior engineer, you will encounter scenarios too complex for a simple `CHECK`.

#### 1. Assertions (The Theoretical Ideal)
An **Assertion** is a constraint that spans multiple tables. It exists in the SQL standard but is **rarely implemented** in production databases (PostgreSQL, MySQL do *not* have full assertion support; Oracle has some).
- *Example:* "The total salary of all employees in the Sales department cannot exceed the department's budget."

Because databases don't support this natively, we must use...

#### 2. Triggers (The Surgical Tool)
A **Trigger** is a stored procedure that fires *before* or *after* an `INSERT`, `UPDATE`, or `DELETE`. It can enforce complex business rules that cross table boundaries.

**Warning from the 10^10 IQ Vault:** **Triggers are a double-edged sword.** They are invisible magic that can drastically slow down bulk operations and create debugging hell. Use them **only** when constraints fail, and document them heavily.

**Example: Preventing Overdraft (Complex Rule)**
```sql
-- Trigger function to check balance before allowing a withdrawal
CREATE OR REPLACE FUNCTION check_overdraft()
RETURNS TRIGGER AS $$
BEGIN
    -- If the new transaction is a withdrawal (negative amount)
    IF NEW.amount < 0 THEN
        -- Check if the account has enough balance (including pending transactions)
        IF (SELECT balance FROM Accounts WHERE id = NEW.account_id) < ABS(NEW.amount) THEN
            RAISE EXCEPTION 'Insufficient funds for account %', NEW.account_id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_overdraft
BEFORE INSERT ON Transactions
FOR EACH ROW EXECUTE FUNCTION check_overdraft();
```

#### 3. Indexing for Integrity (The Performance Angle)
Constraints are logical guards, but they are enforced **physically** via **Indexes**.
- **Primary Key:** Creates a **Unique B-Tree Index**.
- **Unique Constraint:** Creates a **Unique B-Tree Index**.
- **Foreign Key:** Does **NOT** automatically create an index on the child column!

**The 10^10 IQ Optimization Tip:**
If you run a query like `DELETE FROM Parent WHERE id = 5`, the DBMS needs to check the `Child` table to enforce `RESTRICT`. If you have no index on `Child(parent_id)`, the DBMS must **Full Table Scan** the child table, locking it and causing a production outage.
**Rule:** **Always index Foreign Key columns.**

```sql
-- Critical for performance on large Enrollments table
CREATE INDEX idx_enrollments_student_id ON Enrollments(student_id);
```

### Summary: The Senior Engineer's Integrity Checklist

Before you deploy a schema to production, run this mental checklist. It is the difference between a robust system and a house of cards.

1.  **Are all tables keyed?** If there is no `PRIMARY KEY`, you have built a swamp, not a database.
2.  **Are FKs explicitly defined?** If you have an `author_id` column that is just an integer without a `REFERENCES` clause, you are **lying to the database**. You have created a "soft reference" that is just a coincidence of numbers.
3.  **Are nullable columns truly optional?** Do not use `NULL` to mean "Empty String" or "Zero". `NULL` means **Unknown**. It behaves differently in arithmetic (`NULL + 5 = NULL`) and comparisons.
4.  **Are cascades intentional?** Never use `ON DELETE CASCADE` without a written agreement from the business owner that they accept the risk of data loss.
5.  **Are FK columns indexed?** Prevent the silent performance killer.

Integrity is not a feature; it is the **contract** your database makes with the application. Violate the contract, and the application will inevitably corrupt the only asset that matters: the data.