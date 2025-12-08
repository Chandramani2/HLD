# 🗄️ The Senior Database Engineering Playbook

> **Target Audience:** Senior Backend Engineers, Data Architects  
> **Goal:** Move beyond "How to write SQL" to "How to store petabytes of data reliably."

In senior interviews, the database is often the **bottleneck**. You must demonstrate deep knowledge of **storage internals**, **consistency models**, and **polyglot persistence**.

---

## 📖 Table of Contents
1. [Part 1: The Great Divide (SQL vs. NoSQL)](#-part-1-the-great-divide-sql-vs-nosql)
2. [Part 2: Database Internals (Indices & Storage)](#-part-2-database-internals-what-happens-on-disk)
3. [Part 3: The NoSQL Taxonomy](#-part-3-the-nosql-taxonomy-choosing-the-right-tool)
4. [Part 4: Scaling Strategies (Sharding & Replication)](#-part-4-scaling-strategies)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## ⚔️ Part 1: The Great Divide (SQL vs. NoSQL)

It is not just about "Relational vs. Non-Relational." It is about **ACID vs. BASE**.

### 1. RDBMS (SQL) - The Systems of Record
* **Structure:** Tables, Rows, Columns, Foreign Keys.
* **Schema:** Strict (must define types before inserting).
* **Philosophy:** **ACID**.
    * **A**tomicity: All or nothing.
    * **C**onsistency: Data is valid according to rules.
    * **I**solation: Transactions don't interfere with each other.
    * **D**urability: Once committed, it's saved forever (WAL - Write Ahead Log).
* **Use Cases:** Financial ledgers, Inventory management, Identity.

### 2. NoSQL - The Systems of Scale
* **Structure:** Documents, Key-Values, Graphs, Wide-Columns.
* **Schema:** Flexible (Schema-on-read).
* **Philosophy:** **BASE** (Eventual Consistency).
    * **B**asically **A**vailable: Guaranteed availability (CAP theorem).
    * **S**oft state: State may change over time without input.
    * **E**ventual consistency: The system will become consistent... eventually.
* **Use Cases:** Social feeds, IoT sensor data, Caching, Recommendations.

---

## ⚙️ Part 2: Database Internals (What happens on disk?)

Seniors must understand *why* a database is slow. Usually, it's the data structure on the disk.

### 1. Indexing Strategies
* **B-Trees (B+ Trees):** Used by MySQL, Postgres, Oracle.
    * *Pros:* Great for **Read-Heavy** workloads and Range Scans (`SELECT * WHERE age > 20`).
    * *Cons:* Updates are random I/O (slow for massive write ingestion).
* **LSM Trees (Log-Structured Merge Trees):** Used by Cassandra, RocksDB, Kafka.
    * *Mechanism:* Writes go to memory (MemTable), then flush to disk sequentially (SSTable). Background compaction merges files.
    * *Pros:* Extremely fast **Write** throughput (Sequential I/O).
    * *Cons:* Slower reads (might need to check multiple files).

[Image of B-Tree vs LSM Tree structure]

### 2. Row vs. Column Oriented Storage
* **Row-Oriented (Postgres, MySQL):** Stores data line-by-line.
    * *Best for:* **OLTP** (Online Transaction Processing). "Fetch user X's profile."
* **Column-Oriented (Redshift, Snowflake, ClickHouse):** Stores data column-by-column.
    * *Best for:* **OLAP** (Online Analytical Processing). "Calculate the average age of all users." (Only reads the 'Age' block, ignores the rest).

[Image of Row oriented vs Column oriented storage diagram]

---

## 🛠️ Part 3: The NoSQL Taxonomy (Choosing the Right Tool)

"NoSQL" is a bucket term. You must be specific.

| Type | Examples | Best Use Case | Senior Note |
| :--- | :--- | :--- | :--- |
| **Key-Value** | Redis, DynamoDB | Caching, Sessions, Shopping Carts. | Extremely fast (O(1)). Cannot query by value (only by key). |
| **Document** | MongoDB, CouchDB | CMS, User Profiles, Catalogs. | Flexible schema. Avoid if you need complex multi-document transactions. |
| **Wide-Column** | Cassandra, HBase | Time-series, IoT logs, Chat History. | optimized for writes. **Query patterns determine Schema.** |
| **Graph** | Neo4j, Amazon Neptune | Social Networks, Fraud Detection. | Efficient for "Friends of Friends" queries (Graph Traversal). |

---

## 📈 Part 4: Scaling Strategies

How do we handle 100TB of data?

### 1. Replication (Read Scaling)
Copies data to multiple nodes.
* **Master-Slave:** All writes go to Master. Slaves replicate logs.
    * *Risk:* Replication Lag. A user writes a post, refreshes page, and it's not there yet (read from a stale slave).
* **Master-Master:** Write to any node.
    * *Risk:* Write conflicts (Split Brain). Complex resolution logic needed.

### 2. Sharding (Write Scaling)
Splitting data horizontally across multiple servers.
* **Sharding Key:** The attribute used to route data (e.g., `UserID`).
* **The Problem: Hot Partitions.**
    * If you shard by `Celebrity_ID` and Justin Bieber tweets, one shard melts down while others are idle.
* **Solution:** Consistent Hashing or careful key selection.

[Image of Database Sharding Architecture]

### 3. Federation
Splitting tables by function into different DBs (e.g., `Users` DB, `Products` DB, `Forums` DB).
* *Pros:* Simple way to reduce load.
* *Cons:* No cross-database joins.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Heavy Write" Problem
**Interviewer:** *"We need to store 1 million IoT sensor events per second. We need to query them by time range later. What DB do you use?"*

* ❌ **Bad Answer:** "MySQL with a big server." (B-Trees cannot handle this write volume).
* ⚠️ **Mid Answer:** "MongoDB." (Better, but locks might become an issue at this scale).
* ✅ **Senior Answer:** "Use a **Wide-Column Store like Cassandra** or a Time-Series DB like **InfluxDB**."
    * **Why?** They use **LSM Trees** (Sequential writes).
    * **Schema Design:** Partition Key = `SensorID + DayBucket`, Clustering Key = `Timestamp`.
    * **Trade-off:** We accept eventual consistency for massive write throughput.

### Scenario B: The "Complex Search" Problem
**Interviewer:** *"We have a product catalog in Postgres. Users want to search by fuzzy text, category, and price range. Postgres `LIKE` queries are too slow."*

* ❌ **Bad Answer:** "Add more indexes to Postgres." (Too many indexes slow down writes; full-text search in SQL is limited).
* ✅ **Senior Answer:** "**Polyglot Persistence.**"
    * Keep the "Source of Truth" in Postgres.
    * Stream changes (CDC - Change Data Capture) to **Elasticsearch**.
    * Direct search queries to Elasticsearch (Inverted Index) for speed.
    * Direct checkout/payment queries to Postgres for ACID safety.

### Scenario C: The "Social Graph" Problem
**Interviewer:** *"Design the 'People You May Know' feature for Facebook."*

* ❌ **Bad Answer:** "A SQL table `Friendships` with `user_id_1` and `user_id_2`. Do a self-join."
    * *Why it fails:* As friends-of-friends depth increases (2nd/3rd degree), SQL joins become exponential ($O(N^k)$).
* ✅ **Senior Answer:** "Use a **Graph Database (Neo4j)**."
    * Nodes = People. Edges = Friendships.
    * Graph DBs use "Index-Free Adjacency" (pointers), making traversal constant time regardless of total data size.

---

## 📝 Part 6: The "ACID" vs "CAP" Decision Matrix

When asked to choose a DB, draw this mental matrix:

1.  **Is it Financial/Billing?** -> **SQL** (Postgres/MySQL) -> Needs ACID.
2.  **Is it Logs/Metrics/IoT?** -> **NoSQL Columnar** (Cassandra) -> Needs Write Speed.
3.  **Is it a Social Network?** -> **Graph DB** (Neo4j) -> Needs Relationships.
4.  **Is it a Shopping Cart/Session?** -> **Key-Value** (Redis) -> Needs Speed/TTL.
5.  **Is it a Product Catalog?** -> **Document** (MongoDB) -> Needs Flexible Schema.