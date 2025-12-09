# 🗃️ The Senior SQL Query Playbook

> **Target Audience:** Data Engineers, Backend Developers, & Data Analysts  
> **Goal:** Master **Window Functions**, **Recursive CTEs**, and **Performance Optimization**.

In a senior interview, writing the query is only half the battle. You must write it **efficiently**. You need to know why `UNION ALL` is faster than `UNION` and how `RANK()` differs from `DENSE_RANK()`.

---

## 📖 Table of Contents
1. [Part 1: The Order of Execution (The Theory)](#-part-1-the-order-of-execution-the-theory)
2. [Part 2: Window Functions (The Senior Tool)](#-part-2-window-functions-the-senior-tool)
3. [Part 3: Joins & Set Operations (Tricky Edge Cases)](#-part-3-joins--set-operations-tricky-edge-cases)
4. [Part 4: Advanced Scenarios (Gaps, Islands, Recursion)](#-part-4-advanced-scenarios-gaps-islands-recursion)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 📜 Part 1: The Order of Execution (The Theory)

SQL is declarative. You write *what* you want, not *how* to get it. However, the engine executes lines in a specific order. Knowing this explains why you can't use an Alias in a `WHERE` clause.

**The Logical Order:**
1.  **FROM / JOIN:** Load data sets.
2.  **WHERE:** Filter rows (Pre-aggregation).
3.  **GROUP BY:** Aggregate rows.
4.  **HAVING:** Filter aggregated data.
5.  **SELECT:** Select columns.
6.  **ORDER BY:** Sort.
7.  **LIMIT:** Cut off.

> **💡 Senior Interview Tip:**
> *Question:* "Why does this query fail?" `SELECT name AS n FROM users WHERE n = 'John'`
> *Answer:* "Because `WHERE` executes before `SELECT`. The alias `n` does not exist yet."

---

## 🪟 Part 2: Window Functions (The Senior Tool)

If you use a subquery where a Window Function works, you might fail the interview.

### 1. Ranking (`RANK` vs `DENSE_RANK` vs `ROW_NUMBER`)
**Scenario:** Rank employees by salary.
* `ROW_NUMBER()`: Unique ID per row. (1, 2, 3, 4).
* `RANK()`: Skips numbers on ties. (1, 2, 2, **4**).
* `DENSE_RANK()`: No gaps. (1, 2, 2, **3**).

### 2. Time Travel (`LEAD` & `LAG`)
**Scenario:** Calculate Month-over-Month growth.

**Table: `sales`**

| month | revenue |
| :--- | :--- |
| 2023-01 | 10000 |
| 2023-02 | 12000 |
| 2023-03 | 15000 |

```sql
SELECT 
    month, 
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    (revenue - LAG(revenue) OVER (ORDER BY month)) as diff
FROM sales;
```

### 3. Aggregation without Grouping
**Scenario:** Show every employee's salary *alongside* the department average.

**Table: `employees`**

| id | name | salary | dept_id |
| :--- | :--- | :--- | :--- |
| 1 | Alice | 80000 | 101 |
| 2 | Bob | 70000 | 101 |
| 3 | Charlie | 90000 | 102 |

* **Naive:** Join a subquery grouping by department.
* **Senior:**

```sql
SELECT 
    name, 
    salary, 
    AVG(salary) OVER (PARTITION BY dept_id) as dept_avg
FROM employees;
```

---

## 🔗 Part 3: Joins & Set Operations (Tricky Edge Cases)

### 1. The Self-Join (Hierarchy)
**Scenario:** "Given an `Employees` table with `Id`, `Name`, and `ManagerId`. Find employees who earn more than their managers."

**Table: `Employees`**

| Id | Name | Salary | ManagerId |
| :--- | :--- | :--- | :--- |
| 1 | Boss | 150000 | NULL |
| 2 | Alice | 90000 | 1 |
| 3 | Bob | 160000 | 1 |

```sql
SELECT e.Name 
FROM Employees e
JOIN Employees m ON e.ManagerId = m.Id
WHERE e.Salary > m.Salary;
```
*Result: Bob*

### 2. Finding "Missing" Data (Left Join IS NULL)
**Scenario:** "Find Customers who have never placed an Order."

**Table: `Customers`**

| Id | Name |
| :--- | :--- |
| 1 | John |
| 2 | Sarah |
| 3 | Mike |

**Table: `Orders`**

| Id | CustomerId | Amount |
| :--- | :--- | :--- |
| 101 | 1 | 500 |
| 102 | 1 | 200 |
| 103 | 3 | 800 |

* **Avoid:** `NOT IN (SELECT id FROM Orders)` (Performance killer on large sets with NULLs).
* **Use:** `LEFT JOIN`.

```sql
SELECT c.Name
FROM Customers c
LEFT JOIN Orders o ON c.Id = o.CustomerId
WHERE o.Id IS NULL;
```
*Result: Sarah*

### 3. UNION vs. UNION ALL
**Interviewer:** "What's the difference?"
* **UNION:** Removes duplicates. Performs a `DISTINCT` sort operation (Expensive).
* **UNION ALL:** Appends sets. Fast.
* **Rule:** Always default to `UNION ALL` unless you specifically need deduplication.

---

## 🔄 Part 4: Advanced Scenarios (Gaps, Islands, Recursion)

### 1. Recursive CTEs (Org Chart)
**Scenario:** "Get all subordinates of Manager X, deep down the hierarchy."

```sql
WITH RECURSIVE Subordinates AS (
    -- Anchor member: The Manager
    SELECT id, name, manager_id FROM employees WHERE id = 1
    UNION ALL
    -- Recursive member: People managed by the set above
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    INNER JOIN Subordinates s ON e.manager_id = s.id
)
SELECT * FROM Subordinates;
```

### 2. Running Totals
**Scenario:** "Calculate cumulative revenue by day."

**Table: `revenue`**

| date | amount |
| :--- | :--- |
| 2023-01-01 | 100 |
| 2023-01-02 | 150 |
| 2023-01-03 | 200 |

```sql
SELECT 
    date, 
    amount, 
    SUM(amount) OVER (ORDER BY date) as running_total
FROM revenue;
```

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The N-th Highest Salary
**Interviewer:** *"Find the 3rd highest salary in the company."*

**Table: `employees`**

| id | name | salary |
| :--- | :--- | :--- |
| 1 | A | 100k |
| 2 | B | 90k |
| 3 | C | 90k |
| 4 | D | 80k |

* ❌ **Junior Answer:** `SELECT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2`.
    * *Why it fails:* It would return 90k (the 3rd row), but 90k is the 2nd highest salary.
* ✅ **Senior Answer:** "**Use DENSE_RANK().**"

```sql
WITH Ranked AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM employees
)
SELECT salary FROM Ranked WHERE rnk = 3;
```
*Result: 80k*

### Scenario B: Removing Duplicates (Data Cleaning)
**Interviewer:** *"We have a table with duplicate emails. Keep the one with the lowest ID, delete the rest."*

**Table: `users`**

| id | email |
| :--- | :--- |
| 1 | a@x.com |
| 2 | a@x.com |
| 3 | b@x.com |

* ✅ **Senior Answer:** "**Delete with CTE.**"

```sql
WITH Dups AS (
    SELECT id, 
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id ASC) as rn
    FROM users
)
DELETE FROM users 
    WHERE id IN (SELECT id FROM Dups WHERE rn > 1);
```
*Result: Deletes row with id=2.*

### Scenario C: The "Islands" Problem (Consecutive Days)
**Interviewer:** *"Find all users who logged in for 3 consecutive days."*

**Table: `logins`**

| user_id | login_date |
| :--- | :--- |
| 1 | 2023-01-01 |
| 1 | 2023-01-02 |
| 1 | 2023-01-03 |
| 2 | 2023-01-01 |
| 2 | 2023-01-03 |

* **The Logic:** If you subtract the `Row_Number` from the `Date`, the result is constant for consecutive days.
* ✅ **Senior Answer:**

```sql
SELECT user_id, count(*)
FROM (
    SELECT user_id, login_date,
           DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) as grp
    FROM logins
) t
GROUP BY user_id, grp
HAVING count(*) >= 3;
```

### Scenario D: Pivot Table (Rows to Columns)
**Interviewer:** *"Convert this data: `[Month, Product, Revenue]` into columns: `[Month, Product_A_Rev, Product_B_Rev]`."*

**Table: `sales`**

| month | product | revenue |
| :--- | :--- | :--- |
| Jan | A | 100 |
| Jan | B | 200 |
| Feb | A | 150 |

* ✅ **Senior Answer:** "**Conditional Aggregation.**"

```sql
SELECT 
    month,
    SUM(CASE WHEN product = 'A' THEN revenue ELSE 0 END) as Product_A,
    SUM(CASE WHEN product = 'B' THEN revenue ELSE 0 END) as Product_B
    FROM sales
GROUP BY month;
```

---

### **Final Checklist**
1.  **Window Functions:** `RANK`, `LEAD`, `LAG` are mandatory.
2.  **Performance:** `LEFT JOIN ... NULL` > `NOT IN`.
3.  **Recursion:** Know `WITH RECURSIVE` for tree structures.
4.  **Deduplication:** Use `ROW_NUMBER()` partitioning.

**This concludes the SQL Query Playbook.**