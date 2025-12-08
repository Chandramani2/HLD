# ⚡ The Senior Caching Playbook

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Master **Consistency Patterns**, **Eviction Policies**, and **Disaster Prevention** (Avalanches & Stampedes).

In a senior interview, saying "We'll put it in Redis" is not enough. You must explain **how** data gets there, **when** it leaves, and **what happens** when the cache fails.

---

## 📖 Table of Contents
1. [Part 1: The Caching Hierarchy (Layers)](#-part-1-the-caching-hierarchy-layers)
2. [Part 2: The 3 Caching Strategies (Patterns)](#-part-2-the-3-caching-strategies-patterns)
3. [Part 3: Eviction Policies (LRU vs LFU)](#-part-3-eviction-policies-lru-vs-lfu)
4. [Part 4: The 3 Cache Disasters (And how to fix them)](#-part-4-the-3-cache-disasters)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🍰 Part 1: The Caching Hierarchy (Layers)

Caching happens at every layer. Knowing where to cache is half the battle.

| Layer | Technology | Latency | Best For |
| :--- | :--- | :--- | :--- |
| **Browser / Client** | LocalStorage, HTTP Headers | 0ms | User-specific static data (Session tokens, Theme settings). |
| **CDN (Edge)** | Cloudflare, Akamai | ~20-50ms | Static Assets (Images, CSS, JS), Video. |
| **Load Balancer** | Nginx, Varnish | ~50ms | Public API responses, HTML pages. |
| **Distributed Cache** | Redis, Memcached | ~1-5ms | Shared Application State (Sessions, Leaderboards). |
| **Local Cache (In-App)** | Guava, Caffeine, HashMap | < 1ms | Immutable config data, Hot keys (prevents network calls to Redis). |

> **💡 Senior Tip:** **Local Cache** is the fastest but dangerous. If Server A updates its local cache, Server B doesn't know. Only use it for data that rarely changes (e.g., Country Codes).

[Image of Caching Hierarchy Architecture Diagram]

---

## 🔄 Part 2: The 3 Caching Strategies (Patterns)

How do data and cache interact?

### 1. Cache-Aside (Lazy Loading)
* **Flow:** App checks Cache. If `miss`, App reads DB -> Updates Cache -> Returns to User.
* **Pros:** Resilient to cache failure (DB is the fallback). Data model in cache can be different from DB.
* **Cons:** First request is slow (Cache Miss). **Data Staleness** (Data in DB changes, Cache remains old until TTL expires).
* **Use Case:** General purpose (User Profiles).

### 2. Write-Through
* **Flow:** App writes to Cache AND DB at the same time.
* **Pros:** **Strong Consistency**. Cache is always fresh.
* **Cons:** Higher write latency (2 writes).
* **Use Case:** Critical data where you cannot afford to show stale info (e.g., Game High Scores).

### 3. Write-Back (Write-Behind)
* **Flow:** App writes *only* to Cache. Cache asynchronously syncs to DB later.
* **Pros:** **Extreme Write Speed**.
* **Cons:** **Data Loss Risk**. If Cache crashes before syncing, the data is gone forever.
* **Use Case:** Analytics counters, "Likes" on a post (It's okay to lose a few likes).

[Image of Cache Aside vs Write Through vs Write Back Diagram]

---

## 🗑️ Part 3: Eviction Policies (LRU vs LFU)

Memory is finite. What do we delete when it's full?

### 1. LRU (Least Recently Used)
* **Concept:** Remove the item that hasn't been accessed for the longest time.
* **Scenario:** Most social media apps. If you viewed a profile 1 second ago, you are likely to view it again.
* **Implementation:** Doubly Linked List + Hash Map ($O(1)$).

### 2. LFU (Least Frequently Used)
* **Concept:** Remove the item with the fewest total hits.
* **Scenario:** A library system. "Harry Potter" is accessed often (High Frequency), even if nobody borrowed it *today*. Don't evict it just because it wasn't used recently.

### 3. FIFO (First In First Out)
* **Concept:** Remove the oldest item created.
* **Scenario:** Time-series data where old data is useless.

---

## 💥 Part 4: The 3 Cache Disasters

This is the "Senior" section. These problems take down production systems.

### 1. Cache Penetration
* **The Problem:** A malicious user requests `id = -1` (doesn't exist). The Cache misses. The DB misses. The user spams this 100k times. The DB dies.
* **The Fix:**
    * **Bloom Filter:** A fast check "Does this ID possibly exist?". If no, block request.
    * **Cache Nulls:** Store `Key(-1) = "NULL"` in Redis with a short TTL (e.g., 5 min).

### 2. Cache Avalanche
* **The Problem:** We cache 1 million keys. They all have `TTL = 1 hour`. At 12:00 PM, they **all expire at once**. The DB gets 1 million hits instantly.
* **The Fix:**
    * **Jitter:** Randomize TTL. `TTL = 60 mins + Random(0-10 mins)`.

### 3. Hot Key (Thundering Herd)
* **The Problem:** A celebrity tweets. 10M users request the *same* key. The key expires. 10M requests hit the DB simultaneously to calculate the *same* value.
* **The Fix:**
    * **Distributed Locking (Mutex):** Only allow **one** process to rebuild the cache. The other 9.9M wait for the cache to populate.
    * **Pre-warming:** Refresh the cache *before* it expires (Probabilistic Early Expiration).

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Real-Time Leaderboard
**Interviewer:** *"We need a leaderboard. It updates every second. Should we use Cache-Aside?"*

* ❌ **Junior Answer:** "Yes, set TTL to 1 second." (Too many DB hits).
* ✅ **Senior Answer:** "**Redis Sorted Sets (ZSET) with Write-Through.**"
    * Don't touch the DB for every score update.
    * Increment score directly in Redis (`ZINCRBY`).
    * Persist Redis to disk (RDB/AOF) or use a background worker to sync "Final Scores" to SQL every minute (Write-Back).

### Scenario B: Distributed Caching Latency
**Interviewer:** *"We have a global app. Users in Australia complain that the cache (hosted in US) is slow."*

* ✅ **Senior Answer:** "**Geo-Replicated Redis (CRDTs).**"
    * Use Redis Enterprise (Active-Active) or DynamoDB Global Tables.
    * Replicate the cache to a local AWS region in Sydney.
    * **Trade-off:** Eventual Consistency between regions.

### Scenario C: Consistency (The "Stale Data" Problem)
**Interviewer:** *"User updates their profile. They refresh the page, but they still see the old profile. How to fix?"*

* **Root Cause:** Cache-Aside pattern. The DB is updated, but the Cache still holds the old value until TTL expires.
* ✅ **Senior Answer:** "**Double Deletion / Cache Invalidation.**"
    * **Strategy:** On `UpdateProfile()`:
        1.  Delete Key from Cache.
        2.  Update DB.
        3.  (Optional) Delete Key again (Delayed) to catch any race conditions.
    * **Why Delete instead of Update?** Updating the cache is complex (concurrency). Deleting forces a clean fetch on the next read.

### Scenario D: Redis is Full
**Interviewer:** *"Redis memory is 100% full. New writes are failing. What do we do?"*

* ❌ **Bad Answer:** "Just add more RAM." (Temporary fix).
* ✅ **Senior Answer:**
    1.  **Check Eviction Policy:** Are we using `noeviction`? Switch to `allkeys-lru` to automatically delete old data.
    2.  **Audit TTL:** Are we storing keys forever? Enforce a default TTL on everything.
    3.  **Partitioning:** Are we storing huge objects? Compress JSON before caching (Snappy/Gzip) or split big objects.
