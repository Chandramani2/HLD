# Database Selection Guide for System Design Interviews

Choosing the right database is one of the highest-signal decisions in a system design interview. It demonstrates your understanding of data access patterns, consistency requirements (CAP theorem), and scalability trade-offs.

This guide outlines which databases to use and when, structured by category.

---

## 1. Relational Databases (RDBMS)
**Examples:** PostgreSQL, MySQL, Oracle.

**When to use:** This should be your **default choice** unless you have a specific reason to switch (e.g., massive scale or flexible schema).

* **Structured Data:** The data has a strict schema (rows and columns) that won't change often.
* **ACID Compliance:** You need strict data integrity. If a transaction fails, the entire operation must roll back (Atomicity).
* **Complex Relationships:** You need to join multiple tables (e.g., `Users` joined with `Orders`).

| Scenario | Why? |
| :--- | :--- |
| **Banking / Payments** | Strong consistency is non-negotiable. You cannot lose money or double-count it. |
| **User Auth / Profiles** | User data is structured and rarely changes schema. |
| **Inventory Management** | Requires strict counting and locking mechanisms to prevent overselling. |

---

## 2. NoSQL: Key-Value Stores
**Examples:** Redis, Memcached, DynamoDB.
**Data Model:** Simple Hash Map (Key $\rightarrow$ Value).

**When to use:** You need extreme speed (low latency) and the data model is simple (dictionary style).

* **Caching:** Storing frequently accessed data in memory to reduce load on the main DB.
* **Low Latency is critical:** You need sub-millisecond response times.
* **Simple Access Patterns:** You only need to fetch data by one primary key (e.g., `get(userID)`).
* **Transient Data:** Data that can be lost (cache) or expires (TTL)


| Scenario            | Why? |
|:--------------------| :--- |
| **Caching (Redis)** | Storing the result of expensive queries (e.g., "Top 10 News Feeds"). |
| **Sessions**        | Storing user login tokens/session IDs for quick validation. |
| **Leaderboards**    | Redis Sorted Sets are $O(\log N)$ for ranking millions of users. |
| **Shopping Carts**  | Fast read/write access is needed while the user browses; persistence is secondary until checkout. |
| **Rate Limiting**   | Counting requests per user in a time window (Redis `INCR`). |

> **Pro Tip:** If you need persistence (data saved to disk), choose **Redis** or **DynamoDB**. If you only need a pure in-memory cache that disappears on restart, **Memcached** is acceptable (though Redis is generally preferred now).

---

## 3. NoSQL: Document Stores
**Examples:** MongoDB, Couchbase.
**Data Model:** JSON-like documents. Flexible schema.

**When to use:** You have flexible schemas or hierarchical data (JSON-like).

* **Rapid Prototyping:** You don't know the final schema yet.
* **Unstructured Objects:** Different items have different attributes (e.g., a "Product" table where a *Laptop* has `CPU/RAM` but a *Shirt* has `Size/Fabric`).
* **Complex Objects:** You want to store a "User" and their "Addresses" and "Contacts" in a single document (Aggregates) rather than joining 3 tables.
* **Rapid Prototyping:** You are building an MVP and don't want to run database migrations every day.

| Scenario | Why? |
| :--- | :--- |
| **Content Management (CMS)** | Articles/Blogs have varying structures (tags, nested comments, metadata). |
| **Product Catalogs** | E-commerce items vary wildly in attributes; JSON storage fits perfectly. |
| **User Settings** | Flexible JSON blobs for user preferences that might change often. |

---

## 4. NoSQL: Wide-Column Stores
**Examples:** Cassandra, HBase, ScyllaDB.
**Data Model:** A two-dimensional key-value store, where each row can have a massive number of columns. Optimized for writing.

**When to use:** You have **massive** amounts of data (Petabytes) and high **write** throughput.

* **Write-Heavy Workloads:** These DBs are optimized for writes (appending to log files).
* **High Availability:** They run on distributed clusters with no single point of failure (Leaderless replication).

| Scenario | Why? |
| :--- | :--- |
| **Chat History (WhatsApp)** | Billions of messages written daily. Users only fetch recent chats (partitioned by `ChatID`). |
| **IoT Sensor Logs** | Millions of devices sending temperature/heartbeat data every second. |
| **User Activity Logs** | Storing every click or view for analytics. |
| **Time-Series Data** | Storing history of stock prices or metrics (though specialized Time-Series DBs exist, Cassandra is often used here). |

> **Pro Tip:** Cassandra keys are tricky. You can **only** query by the partition key. You cannot easily do "Select * where age > 25". Choose this only if you know your query patterns in advance.
---

## 5. NoSQL: Graph Databases
**Examples:** Neo4j, Amazon Neptune.
**Data Model:** Nodes (entities) and Edges (relationships).

**When to use:** The **relationships** between data are more important than the data itself.

* **Many-to-Many Relationships:** SQL struggles with "friends of friends of friends" queries (too many JOINs). Graph DBs traverse these links in $O(1)$ time.

| Scenario                                                           | Why?                                                            |
|:-------------------------------------------------------------------|:----------------------------------------------------------------|
| **Social Networks**                                                | "People you may know" features rely on graph traversal.         |
| **Recommendation Engines**                                         | "Users who bought X also bought Y."                             |
| **Fraud Detection**                                                | Detecting circular money transfers or device fingerprint rings. |
| **Knowledge Graphs**                                               |  Linking facts (e.g., Google Search sidebar). |
---

## 6. Specialized Storage
These are usually "secondary" databases used alongside one of the primary ones above.

### A. Blob Storage
**Examples:** Amazon S3, Google Cloud Storage (GCS).
* **Use for:** Images, Videos, Large Files, Backups.
* **Tip:** Never store images in the database (BLOB type). Store the image in S3 and save the *URL* in your SQL/NoSQL DB.

### B. Search Engines
**Examples:** Elasticsearch, Solr.
**Data Model:** Inverted Index (like the index at the back of a book).

* **Use for:** Fuzzy search, Autocomplete, Log analysis.
* * **Full-Text Search:** Users need to type keywords and get results (fuzzy matching, typos).
* **Complex Filtering:** "Show me red shirts, size M, under $50, sorted by popularity."

### Interview Scenarios
* **Search Bar:** Any system with a search box (Amazon Product Search, Netflix Movie Search).
* **Log Analysis:** Searching through millions of error logs for a specific error code (ELK Stack).

* **Tip:** If the user needs to "search for a product by name," don't use SQL `LIKE %query%`. Use an inverted index (Elasticsearch).
> **Pro Tip:** Usually, this is a **secondary** database. You store the actual data in SQL/Cassandra (source of truth) and replicate it to Elasticsearch for querying.

---

## 7. Time-Series Databases (TSDB)
**Top Candidates:** InfluxDB, Prometheus, TimescaleDB.
**Data Model:** Optimized for arrays of numbers indexed by time.

### When to use
* **Metrics & Monitoring:** You are storing numbers that change over time.
* **Data Retention Policies:** You need to auto-delete data older than 1 year or "downsample" (aggregate) minute-level data into hour-level data.

### Interview Scenarios
* **System Monitoring:** Designing a service like Datadog or CloudWatch.
* **Stock Market Ticker:** Storing price history.

---

## Summary Table for Quick Reference

| Database Type | Examples | Use Case Trigger | Key Benefit |
| :--- | :--- | :--- | :--- |
| **Key-Value** | Redis, DynamoDB | "Cache", "Session", "Leaderboard" | Ultra-low latency, $O(1)$ access. |
| **Wide-Column** | Cassandra, HBase | "Billions of writes", "Log data", "IoT" | Infinite write scaling, high availability. |
| **Document** | MongoDB | "Catalog", "CMS", "JSON" | Flexible schema, easy for developers. |
| **Graph** | Neo4j | "Social Network", "Recommendations" | Efficient relationship traversal. |
| **Search** | Elasticsearch | "Search bar", "Autocomplete" | Fuzzy search, complex filtering. |
| **Time-Series** | InfluxDB | "Metrics", "Stock prices" | Efficient storage of time-stamped data. |

## Cheat Sheet: Decision Matrix

| If you need... | Use... | Example |
| :--- | :--- | :--- |
| **ACID / Structure** | Relational (SQL) | PostgreSQL |
| **Speed / Caching** | Key-Value | Redis |
| **Flexible Schema** | Document | MongoDB |
| **Massive Writes** | Wide-Column | Cassandra |
| **Complex Relations** | Graph | Neo4j |
| **Search / Text** | Search Engine | Elasticsearch |
| **Images / Video** | Blob Storage | Amazon S3 |