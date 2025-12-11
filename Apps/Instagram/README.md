# 📸 The Senior System Design Playbook: Design Instagram

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Design a photo-sharing platform with **1 Billion Users**, **Low Latency Feeds**, and **Massive Storage**.

In a senior interview, "Design Instagram" is not about storing pictures. It is about **Feed Generation Strategies (Push vs. Pull)**, **Graph Data Models**, and **Optimizing storage for billions of small files**.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The Scale Problem](#-part-1-requirements--the-scale-problem)
2. [Part 2: High-Level Architecture (Write vs. Read Paths)](#-part-2-high-level-architecture)
3. [Part 3: The Data Model (Graph & Sharding)](#-part-3-the-data-model-graph--sharding)
4. [Part 4: Feed Generation (The Fanout Problem)](#-part-4-feed-generation-the-fanout-problem)
5. [Part 5: Storage Optimization (Haystack & Cold Storage)](#-part-5-storage-optimization)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The Scale Problem

### 1. Requirements
* **Functional:** Upload Photos/Videos, Follow Users, Like/Comment, **News Feed**, Stories (24h expiry).
* **Non-Functional:**
    * **High Availability:** Eventual consistency is acceptable (if a Like shows up 5 seconds late, it's fine).
    * **Low Latency:** Feed load time < 200ms.
    * **Reliability:** Never lose a photo.

### 2. Back-of-the-Envelope Math (The Storage Beast)
* **Users:** 1 Billion Active.
* **Uploads:** 100 Million photos/day.
* **Size:** Avg photo = 400KB.
* **Daily Storage:** $100M \times 400KB = 40TB$ / day.
* **10 Years:** $40TB \times 365 \times 10 \approx 146PB$.
* **Insight:** We need a tiered storage solution. We cannot keep 146PB on expensive SSDs.

---

## 🏗️ Part 2: High-Level Architecture

We split the architecture into two flows: **The Upload (Write)** and **The Feed (Read)**.

### 1. The Upload Path (Async)
1.  **Client:** Uploads image to **CDN/S3** directly (Presigned URL) to avoid blocking App Servers.
2.  **App Server:** Receives metadata ("User A posted Photo ID 123").
3.  **DB:** Save metadata.
4.  **Message Queue (Kafka):** Publish "New Post" event.
5.  **Workers:**
    * *Image Processor:* Generate thumbnails, apply filters.
    * *AI Service:* Detect NSFW content.
    * *Fanout Service:* Update follower feeds.

### 2. The Read Path (The Cache)
* Users almost never read directly from the database.
* **Feed Service:** Queries a **Redis Cluster** where the user's feed is pre-computed.
* **Media:** Fetched from **CDN** (Edge Location).

<img width="1848" height="1426" alt="Image" src="https://github.com/user-attachments/assets/5f7663c7-4ff9-4648-aa5a-298aafdcba86" />

---

## 🕸️ Part 3: The Data Model (Graph & Sharding)

How do we store "User A follows User B"?

### 1. The Schema (SQL vs Graph)
While Graph DBs (Neo4j) seem obvious, Instagram actually runs on **Sharded PostgreSQL**.

* **Users Table:** `ID, Name, Bio`.
* **Photos Table:** `ID, UserID, S3_URL, Caption`.
* **Follows Table:** `FollowerID, FolloweeID`.

### 2. Sharding Strategy (User-Centric)
We have billions of photos. A single DB instance will die.
* **Shard Key:** `UserID`.
* **Why?** When a user opens their profile, we need *all* their photos. If we shard by `PhotoID`, a user's photos are scattered across 50 shards (50 DB queries). If we shard by `UserID`, all their photos are on **Shard #4** (1 DB query).
* **Global Lookup:** We need a separate generic service (or Zookeeper) to map `UserID -> Shard ID`.

### 3. Generating IDs (Snowflake)
* We cannot use Auto-Increment (it's not unique across shards).
* **Instagram's solution:** Custom 64-bit ID.
    * `41 bits`: Timestamp.
    * `13 bits`: **Logical Shard ID** (Crucial: The ID itself tells us which DB shard to look in).
    * `10 bits`: Sequence.

---

## 📡 Part 4: Feed Generation (The Fanout Problem)

This is the most critical part of the interview. **How do we generate the Home Feed?**

### Strategy A: Pull Model (Fanout-on-Load)
* **Logic:** When User opens app -> Query DB for all 500 followings -> Fetch latest posts -> Sort in memory.
* **Pros:** Simple Writes.
* **Cons:** **Terrible Read Latency**. If I follow 1,000 users, that's a massive DB query every time I open the app.

### Strategy B: Push Model (Fanout-on-Write) 🏆
* **Logic:**
    1.  User A posts a photo.
    2.  System finds all of User A's followers.
    3.  System pushes the Photo ID into every follower's **Redis List** (Feed Cache).
    4.  When Follower opens app, we just read their Redis List ($O(1)$).
* **Pros:** **Instant Reads**.
* **Cons:** **The "Celebrity" Problem**.

### The Hybrid Approach (The Senior Solution)
What happens if **Justin Bieber (100M followers)** posts?
* **Push:** Writing to 100M Redis caches takes too long.
* **Solution:**
    * **Normal Users:** Use **Push**.
    * **Celebrities (VIPs):** Use **Pull**.
    * **Feed Construction:** When I open my feed, the system merges my "Pushed" Redis feed + "Pulled" updates from any Celebrities I follow.



---

## 💾 Part 5: Storage Optimization (Haystack)

How do we store 100 billion 50KB files efficiently?

### The Problem: Metadata Overhead (Inodes)
Standard Linux filesystems (ext4) perform poorly with billions of tiny files. The metadata lookups (checking permissions/location) become the bottleneck, not the data transfer.

### The Solution: Object Storage (Haystack / Facebook F4)
* **Concept:** Don't store 1 file per photo. Store **10,000 photos in 1 giant physical file**.
* **Mechanism:**
    * Physical File: `[Header | Photo 1 | Photo 2 | ... | Photo N]`
    * Database: Stores `Offset` and `Length`.
    * Read: "Go to Physical File A, seek to Byte 4096, read 50KB."
* **Result:** 1 Disk Seek = 1 Photo. Massive performance boost.

---

## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: Stories (Ephemeral Data)
**Interviewer:** *"Design Instagram Stories. They disappear after 24 hours. They have higher throughput than the main feed."*

* ✅ **Senior Answer:** "**Redis with TTL + Write-Through.**"
    * Stories are temporary.
    * Use **Redis** as the primary store for active stories.
    * Set `TTL = 24 hours` on the key. Redis automatically deletes it.
    * **Persistence:** Asynchronously write to Cassandra (Cold Storage) just for legal/backup reasons, but serve 100% of reads from Cache.

### Scenario B: The "Like" Counter (High Concurrency)
**Interviewer:** *"Justin Bieber posts. 1 million people 'Like' it in 1 minute. How do we count this without locking the DB row?"*

* ❌ **Bad Answer:** `UPDATE photos SET likes = likes + 1 WHERE id = 123`. (Database Row Lock will choke).
* ✅ **Senior Answer:** "**Async Aggregation / Sharded Counters.**"
    * **Option 1 (Redis):** Increment a Redis counter (Atomic). Flush to DB every 10 seconds.
    * **Option 2 (Kafka):** Stream "Like Events" to Kafka. A Flink job aggregates them (`+500 likes`) and writes a batch update to the DB.

### Scenario C: Search (Hashtags)
**Interviewer:** *"How do we search for #Sunset?"*

* ✅ **Senior Answer:** "**Inverted Index (Elasticsearch).**"
    * We cannot search the main SQL DB (User Sharded).
    * We need a separate Search Service.
    * When a photo is posted with `#Sunset`, we send an event to Elastic.
    * **Index:** `Key: "sunset" -> Value: [List of PhotoIDs]`.

### Scenario D: Data Locality (CDN)
**Interviewer:** *"A user in India uploads a photo. A user in the USA views it. It's slow."*

* ✅ **Senior Answer:** "**Geo-Routing & Edge Caching.**"
    * Upload: User uploads to the nearest Edge Node (India).
    * Async Replication: The image is copied to the USA region (S3 Cross-Region Replication).
    * View: The USA user requests the image. CDN checks local USA Edge cache. If miss, fetches from USA S3 bucket (not India).

---

### **Final Summary for the Interview**
To win the Social Media interview, focus on:
1.  **Hybrid Feed Generation:** Push for us, Pull for Bieber.
2.  **Sharding:** Shard by `UserID`, not `PhotoID`.
3.  **Storage:** Small file optimization (Haystack).
4.  **CDN:** Essential for media delivery.

**Would you like to simulate "Designing the Comments System" specifically (Nested threads, Pagination, Real-time updates)?**