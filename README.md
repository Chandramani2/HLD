# 🏛️ The Senior High-Level Design (HLD) Playbook

> **Target Audience:** Senior Software Engineers & System Architects  
> **Goal:** Move beyond definitions to trade-offs, bottleneck analysis, and failure handling.

High-Level Design (HLD) interviews for senior roles are less about "knowing the definition" and more about **trade-offs**, **bottleneck analysis**, and **failure handling**. A senior engineer must demonstrate the ability to take vague requirements and convert them into a scalable, fault-tolerant architecture.

---

## 📖 Table of Contents
1. [Part 1: The Foundation](#-part-1-the-foundation-basic-to-intermediate)
2. [Part 2: Advanced Distributed Concepts](#-part-2-advanced-distributed-concepts-senior-level)
3. [Part 3: Senior Level Q&A Scenarios](#-part-3-senior-level-qa-scenarios)
4. [Part 4: The Interview Framework](#-part-4-the-senior-interview-framework)

---

## 🧱 Part 1: The Foundation (Basic to Intermediate)

Before designing complex systems, you must master the building blocks.

### 1. Scalability Strategies
The first decision in any design: Do we build bigger or wider?

| Strategy | Description | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Vertical (Scale Up)** | Adding more power (CPU, RAM) to an existing server. | Simple; No code changes required. | Hard hardware limit; Single point of failure. |
| **Horizontal (Scale Out)** | Adding more servers to the pool. | Theoretically infinite scaling; Redundancy. | Complexity; Requires LB; Data consistency issues. |

### 2. Load Balancing
Distributes incoming network traffic across multiple servers to ensure no single server bears too much load.

* **Layer 4 (Transport Layer):** Balances based on IP address and Port (TCP/UDP). It is faster but has less context.
* **Layer 7 (Application Layer):** Balances based on URL, Cookies, or Headers. It allows for "Smart Routing" (e.g., routing `/payment` requests to high-security servers).

### 3. Caching Patterns
Caching is the primary tool for reducing latency and database load.

* **Cache-Aside:** The App checks the cache. If `miss`, it reads the DB and updates the cache. *(Most common)*.
* **Write-Through:** App writes to Cache and DB simultaneously. *(High data consistency, slower write)*.
* **Write-Back:** App writes to Cache only; Cache asynchronously writes to DB. *(Fastest write, risk of data loss on crash)*.

### 4. Database Scaling
* **Replication (Master-Slave):** Master handles **Writes**, Slaves handle **Reads**.
    * *Bottleneck:* If the Master goes down, you need complex failover logic.
* **Sharding:** Splitting data across multiple machines based on a `Partition Key` (e.g., UserID).
    * *Senior Trade-off:* Cross-shard transactions (Joins) become extremely expensive or impossible.

---

## 🚀 Part 2: Advanced Distributed Concepts (Senior Level)

This is where the interview is won or lost. These concepts address specific problems in massive-scale systems.

### 1. CAP Theorem & PACELC
You cannot achieve **Consistency**, **Availability**, and **Partition Tolerance** simultaneously. You must pick two.

* **CP (Consistency + Partition Tolerance):** Banking systems. *(Better to return an error than show the wrong balance).*
* **AP (Availability + Partition Tolerance):** Social media feeds. *(Better to show an old post than a "Site Down" error).*

> **💡 The Senior Upgrade: PACELC** > The CAP theorem only applies when there is a partition. **PACELC** states:
> * **If Partition (P):** Choose A or C.
> * **Else (E):** Choose **Latency (L)** or **Consistency (C)**.
> * *Example:* DynamoDB allows you to choose between strong consistency (slower) or eventual consistency (faster).

### 2. Consistent Hashing
In standard hashing (`key % n`), adding a new server requires remapping almost all keys. In Consistent Hashing, only `k/n` keys need remapping.

* **Use Case:** Distributed Caches (DynamoDB, Cassandra, Discord Ring).
* **Virtual Nodes:** Used to ensure even distribution of data on the ring to prevent "hot spots."

### 3. Bloom Filters
A space-efficient probabilistic data structure used to test whether an element is a member of a set.

* **The Rule:** False positives are **possible**, but false negatives are **impossible**.
* **Use Cases:**
    * *Databases:* "Does this row exist in this disk file?" (Avoids expensive disk seeks).
    * *Usernames:* "Is this username already taken?"

### 4. Distributed Transactions (Saga Pattern)
Microservices break ACID properties. A transaction spanning multiple services cannot rely on standard DB locks.

* **2-Phase Commit (2PC):** Strict consistency but blocking and slow. Not recommended for high scale.
* **Saga Pattern:** A sequence of local transactions. If one fails, a series of **Compensating Transactions** are triggered to undo changes.
    * *Example:* If `Payment` fails, trigger `Release_Reserved_Seat` in the Booking Service.

### 5. Idempotency
Ensuring that performing an operation multiple times has the same result as performing it once.

* **Critical For:** Payments & Messaging.
* **Implementation:** Client generates a unique `idempotency_key` (UUID). Server checks if this key was already processed before executing logic.

---

## 🧠 Part 3: Senior Level Q&A Scenarios

At a senior level, avoid "textbook" answers. Use the context to drive decisions.

### Scenario A: Design a Unique ID Generator (e.g., Twitter Snowflake)
**Interviewer:** *"We need a distributed ID generator. It must be unique, numerical, and sortable by time. We cannot use a central database."*

* ❌ **Bad Answer:** "Use UUID." (Not sortable, 128-bit is too heavy for primary indexing).
* ✅ **Senior Answer (Snowflake Approach):**

    * **Proposal:** Design a custom 64-bit integer ID.
    * **Bit Layout:**
        * `1 bit`: Sign bit (unused).
        * `41 bits`: Timestamp (milliseconds). Gives ~69 years of IDs.
        * `10 bits`: Machine ID (datacenter ID + worker ID).
        * `12 bits`: Sequence number (resets every millisecond).
    * **Why?** Allows generating IDs locally without network coordination (fast), and they are `k`-sortable by time.

### Scenario B: Real-time Leaderboard (Top K Problem)
**Interviewer:** *"How do we display the top 10 gamers in real-time? We have 10 million active users."*

* ❌ **Bad Answer:** SQL `ORDER BY score DESC LIMIT 10`. (Too slow on updates/reads).
* ⚠️ **Mid Answer:** "Use a standard Redis Sorted Set." (Works for small scale, but 10M users on one Redis instance is a bottleneck).
* ✅ **Senior Answer (Partitioning):**

    * **Challenge:** Single Redis instance memory/CPU limits.
    * **Solution: Sharded Leaderboards.**
        1.  Split users into `N` partitions (e.g., `User_ID % 10`).
        2.  Each partition maintains its own Top 10 using Redis ZSET.
        3.  A "Scatter-Gather" service queries all 10 partitions, aggregates the results, and calculates the Global Top 10.

### Scenario C: The "Thundering Herd" (Cache Stampede)
**Interviewer:** *"A celebrity tweets. 10 million users request their profile simultaneously. The cache expires at that exact moment. What happens?"*

* **The Disaster:** 10M requests miss cache -> 10M requests hit DB -> DB crashes.
* ✅ **Senior Solutions:**
    1.  **Request Coalescing:** The server identifies that 10M people are asking for key `A`. It holds 9,999,999 requests, sends **one** request to the DB, updates the cache, and then serves the holding requests.
    2.  **Probabilistic Early Expiration:** If TTL is 60s, users fetch the new value at 55s based on a random probability ("Gap jitter"), refreshing it *before* it actually expires.

---

## 📝 Part 4: The Senior Interview Framework

When asked to "Design X", follow this structure:

1.  **Requirements Clarification:** "Is this read-heavy or write-heavy? What is the expected latency?"
2.  **Back-of-Envelope Math:** "If we have 1M DAU, and each posts 1KB, we need X storage per year."
3.  **High-Level Diagram:** Draw the boxes (Client, LB, Service, DB).
4.  **Deep Dive & Bottlenecks:** "The DB is the bottleneck here, let's add Sharding."
5.  **Failure Analysis:** "What happens if the Message Queue goes down? We need a Dead Letter Queue."