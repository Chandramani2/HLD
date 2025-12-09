# 🔴 The Senior Redis Command Playbook

> **Target Audience:** Backend Engineers, Game Developers, & Architects  
> **Goal:** Master **Atomic Operations**, **Complex Data Structures**, and **Performance Tuning**.

In a senior interview, you must know why **Lua Scripts** are better than Transactions, how to implement **Rate Limiting** without race conditions, and how to query **Geospatial** data.

---

## 📖 Table of Contents
1. [Part 1: The "Kill" Command (`KEYS` vs `SCAN`)](#-part-1-the-kill-command-keys-vs-scan)
2. [Part 2: Atomicity (Transactions vs. Lua)](#-part-2-atomicity-transactions-vs-lua)
3. [Part 3: Throughput (Pipelining vs. MGET)](#-part-3-throughput-pipelining-vs-mget)
4. [Part 4: Specialized Structures (Geo, Streams, HyperLogLog)](#-part-4-specialized-structures)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 💀 Part 1: The "Kill" Command (`KEYS` vs `SCAN`)

Redis is Single-Threaded. One slow command stops the entire company.

### 1. The Forbidden Command: `KEYS *`
* **Command:** `KEYS user:*`
* **Complexity:** $O(N)$.
* **Impact:** If you have 100 million keys, this command takes 20 seconds. During those 20 seconds, **no other client can read/write**. The app goes down.

### 2. The Senior Solution: `SCAN`
* **Command:** `SCAN 0 MATCH user:* COUNT 100`
* **Complexity:** $O(1)$ per call.
* **Mechanism:** It returns a **Cursor** and a small batch of keys. You keep calling it with the new cursor until it returns `0`.
* **Impact:** The main thread is never blocked for more than a few microseconds. Other clients can interleave their requests.

---

## ⚛️ Part 2: Atomicity (Transactions vs. Lua)

How do you perform "Read-Modify-Write" safely?

### 1. The Old Way: `MULTI` / `EXEC`
Redis transactions are *not* like SQL transactions. They queue commands.
* **Flow:**
    1.  `WATCH balance` (Optimistic Lock).
    2.  `MULTI`.
    3.  `DECR balance`.
    4.  `EXEC`.
* **Problem:** If `balance` changes between Step 1 and 4, the transaction fails. You must handle retries in your code.

### 2. The Senior Way: Lua Scripting (`EVAL`)
Redis guarantees that a Lua script executes **atomically**. No other script or command will run while it is executing.

**Scenario:** "Rate Limit: Allow request only if count < 10."

```lua
local current = redis.call('GET', KEYS[1])
if current and tonumber(current) >= 10 then
    return 0 -- Rejected
else
    redis.call('INCR', KEYS[1])
    return 1 -- Allowed
end
```

**Command:**
```redis
EVAL "local c = ..." 1 my_rate_limit_key
```
* **Benefit:** 1 Network Round Trip. Zero Race Conditions. No `WATCH` complexity.

---

## 🚀 Part 3: Throughput (Pipelining vs. MGET)

Latency kills. If Ping is 1ms, doing 1,000 sequential `GET`s takes 1 second.

### 1. Batch Commands (`MGET` / `MSET`)
* **Use:** Fetching known keys.
* **Command:** `MGET user:1 user:2 user:3`
* **Benefit:** 1 Round Trip. Atomic (Snapshot isolation).

### 2. Pipelining
* **Use:** Sending unrelated commands.
* **Mechanism:** The client sends 100 commands into the socket without waiting for replies. Redis processes them and sends 100 replies back in one packet.
* **Benefit:** Throughput increases by 10x-50x.
* **Senior Note:** This is **not atomic**. Command 5 might fail while Command 6 succeeds.

---

## 🗺️ Part 4: Specialized Structures

Don't use Strings for everything.

### 1. Geospatial (`GEO`)
**Scenario:** "Find drivers within 5km."
* **Under the hood:** It uses **Geohash** stored in a Sorted Set (ZSET).
* **Command:**
    * Add: `GEOADD drivers 13.36 38.11 "DriverA"`
    * Search: `GEORADIUS drivers 15 37 200 km WITHDIST`

### 2. Streams (`XADD`)
**Scenario:** "Lightweight Kafka."
* **Use:** Event sourcing, Activity Feeds.
* **Command:**
    * Add: `XADD mystream * sensor-id 1234 temp 19.8`
    * Read: `XREAD COUNT 2 STREAMS mystream 0`
* **Features:** Supports Consumer Groups (like Kafka) to distribute processing.

### 3. HyperLogLog (`PFADD`)
**Scenario:** "Count unique IP addresses (Approximate)."
* **Command:**
    * Add: `PFADD visits 1.2.3.4`
    * Count: `PFCOUNT visits`
* **Memory:** Fixed 12KB. Even for 1 Billion unique IPs.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Real-Time Leaderboard
**Interviewer:** *"Design a leaderboard where we can get the rank of a user and the top 10 players."*

* ✅ **Senior Answer:** "**Sorted Sets (ZSET).**"
    * **Add Score:** `ZADD leaderboard 1500 "UserA"`
    * **Get Rank:** `ZREVRANK leaderboard "UserA"` (0-based index from top).
    * **Get Top 10:** `ZREVRANGE leaderboard 0 9 WITHSCORES`.
    * *Efficiency:* $O(\log N)$ update and retrieval.

### Scenario B: Priority Queue (Job Processing)
**Interviewer:** *"We need a job queue. High priority jobs must be processed first."*

* ❌ **Bad Answer:** "Use `RPUSH` and sort in the client."
* ✅ **Senior Answer:** "**Use `BLPOP` on multiple lists.**"
    * Create lists: `queue:high`, `queue:medium`, `queue:low`.
    * **Worker Command:** `BLPOP queue:high queue:medium queue:low 0`
    * **Behavior:** Redis checks `queue:high` first. If empty, checks `queue:medium`.
    * **Result:** Strict priority enforcement with blocking waits (no CPU burn).

### Scenario C: Distributed Locking (Safety)
**Interviewer:** *"How do we ensure a lock expires safely if the server crashes?"*

* ✅ **Senior Answer:** "**SET with NX and PX.**"
    ```redis
    SET lock:resource random_token NX PX 30000
    ```
    * `NX`: Only set if not exists (Atomic lock acquisition).
    * `PX`: Auto-expire in 30s (Prevents deadlock if app crashes).
    * **Release:** Use a Lua script to check if `GET lock:resource == random_token` before deleting (Ownership check).

### Scenario D: Time-Series Data (Throttling Memory)
**Interviewer:** *"We store metrics for the last 24 hours. We don't care about older data."*

* ✅ **Senior Answer:** "**ZSET with `ZREMRANGEBYSCORE`.**"
    * Store metrics in ZSET where `Score = Timestamp`.
    * **Write:** `ZADD metrics <timestamp> <value>`
    * **Cleanup:** Every insert, run `ZREMRANGEBYSCORE metrics -inf <timestamp_24h_ago>`.
    * *Better Option:* Use **RedisTimeSeries** module if available, but ZSET is the native way.

---

### **Final Checklist**
1.  **Iterating:** Use `SCAN`, never `KEYS`.
2.  **Logic:** Use Lua (`EVAL`) for complex atomic operations.
3.  **Queues:** Use `BLPOP` for blocking reads or `XADD` for streams.
4.  **Counting:** Use `HyperLogLog` for massive sets with low RAM.

**This concludes the Redis Command Playbook.**