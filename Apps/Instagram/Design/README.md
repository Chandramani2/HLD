# System Design: Photo Sharing (Instagram)

## 1. Requirements

### Functional
* **Upload:** Users upload photos/videos.
* **Feed:** Users view a scrolling feed of photos from people they follow.
* **Interactions:** Like, Comment, and Follow users.
* **Stories:** Ephemeral content (24h expiry).
* **Search/Explore:** Discover new content based on hashtags or interests.

### Non-Functional
* **High Availability:** The service must never go down (CAP Theorem: AP over CP).
* **Low Latency:** Feed generation must be < 200ms.
* **Read-Heavy System:** Read-to-Write ratio is roughly **100:1** (People view far more than they post).
* **Reliability:** No data loss (Photos are precious memories).

## 2. Estimations (Scale)

* **Users:** 2 Billion MAU.
* **Usage:** 100 Million photos uploaded per day.
* **Storage:**
    * Average photo size = 2MB.
    * Daily Storage = 100M * 2MB = **200 TB/day**.
    * **10 Years:** ~700 Petabytes (Requires massive Blob Storage).
* **Throughput:**
    * Writes: 100M / 86400 ≈ 1,150 uploads/sec.
    * Reads: 100M * 100 ≈ 115,000 requests/sec (Peak is much higher).

---

## 3. High-Level Architecture



[Image of instagram system architecture]


Since the system is Read-Heavy, we focus heavily on **Caching** and **Pre-computation**.

| Component | Tech Choice | Reason |
| :--- | :--- | :--- |
| **Object Storage** | **AWS S3 / Azure Blob** | Stores the actual `.jpg` / `.mp4` files. |
| **Metadata DB** | **PostgreSQL (Sharded)** | Stores User info, Photo metadata, Likes. Instagram famously uses Postgres. |
| **Graph DB** | **Custom / TAO** | Manages the "Follow" relationships (Social Graph). |
| **Cache** | **Redis / Memcached** | Stores pre-generated News Feeds and User Sessions. |
| **CDN** | **Cloudfront / Akamai** | Caches images geographically close to users. |

---

## 4. Schema Design (The Sharding Strategy)

A single database cannot hold this data. We must **Shard**.
Instagram uses a "Directory-Free" sharding approach based on **User ID**. All data for User X lives on Shard Y.

### A. Users Table
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY, -- Custom 64-bit ID
    username VARCHAR(32) UNIQUE,
    email VARCHAR(255),
    created_at TIMESTAMP
);
```

### B. Photos Table
**Key Design Choice:** The `photo_id` contains the Shard ID and Timestamp (Snowflake ID). This allows sorting by time *without* a separate index.

```sql
CREATE TABLE photos (
    photo_id BIGINT PRIMARY KEY, -- Contains Timestamp + Shard ID
    user_id BIGINT,
    caption TEXT,
    image_url VARCHAR(255), -- Points to S3 (e.g., s3://bucket/img_123.jpg)
    latitude DOUBLE,
    longitude DOUBLE,
    created_at TIMESTAMP
);
```

### C. Follows (The Social Graph)
This table grows exponentially ($N^2$). Often stored in a Graph DB or a specialized K-V store.
```sql
CREATE TABLE user_follows (
    follower_id BIGINT,
    followee_id BIGINT,
    PRIMARY KEY (follower_id, followee_id)
);
```

---

## 5. Senior Interview Topics & Logic

### Q1: "How do we generate the News Feed efficiently?"
**Naive SQL (Don't use this):**
```sql
SELECT * FROM photos 
WHERE user_id IN (SELECT followee_id FROM user_follows WHERE follower_id = Me)
ORDER BY created_at DESC 
LIMIT 20;
```
*Why it fails:* If I follow 1,000 people, the DB has to scan 1,000 tables, merge millions of rows, and sort them. Too slow.

**Senior Solution: Feed Generation Service (Fan-out on Write).**
1.  **User A uploads** a photo.
2.  **Async Job:** Server finds all followers of User A.
3.  **Fan-out:** Push the `photo_id` to the top of every follower's **Pre-computed Feed (Redis List)**.
4.  **Read:** When User B opens the app, we just read their Redis List ($O(1)$).

### Q2: "What about celebrities (Justin Bieber Problem)?"
**Problem:** Justin Bieber has 200M followers. Pushing one photo to 200M Redis lists is too much work ("Fan-out on Write" fails here).
**Senior Solution: Hybrid Approach.**
* **Normal Users:** Push model (Fan-out on Write).
* **Celebrities:** Pull model (Fan-out on Read).
* **Execution:** When I open my feed, the system loads my pre-computed list (friends) AND merges in tweets/photos from the celebrities I follow at query time.

### Q3: "How does the custom ID generation work?"
We cannot use standard auto-increment IDs because we are sharded. We need IDs that are unique *across* shards and roughly sortable by time.
**Instagram ID Format (64 bits):**
* **41 bits:** Timestamp (Milliseconds since epoch).
* **13 bits:** Shard ID (Logical partition).
* **10 bits:** Sequence number (Auto-increment per shard).
* *Result:* IDs are naturally ordered. `ORDER BY photo_id` is the same as `ORDER BY time` but much faster.

---

## 6. API Design

### A. Upload Photo
* **Endpoint:** `POST /v1/media/upload`
* **Process:**
    1.  Client uploads binary to API.
    2.  API writes to **S3** (Incoming Bucket).
    3.  **Lambda/Worker** triggers: Compresses image, generates thumbnails (small, medium, large), applies filters.
    4.  Moves processed image to **CDN Ready Bucket**.
    5.  Writes metadata to **Postgres**.

### B. Get Feed
* **Endpoint:** `GET /v1/feed?cursor=123456`
* **Response:** List of Photo Objects.
* **Pagination:** Uses **Cursor-based pagination** (using `photo_id`), NOT Offset-based (`LIMIT 10 OFFSET 50`).
    * *Why?* Offset is slow on large tables and causes duplicates if new items are added while scrolling.

---

## 7. Reliability & Redundancy

* **Photos:** S3 provides 99.999999999% durability. We also replicate buckets across regions (e.g., US-East to EU-West) for disaster recovery.
* **Database:** Master-Slave replication.
    * **Writes:** Go to Master.
    * **Reads:** Go to Replicas (Read consistency might lag by milliseconds, which is acceptable).

---

## 8. Summary Checklist

* [ ] **Storage:** S3 for images, Postgres for metadata.
* [ ] **Feed:** "Fan-out on Write" (Push) for normal users; "Pull" for celebs.
* [ ] **Sharding:** Based on User ID.
* [ ] **ID Generation:** Snowflake pattern (Time-sortable IDs).
* [ ] **Caching:** Extensive use of Redis for feeds to ensure < 200ms latency.