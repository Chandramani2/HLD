# 🍃 The Senior MongoDB Playbook

> **Target Audience:** Senior Backend Engineers & Data Architects  
> **Goal:** Master **Schema Design Patterns**, **Sharding Strategies**, and **Consistency Tuning**.

In a senior interview, "using Mongo" implies you know how to handle the **16MB Document Limit**, avoid **Hot Shards**, and tune for **CAP Theorem trade-offs**.

---

## 📖 Table of Contents
1. [Part 1: Architecture (Replica Sets vs. Sharding)](#-part-1-architecture-replica-sets-vs-sharding)
2. [Part 2: Internals (WiredTiger & The Journal)](#-part-2-internals-wiredtiger--the-journal)
3. [Part 3: Schema Design (Embed vs. Reference)](#-part-3-schema-design-embed-vs-reference)
4. [Part 4: Consistency & Durability (Write Concerns)](#-part-4-consistency--durability-write-concerns)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: Architecture (Replica Sets vs. Sharding)

MongoDB scales in two ways. Know the difference.

### 1. Replica Sets (High Availability)
* **Structure:** 1 Primary (RW), N Secondaries (Read-Only).
* **Failover:** Automatic. If Primary dies, Secondaries hold an election.
* **Oplog (Operations Log):** The "Stream."
    * Primary writes changes to its Oplog.
    * Secondaries tail the Oplog and apply changes asynchronously.
    * *Senior Note:* If the Oplog is too small (fixed size), and a Secondary falls too far behind, it becomes "Stale" and requires a full resync (Initial Sync).

### 2. Sharded Cluster (Horizontal Scaling)
When data > 2TB or writes > 20k/sec, Replica Sets hit a wall.
* **Mongos (The Router):** Acts as a load balancer. App connects here. It routes queries to the correct shard.
* **Config Servers:** Store the metadata (Mapping of `Chunk Range -> Shard ID`).
* **Shards:** Individual Replica Sets that hold a subset of the data.
* **The Balancer:** A background process that moves chunks of data between shards to keep them even.



---

## ⚙️ Part 2: Internals (WiredTiger & The Journal)

Since version 3.2, **WiredTiger** is the default storage engine.

### 1. Document Level Locking
* **Old (MMAPv1):** Had Collection-level locking. One write blocked the whole table.
* **New (WiredTiger):** Uses **Document-level locking** and **MVCC** (Multi-Version Concurrency Control).
    * Optimistic Concurrency: Multiple threads can modify the collection simultaneously. Conflicts are retried.

### 2. Compression
* WiredTiger compresses data on disk (Snappy/Zlib) and in RAM (Prefix Compression).
* *Result:* 100GB of JSON often takes only 30GB on disk.

### 3. The Journal (Write-Ahead Log)
* Before writing to the data files (Checkpoints), Mongo writes to the **Journal** on disk.
* **Purpose:** Crash recovery. If the server loses power, Mongo replays the Journal on startup to recover RAM data that hadn't been flushed to disk yet.

---

## 📝 Part 3: Schema Design (Embed vs. Reference)

This is the "Art" of NoSQL. There is no normal form.

### 1. Embedding (Denormalization)
Store related data in a single document.

```json
// User Document
{ 
  "name": "John", 
  "addresses": [ 
    { "city": "NY" }, 
    { "city": "SF" } 
  ] 
}
```

* **Pros:** **Data Locality**. 1 Read fetches everything. Atomic updates.
* **Cons:** **The 16MB Limit.** If "Addresses" grows unbounded, the document fails.
* **Rule:** Use for "One-to-Few".

### 2. Referencing (Normalization)
Store IDs and join later.

```json
// User Doc
{ "name": "John", "_id": 1 }

// Address Doc
{ "city": "NY", "user_id": 1 }
```

* **Pros:** Unlimited growth. Smaller documents.
* **Cons:** Requires `$lookup` (Left Join) in aggregation, which is slower.
* **Rule:** Use for "One-to-Many" or "Many-to-Many".

### 3. The Bucket Pattern (IoT / Time-Series)
Instead of 1 document per second, group them.
* **Bad:** 1 Million documents for 1 million temperature readings. (Index size explodes).
* **Good (Bucket):** 1 Document per *Hour*.

```json
{
  "sensor_id": 123,
  "date": "2023-10-01",
  "readings": [ 
    { "time": "10:01", "temp": 20 },
    { "time": "10:02", "temp": 21 }
  ]
}
```

* **Result:** Reduces index size by 100x.

---

## 🛡️ Part 4: Consistency & Durability (Write Concerns)

You can tune Mongo to be as safe as Postgres or as fast as Redis.

### 1. Write Concern (`w`)
How many nodes must acknowledge the write?
* **`w: 0` (Fire & Forget):** Don't wait for ack. Fastest. Unsafe.
* **`w: 1` (Default):** Wait for Primary to Ack. (Safe from network errors, risky if Primary crashes immediately).
* **`w: majority`:** Wait for Primary + Majority of Secondaries. **Data is practically durable.** Slower.

### 2. Read Preference
* **`primary`:** Always read from Master. Strong Consistency.
* **`secondaryPreferred`:** Read from Slave. Eventual Consistency. Good for analytics/reporting.

### 3. Read Concern (`isolation`)
* **`local`:** Return data immediately. It might be rolled back later.
* **`majority`:** Return data only if it has been replicated to a majority (guaranteed not to roll back).

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Hot Shard" Problem
**Interviewer:** *"We sharded by `CreationDate`. Now the newest shard is at 100% CPU and the others are idle."*

* **The Problem:** **Monotonically Increasing Keys**. All new writes ("Today") go to the chunk range ending in `Infinity`. That chunk lives on one specific shard.
* ✅ **Senior Answer:** "**Hashed Sharding.**"
    * Don't shard on `Date`. Shard on `Hashed(Date)`.
    * This randomizes the distribution. Write 1 goes to Shard A, Write 2 goes to Shard B.
    * **Trade-off:** Range queries (`Find dates between X and Y`) become Scatter-Gather (query all shards), which is slower.

### Scenario B: Pagination at Scale
**Interviewer:** *"Skip/Limit is getting slow as the user goes deeper into pages."*

* **The Reason:** `skip(100000)` requires the DB to scan and throw away 100k docs.
* ✅ **Senior Answer:** "**Keyset Pagination (The Bucket Pattern helps here too).**"
    * Use `_id` or a sorted field.
    * `find({ _id: { $gt: last_seen_id } }).limit(10)`.
    * Uses the Index to jump instantly.

### Scenario C: ACID Transactions
**Interviewer:** *"We need to update Inventory and Order History atomically. Should we use Two-Phase Commit?"*

* ✅ **Senior Answer:** "**Native Multi-Document Transactions (v4.0+).**"
    * MongoDB now supports ACID transactions across multiple documents (and shards).
    * `session.startTransaction() ... commitTransaction()`.
    * **Caveat:** They add performance overhead. Use them only for critical financial logic, not for every write.

### Scenario D: Optimizing Indexes (ESR Rule)
**Interviewer:** *"How do you order the fields in a compound index?"*

* ✅ **Senior Answer:** "**The ESR Rule.**"
    1.  **E (Equality):** Fields you run exact matches on (`status: "active"`).
    2.  **S (Sort):** Fields you sort by (`date: -1`).
    3.  **R (Range):** Fields you filter by range (`price > 10`).
    * *Index:* `{ status: 1, date: -1, price: 1 }`.
    * If you put Range before Sort, Mongo cannot use the index for sorting.