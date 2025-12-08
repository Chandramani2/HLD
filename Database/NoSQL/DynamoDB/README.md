# ⚡ The Senior DynamoDB Playbook

> **Target Audience:** Cloud Architects & Backend Engineers  
> **Goal:** Master **Access Patterns**, **Single Table Design**, and **Serverless Scale**.

In a senior interview, "using DynamoDB" means you understand that **SQL Joins are impossible**, **Scans are expensive**, and **Data Modeling starts with Queries, not Entities**.

---

## 📖 Table of Contents
1. [Part 1: Core Architecture (Partitions & Keys)](#-part-1-core-architecture-partitions--keys)
2. [Part 2: Data Modeling (Single Table Design)](#-part-2-data-modeling-single-table-design)
3. [Part 3: Indexes (LSI vs GSI)](#-part-3-indexes-lsi-vs-gsi)
4. [Part 4: Consistency & Transactions (ACID)](#-part-4-consistency--transactions-acid)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: Core Architecture (Partitions & Keys)

DynamoDB is a distributed B-Tree. It stores data on SSDs across 3 Availability Zones.

### 1. The Primary Key (PK)
This is how DynamoDB distributes data.
* **Simple Primary Key:** Just a Partition Key (e.g., `UserID`).
    * *Behavior:* Like a Key-Value store (Redis).
* **Composite Primary Key:** Partition Key + Sort Key.
    * *Behavior:* Like a HashMap of Sorted Maps.
    * *Example:* PK=`Artist`, SK=`SongTitle`.
    * *Query:* "Give me all songs by Artist X, sorted by Title."

### 2. Physical Partitions (The 10GB Rule)
* DynamoDB slices your table into 10GB chunks called **Partitions**.
* **Hashing:** Data is assigned to a partition based on the `Hash(PartitionKey)`.
* **Throughput Limit:** Each partition can support max **3,000 RCUs** (Read Capacity Units) and **1,000 WCUs** (Write Capacity Units).
* **Senior Insight:** If one Partition Key (e.g., `TenantID: CocaCola`) gets too much traffic, it hits the 3,000 RCU limit, even if the rest of the table is idle. This is a **Hot Partition**.

[Image of DynamoDB Partitioning and Hashing]

---

## 📝 Part 2: Data Modeling (Single Table Design)

In SQL, you normalize. In DynamoDB, you optimize for the **Number of Network Requests**. Ideally, 1 Request = All Data.

### 1. Access Pattern First
You **cannot** design a schema until you know every query.
* **Bad:** "I'll store Users and Orders."
* **Good:** "I need to fetch 'User Profile' AND 'Last 5 Orders' in one call."

### 2. Single Table Design (Key Overloading)
Put different entities (`User`, `Order`, `Product`) in the **same table**.
* **Generic Names:** Columns are named `PK` and `SK` (not `UserID` or `OrderID`).
* **Example:**
    * **User Row:** `PK: USER#123`, `SK: PROFILE`, `Data: {Name: "John"}`.
    * **Order Row:** `PK: USER#123`, `SK: ORDER#999`, `Data: {Total: $50}`.
* **The Magic Query:** `Query(PK="USER#123")`.
    * Returns the Profile **AND** all Orders in a single request. No Joins needed.

### 3. The Item Size Limit (400KB)
* You cannot store large blobs.
* **Strategy:** Store the image/PDF in **S3**. Store the S3 URL in DynamoDB.

---

## 🔎 Part 3: Indexes (LSI vs GSI)

You need to query by something other than the Primary Key? You need an index.

| Feature | Local Secondary Index (LSI) | Global Secondary Index (GSI) |
| :--- | :--- | :--- |
| **Partition Key** | Must be same as Base Table. | Can be any attribute. |
| **Sort Key** | Different attribute. | Different attribute. |
| **Creation** | **Only at Table Creation.** | Anytime. |
| **Consistency** | Strong or Eventual. | **Eventual Only.** |
| **Storage** | Shares partition with Base Table. | Separate Table (Async Replication). |
| **Best For** | Alternative sorting (e.g., Sort Orders by `Date` vs `Price`). | Completely new query patterns (e.g., Find Order by `Email`). |

> **💡 Senior Tip:** **Avoid LSIs.** They impose a 10GB size limit on the Partition Key collection. GSIs are flexible and scalable.

---

## 🛡️ Part 4: Consistency & Transactions (ACID)

DynamoDB is fundamentally **Eventual Consistency**, but can be tuned.

### 1. Read Consistency
* **Eventual (Default):** Fastest. Costs 0.5 RCU. Might reflect data from 100ms ago.
* **Strongly Consistent:** `ConsistentRead: true`. Costs 1.0 RCU. Guaranteed latest data.
    * *Constraint:* Not available on GSIs.

### 2. Transactions (ACID)
* **`TransactWriteItems`:** Group up to 100 actions (Put, Update, Delete) into one atomic unit.
* **Cost:** Expensive (2x WCUs).
* **Use Case:** "Transfer Money" (Debit User A, Credit User B). If one fails, both fail.

### 3. Capacity Modes
* **Provisioned:** You pay for X reads/sec. Good for predictable loads.
* **On-Demand:** You pay per request. Good for spiky/unknown workloads.
    * *Warning:* 7x more expensive per request than Provisioned.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Justin Bieber" Problem (Hot Key)
**Interviewer:** *"We have a `Followers` table. When Justin Bieber signs up, millions of users try to follow him instantly. Partition `PK: Bieber` throttles."*

* **The Problem:** Massive write heat on a single Partition Key.
* ✅ **Senior Answer:** "**Write Sharding.**"
    * Don't write to `PK: Bieber`.
    * Append a random suffix (1-10): `PK: Bieber_1`, `PK: Bieber_2` ... `PK: Bieber_10`.
    * Distributes the writes across 10 partitions.
    * **Read:** Application must query all 10 keys (`Bieber_1`...`10`) and merge results (Scatter-Gather).

### Scenario B: Counting Items (Aggregation)
**Interviewer:** *"How do I run `SELECT COUNT(*) FROM Orders` efficiently?"*

* **The Trap:** `Scan` acts like `COUNT`, but it reads every item and costs $$$$.
* ✅ **Senior Answer:** "**DynamoDB Streams + Aggregation Table.**"
    1.  Enable **DynamoDB Streams** on the `Orders` table.
    2.  Trigger a **Lambda** function on every insert.
    3.  Lambda increments a counter in a separate `Stats` table (`PK: Stats, Count: +1`).
    4.  Read the count from `Stats` table in $O(1)$.

### Scenario C: Filtering Data
**Interviewer:** *"I want to find all 'Active' Users. Should I use `Scan` with a `FilterExpression`?"*

* ✅ **Senior Answer:** "**No. Filters are applied AFTER the read.**"
    * If you Scan 1M users and filter for 10 Active ones, you pay for reading 1M items.
    * **Solution:** Sparse Index (GSI).
    * Create a GSI where Partition Key is `IsActive`.
    * If `IsActive` is null (Inactive), DynamoDB **does not index the item** in the GSI.
    * The GSI only contains Active users. Scanning the GSI is efficient.

### Scenario D: Time To Live (TTL)
**Interviewer:** *"We store session data. How do we delete old sessions?"*

* ❌ **Bad Answer:** "Write a script to delete old rows." (Costs WCUs).
* ✅ **Senior Answer:** "**Enable Native TTL.**"
    * Add a `expiry_timestamp` attribute.
    * Enable TTL on that attribute.
    * DynamoDB deletes expired items in the background for **free** (doesn't consume WCUs).

---

### **Final Checklist**
1.  **Modeling:** Access Patterns > Data normalization.
2.  **Architecture:** Single Table Design (PK/SK overloading).
3.  **Scale:** GSI over LSI.
4.  **Events:** DynamoDB Streams for aggregation.

**This concludes the DynamoDB Playbook.**