# API Latency Investigation: 200ms to 2s

This document provides a systematic approach to identifying why a GET API has slowed down and how to apply the necessary optimizations across the infrastructure and application layers.

---

## 1. The Problem Statement
- **Initial Performance:** 200ms (Healthy)
- **Current Performance:** 1,000ms - 2,000ms (Critical)
- **Primary Goal:** Identify the bottleneck and return to the 200ms baseline.

---

## 2. Phase 1: Identifying the Bottleneck (The "Where")

Before optimizing, you must determine which segment of the request lifecycle is consuming the extra time.

### A. Network & Infrastructure Leg
If the delay occurs between the user and the Load Balancer (LB) or between the LB and the Server:
- **Client to LB:** Usually caused by geographic distance or slow SSL handshakes.
- **LB to Server:** Caused by high TCP connection overhead or "limping" backend instances.
- **Diagnosis:** Compare `Total Latency` vs. `Target Response Time` in your Load Balancer metrics.

### B. Application Leg (Java/Connection Pooling)
If the database is fast but the API is slow, the application might be struggling to "talk" to the database.
- **Connection Exhaustion:** Threads are waiting in a queue because all database connections are in use.
- **Connection Leaks:** Connections are opened but never closed, eventually starving the pool.
- **Diagnosis:** Check HikariCP metrics for `PendingThreads` or use a Thread Dump to look for `BLOCKED` states.

### C. Data Leg (Database Performance)
The most common cause of 2s latency is a query that no longer scales with data growth.
- **Nested Loop Joins:** Without proper indexes, the database performs a brute-force search ($O(N \times M)$).
- **Table Bloat:** As rows increase, a "Full Table Scan" that took 10ms now takes 1.5s.
- **Diagnosis:** Run `EXPLAIN ANALYZE` on the specific SQL query.

---

## 3. Phase 2: Solutions & Optimizations

### Solution A: Fixing Network Latency
1. **Content Delivery Network (CDN):** Use a CDN to terminate the SSL handshake closer to the user, reducing the "Round Trip Time" (RTT).
2. **TLS 1.3:** Upgrade your security protocols to reduce the number of handshakes required for a secure connection.
3. **Keep-Alive Tuning:** Ensure your server's Keep-Alive timeout is longer than your Load Balancer's timeout to avoid frequent connection re-establishment.

To fix latency, you first need to pinpoint which **segment** of the journey is slow. A request has two main network legs:
1.  **Client to Load Balancer (LB):** The "External" leg.
2.  **Load Balancer to Server (Target):** The "Internal" leg.

---

## 1. How to Identify the Source
You can use **Load Balancer Metrics** (like AWS CloudWatch, Azure Monitor, or Nginx logs) to distinguish between these two.



### Check these specific metrics:
* **Target Response Time** (or `target_processing_time`): This is the time the Load Balancer spent waiting for your Server to respond.
    * *If this is high (e.g., 1.8s):* The problem is between the LB and the Server (or inside the server itself).
* **Request Preprocessing Time:** The time the LB spent processing the request before sending it to the server.
    * *If this is high:* The LB itself is overloaded or misconfigured.
* **Total Latency minus Target Response Time:** If the total time the client waited is 2s, but the LB says the target only took 200ms, the remaining 1.8s is lost between the Client and the LB.

---

## 2. Fixing Latency: Client → Load Balancer
If the delay happens here, it's usually due to geographical distance, DNS issues, or SSL handshakes.

* **Geographical Distance:** If your server is in New York and the client is in Singapore, physics limits the speed.
    * **Fix:** Use a **CDN (Content Delivery Network)** like Cloudflare or AWS CloudFront to terminate the connection closer to the user.
* **SSL/TLS Handshake:** Negotiating a secure connection can take multiple round trips.
    * **Fix:** Enable **TLS 1.3** (faster than 1.2) and **OCSP Stapling** to reduce the number of round trips required for a secure handshake.
* **TCP Window Scaling:** If the client or LB has a small buffer, data "bottlenecks."
    * **Fix:** Ensure your LB has **Keep-alive** enabled so it doesn't have to rebuild the connection for every request.

---

## 3. Fixing Latency: Load Balancer → Server
If the delay happens here, the internal network or the application server is struggling.

* **Idle Timeout / Connection Pooling:** If the LB and Server keep closing connections, you pay a "handshake tax" every time.
    * **Fix:** Ensure the **Keep-Alive timeout** on your server (e.g., Nginx, Apache, or Tomcat) is **higher** than the LB's idle timeout. This prevents the server from closing a connection while the LB is trying to use it.
* **Target Health Issues:** If the LB is sending traffic to "limping" servers that are about to fail health checks.
    * **Fix:** Tighten your **Health Check** intervals. If a server is slow, the LB should mark it "unhealthy" faster and stop sending it traffic.
* **Resource Saturation:** The server may be out of CPU or RAM, causing it to take longer to "accept" the connection from the LB.
    * **Fix:** Check the **Target Connection Errors** metric. If it's rising, your server's "backlog" (queue of pending connections) is likely full. Increase the server's `accept_count` or `MaxClients`.

---

## 4. Summary Table

| Metric to Check | High Value Means... | Primary Fix |
| :--- | :--- | :--- |
| **TargetResponseTime** | Server/App is slow | Optimize code, DB, or add more servers (Scaling). |
| **TargetConnErrorCount** | Network/Port issue | Check Security Groups/Firewalls and Port limits. |
| **Client TLS Latency** | Handshake is slow | Upgrade to TLS 1.3; use a CDN. |
| **ELB 504 (Timeout)** | Server took too long | Increase LB timeout or optimize application logic. |

### Solution B: Optimizing Java Connection Pooling
1. **Try-With-Resources:** Ensure every database interaction uses a pattern that guarantees the connection is returned to the pool, even if an error occurs.
2. **Pool Sizing:** Use the formula `(DB_CPU_Cores * 2) + 1` to set your `maximum-pool-size`. Avoid setting it too high (e.g., 100+), as this causes database context switching.
3. **Leak Detection:** Enable `leak-detection-threshold` in your Hikari settings to log a stack trace whenever a thread holds a connection for more than 30 seconds.

```properties
# Match min/max to avoid "cold start" latency
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=10

# Fail fast if no connection is available (2 seconds)
spring.datasource.hikari.connection-timeout=2000

# Critical for finding code that forgets to close connections
spring.datasource.hikari.leak-detection-threshold=30000
```

### Solution C: Solving Database Bottlenecks
1. **Index Implementation:** Replace "Full Table Scans" with "Index Scans." Ensure every column used in a `WHERE` clause or a `JOIN` condition is indexed.
2. **Fixing Nested Loops:** When joining large tables, the database needs indexes to avoid exponential slowdowns. An index turns a brute-force search into a high-speed B-Tree lookup.
3. **Query Refactoring:** Avoid `SELECT *`. Only fetch the columns required for the GET response to reduce I/O and memory usage.

### Visualizing the Thread Queue
When your pool size is too small or connections are leaked, your Java threads enter a **BLOCKED** or **WAITING** state.



#### How to verify in Java:
* **JConsole / JVisualVM:** Connect to your running App and look at the "Threads" tab. If you see many threads in `parking` or `waiting` state on `HikariPool.getConnection`, your pool is exhausted.
* **Actuator (Spring):** Hit the `/actuator/metrics/hikaricp.connections.pending` endpoint. If "pending" is greater than 0, users are waiting for connections.

---

### 4. Why 10-20 connections is usually enough
In Java, many developers think they need `maximum-pool-size=100`. However, a single database disk can only do so much I/O at once.

* **Too many connections:** Leads to **"Context Switching."** The DB spends more time swapping tasks than actually executing your SQL.
* **The Sweet Spot:** Start with **10**. If your leak-detection is clean but you still see connection-timeout errors under heavy load, increase it by 5 at a time.

## Database Optimization
If the database is the bottleneck, follow these steps to identify the root cause:

### A. Analyze the Execution Plan
Run an `EXPLAIN ANALYZE` on the query being used by the GET API. Look for these red flags:
* **Sequential Scans:** If you see "Full Table Scan," you are missing an **Index**.
* **Nested Loops:** Large joins without proper indexes can cause exponential slowdowns as data grows.



### B. Connection Pooling
Check if your application is waiting to get a database connection. If your traffic increased but your `max_connections` stayed the same, threads will "queue" waiting for a turn to access the database.

---

## 3. Optimization Strategies
Once you find the "where," apply these specific fixes based on the scenario:

| Strategy | When to Use |
| :--- | :--- |
| **Indexing** | When `EXPLAIN` shows full table scans on filtered columns (`WHERE` or `JOIN`). |
| **Caching (Redis)** | For data that doesn't change every second (e.g., product details, user profiles). |
| **Pagination** | If your GET API is returning 1000+ rows. Use `LIMIT` and `OFFSET` or Cursor-based pagination. |
| **Asynchronous Processing** | If the GET API is doing "work" (like generating a PDF or logging to a slow system), move that to a background worker. |
| **Eager Loading** | If you have an "N+1" problem (1 query to get a list, then 100 queries to get details for each item). |

---

## 4. Troubleshooting Checklist
If the code and queries look fine, check these environmental factors:

* **Check for "Noisy Neighbors":** Is another process on the same server (or a different container on the same host) hogging CPU/IO resources?
* **Check Data Volume:** Did the table grow significantly (e.g., from 10k rows to 1M rows) recently?
* **Check Locks:** Is a long-running CRON job or a heavy `UPDATE` query locking the rows your GET API needs to read?
* **Check Garbage Collection (GC):** In languages like Java or Go, frequent "Stop the World" GC cycles can cause sudden spikes in latency.
---

### Summary Checklist for Java:
- [ ] **Are you using try-with-resources?** (Ensures connections are returned to the pool).
- [ ] **Is leak-detection-threshold enabled in your config?** (Identifies unclosed connections).
- [ ] **Is your maximum-pool-size tuned to your DB CPU cores?** (Prevents database thrashing).
- [ ] **Are you using PreparedStatement?** (Allows the DB to cache the execution plan and prevents SQL injection).
- 
---

## 4. Summary Checklist for Resolution

- [ ] **Step 1:** Check Load Balancer logs for `Target Response Time`.
- [ ] **Step 2:** Run `EXPLAIN ANALYZE` on the API's SQL query to find missing indexes.
- [ ] **Step 3:** Verify that Java threads aren't "waiting" on the connection pool.
- [ ] **Step 4:** Confirm that all database connections are closed using `try-with-resources`.
- [ ] **Step 5:** Monitor CPU/RAM on the database server to ensure it isn't hitting resource limits.