# System Design: Scalable Streaming Service (Netflix)

## 1. Requirements

### Functional
* **Admins:** Upload movies.
* **Users:** Watch movies, Search, View Recommendations, Save Watch History.

### Non-Functional
* **High Availability:** AP over CP (Availability > Consistency).
* **Low Latency:** Start time < 200ms.
* **Reliability:** No buffering.

## 2. Estimations (The Scary Numbers)

* **Users:** 200 Million DAU.
* **Peak Traffic:** 100M concurrent users.
* **Bandwidth:** 100M users × 5 Mbps (HD) = **500 Tbps**.
* **Storage:**
    * 10,000 Titles.
    * Raw 4K Movie = 100GB.
    * Transcoding (versions): 50 versions per movie (Res × Codecs × Devices).
    * Total per movie ≈ 1TB.
    * Total Storage = 10PB (Manageable).
* **Conclusion:** You **cannot** serve this from a central server. You **must** use a CDN.

---

## 3. Database Architecture

At this scale, **Polyglot Persistence** is required.

| Component | Database Choice | Reason |
| :--- | :--- | :--- |
| **User & Billing** | **PostgreSQL (RDBMS)** | ACID compliance is mandatory for payments and subscriptions. |
| **Movie Metadata** | **Cassandra / DynamoDB** | High read availability; Denormalized for fast reads. |
| **Watch History** | **Cassandra (Wide Column)** | Extremely write-heavy (Heartbeats); Linear scalability. |
| **Search** | **Elasticsearch** | Fuzzy search and autocomplete capabilities. |
| **Video Assets** | **S3 + CDN** | Object storage for files; CDN for delivery. |

---

## 4. Schema Design

### A. User & Billing (PostgreSQL)
Transactional integrity required. Shard based on `user_id` if necessary.

```sql
-- Users Table
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    subscription_plan_id INT,
    subscription_status VARCHAR(50), -- 'ACTIVE', 'CANCELLED'
    created_at TIMESTAMP DEFAULT NOW()
);

-- Profiles Table (1 User -> Many Profiles)
CREATE TABLE profiles (
    profile_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(user_id),
    profile_name VARCHAR(100),
    is_kid_friendly BOOLEAN DEFAULT FALSE,
    avatar_url VARCHAR(255)
);
```

### B. Metadata Service (Cassandra)
Read-heavy. Denormalized to avoid joins.

**Table 1: Movies by ID**
```cql
CREATE TABLE movies (
    movie_id UUID,
    title TEXT,
    description TEXT,
    release_year INT,
    rating FLOAT,
    genre_list SET<TEXT>,
    video_url_map MAP<TEXT, TEXT>, -- {'1080p': 's3://...', '4k': 's3://...'}
    PRIMARY KEY (movie_id)
);
```

**Table 2: Movies by Genre (Denormalized View)**
Partition by `genre`, Cluster by `release_year` for sorting.
```cql
CREATE TABLE movies_by_genre (
    genre TEXT,
    release_year INT,
    movie_id UUID,
    title TEXT,
    thumbnail_url TEXT,
    PRIMARY KEY ((genre), release_year, movie_id)
) WITH CLUSTERING ORDER BY (release_year DESC);
```

### C. Watch History (Cassandra)
Write-heavy. Partition by `profile_id` so all history for one user is on one node.

```cql
CREATE TABLE viewing_progress (
    profile_id BIGINT,
    movie_id UUID,
    stopped_at_seconds INT,
    last_watched TIMESTAMP,
    PRIMARY KEY ((profile_id), movie_id)
);
```

---

## 5. Senior Interview Queries

### Q1: "How do we efficiently fetch the 'Continue Watching' row?"
**Query:**
```cql
SELECT * FROM viewing_progress 
WHERE profile_id = 12345 
LIMIT 10;
```
**Explanation:** This hits a single partition (node), making it extremely fast. Sorting by time requires changing the Clustering Key to `last_watched`.

### Q2: "Calculate Top 10 Trending Movies for the last hour."
**Challenge:** `COUNT(*)` on 200M rows is too slow.
**Solution:** Use Stream Processing (Spark/Flink) or Approximate Aggregation (Count-Min Sketch).
**Query (on pre-aggregated table):**
```sql
SELECT movie_id, COUNT(*) as view_count
FROM movie_views_stream
WHERE window_start >= NOW() - INTERVAL '1 hour'
GROUP BY movie_id
ORDER BY view_count DESC
LIMIT 10;
```

### Q3: "Debug Write Consistency for Watch History."
**Scenario:** A user watched a movie, but history didn't save.
**Explanation:** For history, we use **Consistency Level: ONE** (AP over CP). If the node accepts the write but dies before replicating, data is lost. We accept this trade-off to ensure the video keeps playing without blocking for database confirmations.

---

## 6. API Design (The Gateway)

**Protocol:** * **GraphQL** for Mobile/TV (Fetch only what is needed: resolution, thumbnails).
* **REST** for Write ops / Billing.

### Critical Endpoints

**1. Start Stream**
* **POST** `/v1/stream/init`
* **Response:** Returns a **Manifest File** (URL), DRM token, and timestamp to resume.
```json
{
  "stream_id": "s98765",
  "cdn_url": "[https://cdn-edge.netflix.com/manifests/m555/master.m3u8](https://cdn-edge.netflix.com/manifests/m555/master.m3u8)",
  "bookmark_time": 300
}
```

**2. Heartbeat**
* **POST** `/v1/history/heartbeat`
* **Payload:** `{"stream_id": "...", "timestamp": 310}`
* **Note:** "Fire and Forget" strategy.

---

## 7. CDN & Streaming Architecture

### The Backbone: Open Connect
At 500 Tbps, you cannot serve from S3.
1.  **Origin:** Master copy in S3 (Transcoded into H.264/H.265, 480p/4K).
2.  **Edge (OCAs):** Custom hardware servers placed **inside ISPs**. (e.g., The video is streamed from the user's local ISP data center, not across the ocean).
3.  **Proactive Caching:** Predictive algorithms push popular content to Edge servers during off-peak hours (e.g., 3 AM).

### Streaming Protocol: DASH / HLS (Adaptive Bitrate)
* **Manifest File:** Lists available chunks (4-second segments) in different qualities.
* **Client Logic:**
    1.  Check network speed.
    2.  Download chunk at appropriate quality.
    3.  If speed drops, download next chunk at lower quality (prevents buffering).

---

## 8. Summary Checklist

* [x] **Database:** Postgres for money, Cassandra for massive reads/writes.
* [x] **Search:** Elasticsearch.
* [x] **Video:** S3 -> Custom CDN (Open Connect).
* [x] **Consistency:** Eventual Consistency (AP) for video/history; Strong Consistency (CP) for billing.