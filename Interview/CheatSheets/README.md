# System Design Interview Ultimate Cheat Sheet

This document serves as a quick-reference guide for system design interviews (SDI). It covers the standard framework, essential estimation numbers, core concepts, and technology choices.

---

## 1. The 5-Step Framework
*Time Management: 45 Minutes Total*

| Step | Time | Goal | Key Actions |
| :--- | :--- | :--- | :--- |
| **1. Requirements** | 5m | Limit Scope | Functional (Features), Non-Functional (Scale, Latency, CAP). |
| **2. Estimations** | 5m | Feasibility | Calculate Traffic (RPS), Storage (TB/yr), Bandwidth. |
| **3. High-Level Design** | 10m | Blueprint | Draw the "30,000ft view" (Client $\to$ LB $\to$ Server $\to$ DB). |
| **4. Deep Dive** | 20m | Specifics | Scale specific components (Sharding, Caching, API Design). |
| **5. Bottlenecks** | 5m | Critique | Identify SPOFs (Single Points of Failure), Hotspots, Monitoring. |

---

## 2. Back-of-the-Envelope "Magic Numbers"
*Memorize these to perform estimations quickly.*

### Time & Volume
* **Seconds in a Day:** $\approx 86,400$ (Round to **$10^5$** for math).
* **Requests per Second (RPS):**
    * 1 Million DAU $\approx$ 10 RPS (very low).
    * 10 Million DAU $\approx$ 100 RPS.
    * **Rule of Thumb:** 1M req/day $\approx$ 12 RPS.

### Data Sizes
* **char:** 2 Bytes (Unicode) / 1 Byte (ASCII)
* **int / long:** 4 Bytes / 8 Bytes
* **UUID:** 16 Bytes (standard)
* **Image:** ~200 KB (compressed)
* **Video:** ~50-100 MB per minute (HD)

### Latency Comparison (The "Powers of 10")
* **L1 Cache Ref:** 0.5 ns
* **RAM Ref:** 100 ns (200x slower than L1)
* **SSD Read:** 100 $\mu$s ($100,000$ ns)
* **HDD Seek:** 10 ms ($10,000,000$ ns - **Avoid random disk seeks!**)
* **Packet CA $\to$ Netherlands:** 150 ms

---

## 3. Core Concepts & Trade-offs

### A. Scalability
* **Vertical Scaling (Scale Up):** Bigger server (RAM/CPU). Limit: Cost/Hardware cap. Easy setup.
* **Horizontal Scaling (Scale Out):** More servers. Infinite scale. Complexity: Load balancing, data consistency.

### B. CAP Theorem
* *In a Distributed System, you can only pick 2:*
    1.  **Consistency:** Every read receives the most recent write or an error.
    2.  **Availability:** Every request receives a (non-error) response, without the guarantee that it contains the most recent write.
    3.  **Partition Tolerance:** The system continues to operate despite an arbitrary number of messages being dropped/delayed by the network.
* **Reality:** P is mandatory. You choose **CP** (Banking) or **AP** (Twitter/Social Media).

### C. Database Scaling
* **Replication (Master-Slave):**
    * **Master:** Handles Writes.
    * **Slaves:** Handle Reads.
    * *Trade-off:* Eventual Consistency (Slave lag).
* **Sharding (Partitioning):**
    * Splitting data across multiple servers.
    * **Vertical Sharding:** By table (User DB, Tweet DB).
    * **Horizontal Sharding:** By row (Users A-M on Server 1, N-Z on Server 2).
    * *Algo:* **Consistent Hashing** (Prevents total reshuffling when a node dies).

---

## 4. The "Holy Grail" Architecture Diagram

This generic architecture solves 90% of web-scale problems.

```mermaid
flowchart TD
    Client[Client App] --> CDN[CDN (Static Assets)]
    Client --> DNS[DNS]
    Client --> LB[Load Balancer]
    
    LB --> Service[Stateless Web Servers]
    
    Service --> Cache[Distributed Cache (Redis)]
    Service --> DB_Master[(DB Master)]
    
    subgraph Data Layer
        DB_Master -.-> DB_Slave[(DB Read Replicas)]
        DB_Master -.-> MQ[Message Queue (Kafka)]
    end
    
    MQ --> Workers[Async Workers]
    Workers --> Archive[Object Storage (S3)]
```

---

## 5. Technology "Dictionary" (When to use what)

| Component | Technology Examples | Use Case |
| :--- | :--- | :--- |
| **Load Balancer** | Nginx, HAProxy, AWS ELB | Distributing traffic, SSL termination. |
| **Relational DB (SQL)** | PostgreSQL, MySQL | Structured data, ACID transactions (Payments, User Auth). |
| **NoSQL (Columnar)** | Cassandra, HBase | Massive write volume, Time-series data (Chat logs, Metrics). |
| **NoSQL (Document)** | MongoDB, DynamoDB | Flexible schema, JSON blobs (Product catalog). |
| **Cache** | Redis, Memcached | Key-Value lookups, Counters, Leaderboards. |
| **Message Queue** | Kafka, RabbitMQ | Decoupling services, Peak shaving, Async processing. |
| **Object Storage** | AWS S3, Google GCS | Large files (Images, Videos, Backups). |
| **Search Engine** | Elasticsearch, Solr | Full-text search, Fuzzy matching. |

---

## 6. Common Design Patterns

### A. Caching Strategies
* **Cache-Aside (Lazy Loading):** App checks Cache. If miss, App reads DB $\to$ writes to Cache. (Most common).
* **Write-Through:** App writes to Cache & DB simultaneously. (Safe but slow writes).
* **Eviction Policies:**
    * **LRU (Least Recently Used):** Discard items not used for the longest time.
    * **LFU (Least Frequently Used):** Discard items used least often.

### B. Unique ID Generation
* **Auto-increment (SQL):** Simple, but hard to shard.
* **UUID:** Simple, unique, but huge (16 bytes) and not sortable.
* **Snowflake (Twitter):** Timestamp + MachineID + Sequence. Sortable, unique, fits in 64-bit integer.

### C. Rate Limiting Algorithms
* **Token Bucket:** Allow "bursts" of traffic up to a limit.
* **Leaky Bucket:** Smooths out bursts to a constant rate.
* **Fixed Window Counter:** Simple, but edge-case issues at window boundaries.

---

## 7. Key Bottleneck Solutions

| Symptom | Potential Solution |
| :--- | :--- |
| **High Read Latency** | Add Cache (Redis), Add DB Read Replicas. |
| **High Write Latency** | Sharding, Use NoSQL (Cassandra), Async Queue (Kafka). |
| **Static Content Slow** | Use CDN (Cloudfront). |
| **Single Point of Failure** | Add Redundancy (Active-Passive), Heartbeat monitoring. |
| **Database Overload** | Implement Caching, Sharding, or separate Read/Write paths (CQRS). |