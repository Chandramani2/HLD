# ⚡ The Senior DynamoDB Query Playbook

> **Target Audience:** Cloud Architects, Backend Leads, & Serverless Engineers  
> **Goal:** Master **Access Patterns**, **Key Condition Expressions**, and **Single-Request Retrievals**.

In a senior interview, you must understand that **Scans are a failure**, **Filter Expressions do not save money**, and **Pagination is not optional**.

---

## 📖 Table of Contents
1. [Part 1: The Golden Rule (Query vs. Scan)](#-part-1-the-golden-rule-query-vs-scan)
2. [Part 2: Key Conditions (Slicing the Sort Key)](#-part-2-key-conditions-slicing-the-sort-key)
3. [Part 3: The "Filter" Trap (Read Cost)](#-part-3-the-filter-trap-read-cost)
4. [Part 4: Advanced Patterns (Sparse Indexes & Overloading)](#-part-4-advanced-patterns-sparse-indexes--overloading)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🔑 Part 1: The Golden Rule (Query vs. Scan)

### 1. The Query (`O(1)` or `O(log N)`)
* **Mechanism:** Goes directly to a Partition. Can strictly target a subset of data using the Sort Key.
* **Cost:** You pay only for the items you retrieve.
* **Requirement:** You **MUST** provide the exact Partition Key (PK).

### 2. The Scan (`O(N)`)
* **Mechanism:** Reads every single item in the table, from start to finish.
* **Cost:** You pay to read the entire table (even if you filter 99% of it out).
* **Use Case:** Only acceptable for Data Warehousing export (ETL) or tiny tables (< 1MB).

> **💡 Senior Interview Tip:**
> If an interviewer asks "How do I find all users named 'John'?", and the PK is `UserID`:
> * **Junior:** "Use a Scan with a filter."
> * **Senior:** "That's an inefficient access pattern. I would create a GSI (Global Secondary Index) where `PK = Name` to turn that Scan into a Query."

---

## ✂️ Part 2: Key Conditions (Slicing the Sort Key)

The power of DynamoDB lies in the **Sort Key (SK)**. It allows range queries inside a partition.

**Context:** Storing Orders.
* **PK:** `USER#123`
* **SK:** `ORDER#{Date}` (e.g., `ORDER#2023-01-01`)

### 1. `begins_with()`
Fetch a hierarchy.
* **Scenario:** "Get all Orders."
* **Query:** `PK = "USER#123" AND begins_with(SK, "ORDER#")`.
* **Why?** If the user also has `PROFILE` data in the same partition, this efficiently skips it.

### 2. `between()`
Fetch a time range.
* **Scenario:** "Get Orders for January."
* **Query:** `PK = "USER#123" AND SK between "ORDER#2023-01-01" AND "ORDER#2023-01-31"`.

### 3. Inequalities (`>`, `<`)
* **Scenario:** "Get all orders since March."
* **Query:** `PK = "USER#123" AND SK > "ORDER#2023-03-01"`.

---

## 💸 Part 3: The "Filter" Trap (Read Cost)

This is the most misunderstood concept in DynamoDB billing.

### The Workflow
1.  **Lookup:** DynamoDB finds the items matching your `KeyConditionExpression` (PK + SK).
2.  **Read:** It reads the data from disk (You pay for **all** of this).
3.  **Filter:** It applies your `FilterExpression` to remove items (e.g., `status = 'cancelled'`).
4.  **Return:** It returns the remaining items (Max 1MB).

### The Implication
* **Scenario:** You Query 1,000 items. 999 are "cancelled". 1 is "active". You filter for "active".
* **Result:** You receive 1 item. **You pay for reading 1,000 items.**
* **Senior Strategy:** If you frequently filter by `status`, move `status` into the Key or an Index. Don't use Filter Expressions for heavy lifting.

---

## 🧠 Part 4: Advanced Patterns (Sparse Indexes & Overloading)

### 1. Index Overloading (GSI)
Instead of creating 5 Indexes for 5 access patterns, create **One Generic GSI**.
* **Base Table:** `PK`, `SK`, `Data`, `GSI1_PK`, `GSI1_SK`.
* **Pattern A:** Access by Email. -> Copy Email to `GSI1_PK`.
* **Pattern B:** Access by Date. -> Copy Date to `GSI1_PK`.
* **Query:** Query the GSI. The data types in the column are mixed, but your query targets specific entities.

### 2. Sparse Indexes (The "Null" Trick)
DynamoDB only writes to a GSI if the index key **exists**.
* **Scenario:** Find "Urgent" tasks among 1 Million tasks. Only 100 are urgent.
* **Strategy:**
  * Create a GSI on attribute `IsUrgent`.
  * For normal tasks, leave `IsUrgent` as `null` (undefined).
  * For urgent tasks, set `IsUrgent = "YES"`.
* **Result:** The GSI only contains 100 items. Scanning the GSI is extremely cheap and fast.

[Image of DynamoDB Sparse Index Diagram]

---

## 🧪 Part 5: Senior Level Q&A Scenarios

### Scenario A: Pagination (The Infinite Scroll)
**Interviewer:** *"We have 1 million orders. How do we implement page 2?"*

* ❌ **Bad Answer:** "Use an Offset." (DynamoDB doesn't support offset).
* ✅ **Senior Answer:** "**LastEvaluatedKey.**"
  * **Request 1:** `Limit: 10`.
  * **Response 1:** Returns 10 items + `LastEvaluatedKey: { PK: "A", SK: "10" }`.
  * **Request 2:** `Limit: 10`, `ExclusiveStartKey: { PK: "A", SK: "10" }`.
  * **Result:** Stateless, efficient cursor-based pagination.

### Scenario B: Fetching "User + Orders" (One Query)
**Interviewer:** *"I want to load the User Profile and their recent Orders. I don't want to make two network calls."*

* ✅ **Senior Answer:** "**Single Table Design (Query Collection).**"
  * **Data:**
    * User Row: `PK: USER#1`, `SK: A_PROFILE`, `Name: "John"`.
    * Order Row: `PK: USER#1`, `SK: ORDER#99`, `Total: $50`.
  * **Query:** `PK = "USER#1"`.
  * **Result:** DynamoDB returns the User Profile (sorted first because 'A' < 'O') followed by all Orders.
  * **Client-side:** Parse the array. First item is Profile, rest are Orders.

### Scenario C: Sorting by Non-Key Attribute
**Interviewer:** *"I want to fetch Orders sorted by Price. The Sort Key is Date."*

* **Constraint:** DynamoDB can *only* sort by the Sort Key.
* ✅ **Senior Answer:** "**Create a GSI.**"
  * **GSI PK:** `UserID`.
  * **GSI SK:** `Price` (Pad numbers: `00050.00` for correct string sorting).
  * **Query:** Query the GSI.
  * **Note:** This adds cost (write amplification). Only do this if "Sort by Price" is a critical feature.

### Scenario D: Strong Consistency on GSI
**Interviewer:** *"I wrote to the table, then immediately queried the GSI, but the data is missing."*

* ✅ **Senior Answer:** "**GSIs are always Eventually Consistent.**"
  * There is a milliseconds replication lag between Base Table and GSI.
  * **Fix:** If you absolutely need Strong Consistency, you **cannot** use a GSI. You must design your Base Table PK/SK to support the query directly and use `ConsistentRead: True`.

---

### **Final Checklist**
1.  **Params:** `KeyConditionExpression` is fast. `FilterExpression` is slow/expensive.
2.  **Sorting:** You can only sort by the Sort Key.
3.  **Pagination:** Use `LastEvaluatedKey`.
4.  **Cost:** Minimize GSI projections to save storage and IO.

**This concludes the DynamoDB Query Playbook.**