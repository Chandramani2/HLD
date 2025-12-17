# Redis Data Structures & System Design Guide

Redis is a high-performance, in-memory key-value store. However, calling it a "key-value store" under-sells it; it is a **data structures server**. Understanding which structure to use is critical for efficient system design.



---

## 1. Strings
**Concept:** The fundamental data type. A string can contain text, serialized objects (JSON, XML), or binary arrays (images), up to 512MB.

### Key Commands
| Command | Description | Complexity | Example |
| :--- | :--- | :--- | :--- |
| **`SET`** | Set key to value. | $O(1)$ | `SET user:1 "Alice"` |
| **`GET`** | Get key value. | $O(1)$ | `GET user:1` |
| **`INCR`** | Increment integer at key. | $O(1)$ | `INCR views` |
| **`SETEX`** | Set with expiration (TTL). | $O(1)$ | `SETEX sess:1 10 "data"` |
| **`MGET`** | Get multiple keys. | $O(N)$ | `MGET k1 k2` |

### 🏛️ System Design Use Cases
1.  **Session Caching:**
    * *Design:* Store user session tokens as keys and serialized user objects as values.
    * *Why:* Fast lookup $O(1)$ and automatic expiration (`SETEX`) handles session timeouts.
2.  **Distributed Locking:**
    * *Design:* Use `SET resource_lock unique_id NX PX 10000`.
    * *Why:* Ensures only one process can hold a lock (`NX` = Only set if Not Exists) with a failsafe TTL (`PX`).
3.  **Rate Limiter (Fixed Window):**
    * *Design:* Key is `user_id:minute_timestamp`. Value is a counter.
    * *Why:* `INCR` is atomic. If the count exceeds the limit, reject the request.

---

## 2. Lists
**Concept:** Linked lists of strings. Insertion at head/tail is fast, but random access by index is slow.

### Key Commands
| Command | Description | Complexity | Example |
| :--- | :--- | :--- | :--- |
| **`LPUSH`** | Push to head (Left). | $O(1)$ | `LPUSH q "job"` |
| **`RPUSH`** | Push to tail (Right). | $O(1)$ | `RPUSH q "job"` |
| **`LPOP`** | Pop from head. | $O(1)$ | `LPOP q` |
| **`RPOP`** | Pop from tail. | $O(1)$ | `RPOP q` |
| **`LRANGE`** | Get range of items. | $O(S+N)$ | `LRANGE q 0 -1` |

### 🏛️ System Design Use Cases
1.  **Asynchronous Message Queues:**
    * *Design:* Producer uses `LPUSH` to add jobs; Consumer uses `BRPOP` (blocking pop) to process them.
    * *Why:* Decouples services (e.g., an image upload service pushes a "resize" job to a worker queue).
2.  **Activity Feeds (Timeline):**
    * *Design:* Store the last 100 post IDs for a user in a list.
    * *Why:* `LPUSH` adds the newest post. `LTRIM user:feed 0 99` keeps the list size constant, discarding old data automatically.

---

## 3. Sets
**Concept:** Unordered collections of unique strings. Supports set math (union, intersection, difference).

### Key Commands
| Command | Description | Complexity | Example |
| :--- | :--- | :--- | :--- |
| **`SADD`** | Add member. | $O(1)$ | `SADD tags "ai"` |
| **`SREM`** | Remove member. | $O(1)$ | `SREM tags "ai"` |
| **`SISMEMBER`** | Check existence. | $O(1)$ | `SISMEMBER tags "ai"` |
| **`SMEMBERS`** | Return all members. | $O(N)$ | `SMEMBERS tags` |
| **`SINTER`** | Intersect sets. | $O(N*M)$ | `SINTER s1 s2` |

### 🏛️ System Design Use Cases
1.  **Recommendation Engines (Collaborative Filtering):**
    * *Design:* Store `user:1:likes` and `user:2:likes`.
    * *Why:* Use `SINTER` to find common interests or `SDIFF` to suggest items user 1 likes that user 2 hasn't seen yet.
2.  **Unique Tracking:**
    * *Design:* Track unique IP addresses visiting a page per day (`page:date:ips`).
    * *Why:* `SADD` automatically handles deduplication. $O(1)$ check for existence.
3.  **Access Control / Whitelisting:**
    * *Design:* Store allowed User IDs in a set `whitelist`.
    * *Why:* Fast $O(1)$ verification with `SISMEMBER`.

---

## 4. Hashes
**Concept:** Maps fields to values within a key. Ideal for storing objects without serializing the whole object into a single string.

### Key Commands
| Command | Description | Complexity | Example |
| :--- | :--- | :--- | :--- |
| **`HSET`** | Set field value. | $O(1)$ | `HSET u:1 name "A"` |
| **`HGET`** | Get field value. | $O(1)$ | `HGET u:1 name` |
| **`HGETALL`** | Get all fields. | $O(N)$ | `HGETALL u:1` |
| **`HINCRBY`** | Increment field. | $O(1)$ | `HINCRBY u:1 age 1` |

### 🏛️ System Design Use Cases
1.  **User Profiles / Object Storage:**
    * *Design:* Key `user:1001`, Fields: `name`, `email`, `login_count`.
    * *Why:* More memory efficient than storing JSON strings if you need to access/update individual fields frequently (e.g., incrementing just `login_count` without reading the whole object).
2.  **Shopping Cart:**
    * *Design:* Key `cart:session_id`, Field: `product_id`, Value: `quantity`.
    * *Why:* Easily check if an item is in the cart, update quantity, or remove specific items.

---

## 5. Sorted Sets (ZSets)
**Concept:** A Set where every member has an associated floating-point **Score**. Members are unique; scores can repeat. Ordered by score.

### Key Commands
| Command | Description | Complexity | Example |
| :--- | :--- | :--- | :--- |
| **`ZADD`** | Add with score. | $O(\log N)$ | `ZADD lb 100 "p1"` |
| **`ZRANGE`** | Get by rank order. | $O(\log N + M)$ | `ZRANGE lb 0 10` |
| **`ZREVRANGE`** | Get top scores. | $O(\log N + M)$ | `ZREVRANGE lb 0 10` |
| **`ZRANK`** | Get rank of user. | $O(\log N)$ | `ZRANK lb "p1"` |

### 🏛️ System Design Use Cases
1.  **Gaming Leaderboards:**
    * *Design:* Key `leaderboard`. Value: `PlayerID`, Score: `Points`.
    * *Why:* `ZREVRANGE` gives you the "Top 10 Players" instantly. `ZRANK` gives a player their specific global ranking efficiently.
2.  **Rate Limiter (Sliding Window):**
    * *Design:* Key `user:limit`. Score: `timestamp (ms)`, Member: `request_id`.
    * *Why:* To check limits, use `ZREMRANGEBYSCORE` to remove requests older than the window (e.g., 1 min ago), then count remaining items. This is more accurate than the fixed window counter.
3.  **Priority Queues:**
    * *Design:* Score is "priority level" (1=High, 3=Low).
    * *Why:* Workers always pop items with the lowest score (highest priority) first.

---

## 6. Bitmaps
**Concept:** Bit-level operations on strings. Extremely space-efficient.

### Key Commands
| Command | Description | Example |
| :--- | :--- | :--- |
| **`SETBIT`** | Set bit at offset. | `SETBIT login:2025-12 105 1` |
| **`BITCOUNT`** | Count set bits. | `BITCOUNT login:2025-12` |
| **`BITOP`** | Bitwise AND/OR/XOR. | `BITOP AND res k1 k2` |

### 🏛️ System Design Use Cases
1.  **Daily Active Users (DAU) Analytics:**
    * *Design:* Create a key per day. The "offset" is the User ID. If User 105 logs in, flip bit 105 to 1.
    * *Why:* Storing 1 million users takes only ~128KB of memory. You can run `BITOP AND` across 30 keys to find users who logged in *every day* this month.
2.  **Feature Flags:**
    * *Design:* Enable features for specific user IDs by setting their corresponding bit.

---

## 7. HyperLogLog
**Concept:** Probabilistic structure to count unique things (cardinality) with ~0.81% error rate, using constant memory (12KB).

### Key Commands
| Command | Description | Example |
| :--- | :--- | :--- |
| **`PFADD`** | Add element. | `PFADD hll "ip1"` |
| **`PFCOUNT`** | Get approximate count. | `PFCOUNT hll` |

### 🏛️ System Design Use Cases
1.  **High-Scale Unique Counters:**
    * *Design:* Counting unique search queries or unique visitors for Google/Facebook scale traffic.
    * *Why:* Using a standard Set for 100 million IPs would require gigabytes of RAM. HLL does it in 12KB.

---

## 8. Geospatial (Geo)
**Concept:** Stores coordinates and calculates distances/radii. Uses Sorted Sets internally.

### Key Commands
| Command | Description | Example |
| :--- | :--- | :--- |
| **`GEOADD`** | Add lat/long/name. | `GEOADD map 13.3 38.1 "A"` |
| **`GEORADIUS`** | Find nearby items. | `GEORADIUS map 15 37 20 km` |

### 🏛️ System Design Use Cases
1.  **Ride Sharing / Nearby Friends:**
    * *Design:* Store driver locations with `GEOADD`.
    * *Why:* When a user requests a ride, `GEORADIUS` finds all drivers within 5km instantly.

---

## 9. Streams
**Concept:** Append-only log. Similar to Kafka but lightweight. Supports consumer groups.

### Key Commands
| Command | Description | Example |
| :--- | :--- | :--- |
| **`XADD`** | Append event. | `XADD stream * temp 20` |
| **`XREADGROUP`** | Read as consumer group. | `XREADGROUP ...` |

### 🏛️ System Design Use Cases
1.  **Event Sourcing / Activity Logs:**
    * *Design:* Log every state change (e.g., "OrderPlaced", "PaymentProcessed") into a stream.
    * *Why:* Allows replaying history to rebuild system state or auditing.
2.  **Real-time Chat History:**
    * *Design:* Each chat room is a stream.
    * *Why:* Preserves message order perfectly and allows clients to fetch "messages since ID X" efficiently.