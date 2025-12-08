# 🔴 The Senior Redis Playbook

> **Target Audience:** Backend Engineers & Architects  
> **Goal:** Master **Data Structures**, **Persistence Models**, and **Cluster Sharding**.

In a senior interview, you must explain why running `KEYS *` is a fireable offense, how **Copy-on-Write** works during snapshots, and how to implement **Distributed Locks** safely.

---

## 📖 Table of Contents
1. [Part 1: Architecture (Why Single Threaded?)](#-part-1-architecture-why-single-threaded)
2. [Part 2: Persistence (RDB vs. AOF)](#-part-2-persistence-rdb-vs-aof)
3. [Part 3: Scaling (Sentinel vs. Cluster)](#-part-3-scaling-sentinel-vs-cluster)
4. [Part 4: Advanced Data Structures (Beyond Strings)](#-part-4-advanced-data-structures-beyond-strings)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## ⚡ Part 1: Architecture (Why Single Threaded?)

Redis is primarily single-threaded. Why is this a feature, not a bug?

### 1. The Event Loop (I/O Multiplexing)
* Redis uses **I/O Multiplexing** (epoll/kqueue) to handle thousands of connections on a single thread.
* **Benefit:** No Context Switching. No Race Conditions. No CPU cache invalidation.
* **Speed:** It serves requests purely from RAM (nanoseconds). The CPU is rarely the bottleneck; **Network Bandwidth** is.

### 2. The Danger (Blocking Commands)
* Because there is only 1 thread, **one slow command blocks all other clients.**
* **Never Run:** `KEYS *` (Scans every key. O(N)).
* **Use Instead:** `SCAN` (Cursor-based iteration).

### 3. Threaded I/O (Redis 6.0+)
* Redis 6 introduced multi-threading *only* for network I/O (reading/writing sockets), but the command execution remains single-threaded. This creates a massive performance boost without lock complexity.

---

## 💾 Part 2: Persistence (RDB vs. AOF)

Redis is an In-Memory store. If you pull the plug, data is lost. Unless you configure persistence.

### 1. RDB (Redis Database Snapshot)
* **Mechanism:** Saves a `.rdb` snapshot of the dataset every X minutes.
* **Internals (Copy-on-Write):** Redis forks a child process. The child writes to disk while the parent continues serving traffic.
  * *COW:* Memory is shared. If the parent writes to a key, the OS creates a copy of that page for the parent, leaving the child's view immutable.
* **Pros:** Fast restart (Compact file).
* **Cons:** **Data Loss.** If it crashes, you lose the last 5 minutes of data.

### 2. AOF (Append Only File)
* **Mechanism:** Logs every write command (`SET`, `INCR`) to a file.
* **Internals:** `fsync` strategy.
  * `always`: Slow. Safe.
  * `everysec`: (Default). Buffer flush every second. Max 1s data loss.
* **Pros:** Durable.
* **Cons:** Large file size. Slow restart (must replay millions of commands).

### 3. Hybrid Persistence (The Senior Choice)
* Redis 4.0+ combines them. The AOF file starts with an RDB snapshot, followed by recent AOF logs. Best of both worlds.

---

## ⚖️ Part 3: Scaling (Sentinel vs. Cluster)

### 1. Redis Sentinel (High Availability)
* **Topology:** 1 Master, N Slaves.
* **Role:** Monitoring. If Master dies, Sentinels vote and promote a Slave to Master.
* **Limit:** **Vertical Scaling only.** You cannot store more data than fits in the RAM of *one* machine.

### 2. Redis Cluster (Horizontal Scaling)
* **Topology:** Multi-Master sharding.
* **Slots:** Data is sharded into **16,384 Hash Slots**.
  * `Slot = CRC16(Key) % 16384`.
* **Architecture:** Every node knows the map. If you ask Node A for Key X, and Key X belongs to Node B, Node A returns a `MOVED` error pointing to Node B.
* **Limitations:** Multi-key operations (transactions) are only possible if all keys map to the same hash slot (using `{hashtags}`).



---

## 🛠️ Part 4: Advanced Data Structures (Beyond Strings)

Don't just use `SET/GET`.

| Structure | Best For | Senior Use Case |
| :--- | :--- | :--- |
| **String** | Caching JSON blobs | `SETEX` (Set with TTL) for Sessions. |
| **Hash** | Objects | Storing User Profiles (Field level updates). |
| **List** | Queues | Message Queue (`LPUSH`, `BRPOP`). |
| **Set** | Unordered Unique items | Friends List, Tags. |
| **Sorted Set (ZSET)** | Ranking | **Leaderboards**, Priority Queues. |
| **HyperLogLog** | Counting | Estimating Unique Visitors (99% accurate, 12KB size). |
| **Geo** | Location | `GEORADIUS` (Find drivers within 5km). |
| **Bitmap** | Booleans | User retention (Daily Active Users). |

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The Leaderboard Problem
**Interviewer:** *"Design a real-time leaderboard for a game with 10M players."*

* ❌ **Bad Answer:** "Store score in SQL and `ORDER BY score`." (Too slow).
* ✅ **Senior Answer:** "**Redis Sorted Sets (ZSET).**"
  * `ZADD leaderboard 500 "UserA"` ($O(\log N)$).
  * `ZREVRANGE leaderboard 0 9` (Get Top 10).
  * `ZRANK leaderboard "UserA"` (Get specific user rank).
  * **Scale:** If 10M is too big for one Redis, shard by MMR (Matchmaking Rating) range or partition users into Cohorts.

### Scenario B: Distributed Locking (Critical Section)
**Interviewer:** *"We have a cron job running on 5 servers. Only ONE server should execute it."*

* ✅ **Senior Answer:** "**The Redlock Algorithm (or SETNX with safety).**"
  * **Command:** `SET resource_name my_random_value NX PX 30000`.
    * `NX`: Only set if not exists.
    * `PX`: Expire in 30s (Deadlock protection).
  * **Release:** Do NOT just `DEL`.
    * You must check if the value is *your* random value (ownership check) using a **Lua Script**. This prevents you from deleting a lock acquired by another server if yours timed out.

### Scenario C: OOM (Out of Memory)
**Interviewer:** *"Redis is full. What happens to new writes?"*

* ✅ **Senior Answer:** "**Eviction Policies (`maxmemory-policy`).**"
  * **`noeviction`:** Returns Error (Default). Bad for caches.
  * **`allkeys-lru`:** Deletes least recently used keys (even without TTL). **Best for caching.**
  * **`volatile-lru`:** Deletes LRU keys *that have an expiry set*. Keeps persistent data safe.

### Scenario D: Cache Stampede (Thundering Herd)
**Interviewer:** *"A popular key expires. 10,000 requests hit the DB at once."*

* ✅ **Senior Answer:** "**Probabilistic Early Expiration (Gap Jitter).**"
  * Client side: `if (TTL < random_gap(0-30s)) { recompute_cache() }`.
  * Users refresh the cache *before* it actually expires, preventing the stampede.

### Scenario E: Counting Unique Visitors
**Interviewer:** *"We need to count unique IP addresses for 100M requests per day. We have very little RAM."*

* ❌ **Bad Answer:** "Store IPs in a Set." (100M IPs * 15 bytes = 1.5GB RAM).
* ✅ **Senior Answer:** "**HyperLogLog.**"
  * `PFADD visits 192.168.0.1`.
  * `PFCOUNT visits`.
  * **Result:** Uses fixed **12KB** of memory regardless of count. Error rate ~0.81%.

---

### **Final Checklist**
1.  **Performance:** Avoid `KEYS *`. Use Pipelining for batch jobs.
2.  **Reliability:** RDB for backups, AOF for durability.
3.  **Locks:** Use Lua scripts for atomic lock release.
4.  **Data Types:** Know when to use HyperLogLog and ZSETs.

**This concludes the Redis Playbook.**