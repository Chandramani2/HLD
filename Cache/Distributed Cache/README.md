# ⚡ Distributed Caching: The System Design Guide

> **The Problem:** Databases are slow (Disk I/O). Applications are fast (CPU/RAM). If you fetch the same "Product Details" from the database 1,000 times a second, your database will crash.

**The Solution:** Store frequently accessed data in a fast, in-memory storage (RAM) that is shared across all your servers. This is **Distributed Caching** (e.g., Redis, Memcached).

---

## 1. Local Cache vs. Distributed Cache ⚔️

Before going distributed, understand the difference.

### A. In-Memory (Local) Cache 🏠
Each server stores data in its own RAM (e.g., `HashMap` or Guava in Java).
* **✅ Pros:** Microsecond access (No network call).
* **❌ Cons:** **Inconsistent** (Server A has old price, Server B has new price). **Memory Hog** (Data duplicated on every node).

### B. Distributed Cache 🌐
A separate cluster (Redis) stores the data. All servers talk to it.
* **✅ Pros:** **Consistent** (Single source of truth). **Scalable** (Store TBs of data independent of app servers).
* **❌ Cons:** **Network Latency** (Milliseconds vs Microseconds). Complexity.

---

## 2. Caching Patterns (The "How") 🧠

How do you load data into the cache? There are three main patterns.

### A. Cache-Aside (Lazy Loading) 🐢
**The Standard.** The Application is responsible for talking to the Cache and the DB.

**The Flow:**
1.  App asks Cache: "Do you have User 101?"
2.  **Hit:** Return data immediately.
3.  **Miss:** App reads from Database -> Writes to Cache -> Returns data.

* **Best For:** Read-heavy workloads.
* **Risk:** Data in cache can become stale if DB is updated directly.
* **Pros:** Resilient. If Cache fails, the system works (just slower).
* **Cons:** First request is always slow (Cold Start). Data can become stale.

### B. Write-Through (Synchronous) ✍️
The Application treats the Cache as the main data store. The Cache is responsible for updating the DB.

**The Flow:**
1.  App saves User 101 to Cache.
2.  Cache synchronously writes to DB.
3.  Returns success only when *both* are done.

* **Best For:** Data that cannot be lost (Financial).
* **Risk:** High write latency (Two writes per request).
* **Pros:** Strong consistency. Cache never has old data.
* **Cons:** Write latency is higher (must do two writes).

### C. Write-Behind (Asynchronous) 🚀
The App writes *only* to the Cache and returns "Success" immediately. A background process syncs to the DB later.

* **Best For:** High-volume writes (IOT, Counters, Likes).
* **Risk:** **Data Loss.** If Cache crashes before syncing, data is gone.
* **Pros:** Extremely fast writes.
* **Cons:** **Data Loss Risk.** If the Cache crashes before syncing, the data is gone forever.

---

## 3. The "Big Three" Failure Scenarios 📉

Senior engineers are expected to know how to break (and fix) a cache.

### A. Cache Penetration 🕳️
**The Attack:** A user requests `ID: -1` (or a UUID that doesn't exist).
1.  Cache Miss.
2.  DB Miss.
3.  The request keeps hitting the DB because the data *never* exists to be cached.
    **The Fix:**
* **Cache Nulls:** Store `Key: -1, Value: NULL` with a short TTL (5 mins).
* **Bloom Filters:** A probabilistic data structure that tells you if an ID *might* exist or *definitely does not*.

### B. Cache Avalanche ❄️
**The Disaster:** You set the TTL for all keys to 1 hour. At 12:00 PM, **ALL** keys expire simultaneously.
* Millions of requests hit the DB at once.
* DB crashes.
  **The Fix:**
* **Jitter:** Add a random number to the TTL (e.g., `TTL = 60 mins + random(0-5 mins)`).

### C. Thundering Herd (Hot Key Stampede) 🐘
**The Event:** A viral tweet expires in the cache. 10,000 users request it at the exact same millisecond.
* 10,000 threads see "Cache Miss".
* 10,000 threads query the DB for the *same* row.
  **The Fix:**
* **Mutex Locking:** Acquire a lock on the key. Only the first thread queries DB; others wait.

---

## 4. Implementation Example (Java/Spring) 💻

The **Cache-Aside** pattern with Mutex Locking (simplified).

```java
@Service
public class ProductService {

    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private ProductRepository dbRepo;

    public Product getProduct(String id) {
        String key = "prod:" + id;

        // 1. Try Cache
        String json = redis.opsForValue().get(key);
        if (json != null) return convert(json);

        // 2. Cache Miss: Acquire Lock (Prevent Thundering Herd)
        // setIfAbsent = SETNX (Atomic)
        Boolean lockAcquired = redis.opsForValue().setIfAbsent("lock:" + id, "1", 5, TimeUnit.SECONDS);

        if (lockAcquired) {
            try {
                // Double check cache in case someone else just finished
                json = redis.opsForValue().get(key);
                if (json != null) return convert(json);

                // 3. Query DB
                Product prod = dbRepo.findById(id);
                
                // 4. Update Cache (Add Random Jitter to 10 mins)
                long jitter = ThreadLocalRandom.current().nextLong(0, 60);
                redis.opsForValue().set(key, toJson(prod), 600 + jitter, TimeUnit.SECONDS);
                
                return prod;
            } finally {
                redis.delete("lock:" + id);
            }
        } else {
            // Lock failed: Wait 100ms and retry logic
            Thread.sleep(100);
            return getProduct(id); 
        }
    }
}
```

---

# 🎓 Senior Developer Interview Q&A (15-20 Questions)

These questions test depth, trade-offs, and operational knowledge.

### 🔥 Section 1: Architectural Choices

**Q1: Redis is single-threaded. How does it handle concurrent requests so fast?**
* **A:** It uses **I/O Multiplexing** (epoll/kqueue) and an Event Loop. It handles network I/O asynchronously but executes commands sequentially in memory. Since RAM operations are nanoseconds, the CPU is rarely the bottleneck; the network is.

**Q2: Redis vs. Memcached. When would you choose Memcached?**
* **A:** Almost never nowadays. However, Memcached is strictly multi-threaded, so it *can* scale vertically better on a single massive multicore server for pure simple Key-Value get/set operations involving small objects. Redis is preferred for data structures (Lists, Sets, Sorted Sets) and persistence.

**Q3: How do you handle Data Consistency between Cache and DB?**
* **A:** You cannot guarantee strong consistency without killing performance (2PC). We aim for **Eventual Consistency**.
    * **Strategy:** "Cache-Aside with Double Delete".
    * 1. Delete Cache.
    * 2. Update DB.
    * 3. (Async/Delayed) Delete Cache again to catch any race conditions where a read occurred between step 1 and 2.

**Q4: Explain Consistent Hashing and why it's needed for Caching.**
* **A:** In a cluster, if you use `Mod(N)`, adding a server changes the logic for *all* keys, flushing the entire cache (Cache Miss storm). **Consistent Hashing** maps keys and servers to a ring. Adding a server only affects keys falling in that specific segment (e.g., 1/N keys), keeping the rest valid.

---

### 💀 Section 2: Failure Scenarios

**Q5: What is the difference between Cache Penetration and Cache Breakdown?**
* **A:**
    * **Penetration:** Requesting data that *doesn't exist* in DB (Bypasses cache, hits DB). Fix: Bloom Filter / Cache Nulls.
    * **Breakdown:** A *hot key* exists, but expires, causing massive load (Thundering Herd). Fix: Mutex / Logical Expiration.

**Q6: How do you implement a Rate Limiter using Redis?**
* **A:**
    * **Simple:** `INCR` key with TTL. (Flawed at edges of the window).
    * **Advanced:** **Sliding Window Log** using `Sorted Sets (ZSET)`. The Score is the timestamp. Count elements in the range `[Now - Window, Now]`.

**Q7: Your Redis memory is full. What happens?**
* **A:** It depends on the `maxmemory-policy`.
    * `noeviction`: Returns errors on writes (Dangerous).
    * `allkeys-lru`: Evicts least used keys (Most common).
    * `volatile-lru`: Evicts least used keys *that have a TTL set*.

**Q8: How does a Bloom Filter work in the context of Caching?**
* **A:** It is a space-efficient probabilistic structure. Before hitting the DB, we check the filter.
    * If Filter says "No": The data **definitely** doesn't exist. Return 404.
    * If Filter says "Yes": The data *might* exist. Check Cache/DB. (Small false positive rate, zero false negative rate).

---

### 💾 Section 3: Redis Internals & Persistence

**Q9: Explain RDB vs. AOF persistence in Redis.**
* **A:**
    * **RDB (Snapshot):** Forks the process and dumps memory to disk every N minutes. Compact, fast startup, but potential data loss (last N minutes).
    * **AOF (Append Only File):** Logs every write command. High durability, slower startup, larger file.
    * *Pro Tip:* Use hybrid (RDB for base + AOF for recent changes).

**Q10: Why shouldn't you use `KEYS *` in production?**
* **A:** Redis is single-threaded. `KEYS *` is an O(N) operation. If you have 1 million keys, it blocks the entire server until it finishes scanning. Use `SCAN` (cursor-based iteration) instead.

**Q11: What is the "Redlock" algorithm?**
* **A:** A distributed locking algorithm for Redis clusters. It attempts to acquire locks on N independent Redis instances. If it acquires N/2 + 1 locks, it holds the global lock. (Controversial, but standard answer).

**Q12: How does Redis Replication work?**
* **A:** Asynchronous. Master sends a stream of commands to Slaves. If a Slave disconnects, it tries a partial resync (PSYNC) using an offset. If that fails, it does a full resync (RDB dump transfer).

---

### 🧠 Section 4: System Design Scenarios

**Q13: Design a "Leaderboard" service (Top 10 players).**
* **A:** Use Redis **Sorted Sets (`ZSET`)**.
    * `ZADD leaderboard <score> <player_id>` (O(log N)).
    * `ZREVRANGE leaderboard 0 9` (Get top 10).
    * This is infinitely faster than `SELECT * FROM scores ORDER BY score DESC LIMIT 10`.

**Q14: You have a global application (US, EU, Asia). How do you cache?**
* **A:** **Geo-Replicated Cache** (e.g., Redis Enterprise Active-Active). Or, use local Redis clusters in each region.
    * *Invalidation:* Hard part. If US updates data, propagate invalidation messages via a Message Queue (Kafka) to EU/Asia clusters.

**Q15: What is "Cache Warming"?**
* **A:** When a system deploys or restarts, the cache is empty. Performance will be terrible.
    * **Strategy:** A script runs before traffic is allowed, querying the most popular keys from DB and populating the cache.

**Q16: How do you handle "Big Keys" (e.g., a List with 1 million items) in Redis?**
* **A:** Big keys block the single thread when deleted or accessed.
    * **Access:** Don't read the whole list. Use `LRANGE` (pagination).
    * **Delete:** Don't use `DEL`. Use `UNLINK` (Non-blocking delete / async lazy free).

**Q17: What is a "Race Condition" in Read-Modify-Write and how to fix it?**
* **A:** Two threads read "Count = 10", increment to 11, and write back. Result is 11, should be 12.
    * **Fix:** Use Atomic commands (`INCR`) or Lua Scripts (Redis guarantees Lua scripts execute atomically).