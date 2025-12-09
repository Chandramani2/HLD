# 🛑 The Senior Rate Limiter Playbook

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Design a system that stops abuse without slowing down legitimate users.

In a senior interview, a Rate Limiter is not just `if (count > 10) return false`. It involves **distributed counting**, **race conditions**, **memory optimization**, and **latency reduction**.

---

## 📖 Table of Contents
1. [Part 1: Where to Place It?](#-part-1-architecture--placement)
2. [Part 2: The Algorithms (The Core Decision)](#-part-2-the-algorithms-choose-wisely)
3. [Part 3: Distributed Challenges (Race Conditions)](#-part-3-distributed-challenges-race-conditions)
4. [Part 4: Optimization & Performance](#-part-4-optimization--performance)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🧱 Part 1: Architecture & Placement

Where does the Rate Limiter live? The answer depends on your infrastructure.

| Placement | Pros | Cons |
| :--- | :--- | :--- |
| **Client Side** | Easy to implement. | Unreliable (easy to forge/bypass). **Never trust the client.** |
| **Application Side** | High control; Custom logic. | Bloats application code; Hard to scale memory across instances. |
| **API Gateway (Middleware)** | **Best Practice.** Centralized; Language agnostic; Protects backend completely. | Single point of failure (needs high availability). |

> **💡 Senior Tip:** In a microservices architecture, place the Rate Limiter in the **API Gateway** (e.g., Kong, Nginx, or a custom service) so that downstream services (Order, Payment) don't have to worry about counting requests.

---

## ⚡ Part 2: The Algorithms (Choose Wisely)

Interviews often ask you to compare these. Know the trade-offs.

### 1. Token Bucket 🪣
* **Concept:** A bucket gets tokens added at a fixed rate (e.g., 5 tokens/sec). Each request consumes 1 token. If empty, request is dropped.
* **Pros:** Allows **Bursts** of traffic (if bucket is full, 10 requests can pass instantly). Memory efficient.
* **Cons:** Slightly complex to implement distributedly.
* **Used By:** Amazon, Stripe.



### 2. Leaky Bucket 💧
* **Concept:** Requests enter a queue (bucket) and are processed at a constant fixed rate. If queue is full, new requests are dropped.
* **Pros:** Smooths out traffic (Traffic Shaping).
* **Cons:** **No Bursts allowed**. If 10 users click at once, they must wait in line. Not ideal for real-time user interaction.
* **Used By:** Nginx, Shopify.

### 3. Fixed Window Counter 🪟
* **Concept:** Counter resets every minute. "Max 10 requests per 12:00-12:01".
* **Pros:** Simple to implement (Atomic Increment).
* **Cons:** **The Edge Case.** If a user sends 10 requests at 12:00:59 and 10 more at 12:01:01, they sent 20 requests in 2 seconds. This spikes the backend.

### 4. Sliding Window Log 📜
* **Concept:** Keep a timestamp of *every* request in a sorted set. When a new request comes, remove timestamps older than 1 minute. Count the size of the set.
* **Pros:** Highly accurate. Solves the edge case.
* **Cons:** **Expensive Memory.** Storing timestamps for millions of requests is not scalable.

### 5. Sliding Window Counter (The Hybrid) 🌟
* **Concept:** Breaks time into small chunks (e.g., 10-second blocks). Uses a weighted average of the previous window and current window to calculate rate.
* **Pros:** Memory efficient + Handles edge cases.
* **Cons:** Approximation (but usually good enough).

---

## 🏃 Part 3: Distributed Challenges (Race Conditions)

This is the "Senior" section. How do you implement this across 100 servers?

### The Problem: Race Conditions
Imagine `Limit = 1`.
1.  **Request A** reads counter from Redis (Value: 0).
2.  **Request B** reads counter from Redis (Value: 0).
3.  **Request A** increments to 1 and passes.
4.  **Request B** increments to 1 and passes.
5.  **Result:** 2 requests passed. The limit failed.

### Solution 1: Redis `INCR` (Atomic)
Redis operations are atomic. If you use Fixed Window, `INCR` works perfectly. But for Token Buckets or Sliding Windows, you need to read-check-write.

### Solution 2: Lua Scripts (The Gold Standard)
Send a small script to Redis that performs the logic (Check Time -> Calculate Tokens -> Decrement) on the server side.
* **Why?** Redis is single-threaded. The Lua script executes atomically. No other request can interrupt it.
* **Efficiency:** Reduces network round trips (1 call instead of 3).

---

## 🚀 Part 4: Optimization & Performance

Rate Limiters add latency. You must minimize it.

### 1. Storage: Redis vs. In-Memory
* **Redis:** Centralized truth. slightly slower (network call). Necessary for distributed systems.
* **In-Memory (Guava):** Super fast. But, limits are local. (If you have 10 servers and limit is 100, the total limit effectively becomes 1000).

### 2. Multi-Data Center Replication
If your user is in India, don't check a Redis in US-East.
* **Strategy:** Use Eventually Consistent synchronization between data centers.
* **Trade-off:** You might allow slightly more traffic than the limit for a few seconds, but latency is low.

### 3. The Protocol (HTTP Headers)
Don't just return `429 Too Many Requests`. Be helpful.
* `X-Ratelimit-Limit`: 100
* `X-Ratelimit-Remaining`: 25
* `Retry-After`: 30 (seconds)

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Hard" vs "Soft" Limit
**Interviewer:** *"We don't want to block users immediately if they go slightly over. How do we handle this?"*

* ✅ **Senior Answer:** "Implement **Soft Throttling**."
    * Up to limit 100: Process normally.
    * 101-110: Allow request, but insert a deliberate delay (e.g., `sleep(200ms)`) to slow the user down.
    * 111+: Hard block (429 Error).

### Scenario B: "DoS Attack" vs "High Usage"
**Interviewer:** *"How do you distinguish between a scraper and a heavy user?"*

* ✅ **Senior Answer:** "Tiered Rate Limiting."
    * **Layer 1 (IP Level):** Limit to 10,000 req/day (Prevents DDoS).
    * **Layer 2 (User ID):** Limit to 100 req/minute (Business Logic).
    * **Layer 3 (Endpoint):** Limit `/checkout` to 5 req/minute (Specific abuse prevention).

### Scenario C: The Redis Bottleneck
**Interviewer:** *"We have 100M active users. Checking Redis for every request is adding too much latency. What do you do?"*

* ❌ **Bad Answer:** "Buy a bigger Redis."
* ✅ **Senior Answer:** "**Local Batching / Relaxed Consistency.**"
    * Track the count in the application server's local memory for small periods (e.g., 5 seconds).
    * Sync to Redis every 5 seconds.
    * **Trade-off:** The limit isn't exact (you might allow 105 instead of 100), but you reduce Redis load by 90%.

### Scenario D: Design for "Fairness"
**Interviewer:** *"We have a multi-tenant system. One big customer is using all the API capacity, starving smaller customers."*

* ✅ **Senior Answer:** "Implement **Bulkhead Pattern** or **Weighted Fair Queuing**."
    * Assign separate rate limit pools per Tenant ID.
    * Ensures that Tenant A's usage/limits have zero impact on Tenant B's bucket.

# 🪣 Token Bucket in the Wild: Amazon vs. Stripe

> **Target Audience:** System Architects & Backend Engineers
> **Goal:** Understand how the Token Bucket algorithm powers **AWS EC2 Bursting** and **Stripe's API Reliability**.

The Token Bucket algorithm is the industry standard for handling **Bursty Traffic**. Unlike the Leaky Bucket (which enforces a constant output rate), Token Bucket allows users to save up capacity and spend it all at once.

---

## 📖 Table of Contents
1. [The Algorithm Refresher](#-the-algorithm-refresher)
2. [Case Study A: Amazon EC2 (CPU Credits)](#-case-study-a-amazon-ec2-cpu-credits)
3. [Case Study B: Stripe (API Rate Limiting)](#-case-study-b-stripe-api-rate-limiting)
4. [The Implementation (Redis Lua Script)](#-the-implementation-redis-lua-script)

---

## 🌊 The Algorithm Refresher



1.  **The Bucket:** A container with a maximum capacity (Burst Limit).
2.  **The Refill:** Tokens are added at a fixed rate (e.g., 10 tokens/sec).
3.  **The Consumption:** Every request removes a token.
4.  **The Logic:**
  * If Bucket has tokens -> Request Allowed.
  * If Bucket is empty -> Request Dropped (or Queued).

---

## ☁️ Case Study A: Amazon EC2 (CPU Credits)

Amazon uses Token Buckets to manage **Compute Resources** on T2/T3/T4g "Burstable" instances. This is a fascinating application of the algorithm at the Hypervisor level.

### 1. The Business Problem
Most servers (Web Servers, Dev Environments) sit idle 95% of the time but need 100% CPU for brief moments (Compilation, App Startup). Dedicating 100% CPU constantly is a waste of silicon.

### 2. The Solution: CPU Credits
Amazon treats CPU cycles as tokens.

* **The Bucket:** The "CPU Credit Balance."
* **Refill Rate:** Determined by instance size.
  * *t3.nano:* Earns 6 credits/hour.
  * *t3.large:* Earns 36 credits/hour.
* **Consumption:** 1 CPU Credit = 1 vCPU running at 100% for 1 minute.
* **Burst:** If your website gets a traffic spike, you can burn your saved credits to run at 100% CPU. Once the bucket is empty, you are throttled down to the "Baseline" (Refill Rate).

### 3. Architecture
* **Implementation:** The Hypervisor (Xen/KVM/Nitro) maintains a counter per VM.
* **State:** Stateful. The credit balance persists as long as the instance is running (and not stopped).

> **💡 Senior Takeaway:** Amazon monetizes the Token Bucket. They sell you the *Refill Rate* (Baseline), but give you the *Bucket Size* (Burst) for free to ensure good UX.

---

## 💳 Case Study B: Stripe (API Rate Limiting)

Stripe uses Token Buckets to protect their API from **DDoS attacks** and **"Noisy Neighbors,"** ensuring reliability for payments.

### 1. The Business Problem
If a merchant runs a script that hammers Stripe's API with 10,000 requests/second, it could slow down payment processing for everyone else.

### 2. The Solution: Multi-Layered Buckets
Stripe doesn't just use one bucket. They use a **Hierarchical Token Bucket** strategy.

#### Layer 1: Request Rate Limiter (The Standard)
* **Definition:** "Limit users to 100 requests per second."
* **Implementation:** Token Bucket.
* **Storage:** Redis.
* **Burst:** Allowed. If a user sends 50 requests instantly, it's fine, as long as they have tokens.

#### Layer 2: Concurrent Request Limiter (The Bulkhead)
* **Definition:** "Limit users to 20 *active* heavy write operations at once."
* **Why?** Some endpoints (e.g., `POST /charges`) are CPU intensive. Even 100 req/s might be too much if they all take 5 seconds to process.
* **Implementation:** Token Bucket where tokens are "returned" only when the request finishes.

### 3. Architecture (Distributed Scalability)
Stripe cannot use a single server to count tokens. They use **Redis** with **Lua Scripts**.

* **Cluster:** Rate limiters run on every API node.
* **Sync:** They define a "Shard Key" (API Key or IP). All limits for that key connect to the same Redis shard to ensure atomicity.
* **Optimization:** To prevent "Hot Keys" (everyone hitting the same Redis node), they use high-performance in-memory caching on the API nodes for read-heavy checks, synchronizing with Redis asynchronously.

---

## 💻 The Implementation (Redis Lua Script)

This is how Stripe/Amazon would implement the core logic efficiently. Using Lua ensures the "Read-Check-Write" cycle is atomic.

### The Algorithm
1.  Calculate how many tokens should have been added since the last request.
2.  Add them (capped at Max Burst).
3.  Check if we have enough for the current request.
4.  Deduct and Save.

```lua
-- KEYS[1] = identity (e.g., "rate_limit:user:123")
-- ARGV[1] = refill_rate (tokens per second)
-- ARGV[2] = burst_capacity (max tokens)
-- ARGV[3] = current_timestamp (unix time)
-- ARGV[4] = tokens_requested (usually 1)

local key = KEYS[1]
local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 1. Fetch current state
-- State is stored as hash: { tokens: 5, last_refill: 16789000 }
local state = redis.call("HMGET", key, "tokens", "last_refill")
local last_tokens = tonumber(state[1])
local last_refill = tonumber(state[2])

-- Initialize if missing
if last_tokens == nil then
    last_tokens = capacity
    last_refill = now
end

-- 2. Refill the bucket
-- Calculate time passed
local delta = math.max(0, now - last_refill)
-- Calculate tokens to add: (Time Passed * Rate)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate))

-- 3. Check and Consume
local allowed = false
if filled_tokens >= requested then
    filled_tokens = filled_tokens - requested
    allowed = true
end

-- 4. Save State (Update Last Refill Time to Now)
redis.call("HMSET", key, "tokens", filled_tokens, "last_refill", now)
-- Set Expiry to save Redis memory (e.g., 60 seconds)
redis.call("EXPIRE", key, 60)

return allowed
```
### Why this is "Senior Level":
1.  **Lazy Refill:** We don't have a background process adding tokens every second (that doesn't scale). We calculate the refill math *only* when a request comes in.
2.  **Atomicity:** This script runs as a single operation. No race conditions.
3.  **Memory:** It uses minimal storage (Hash Map) and cleans up after itself (Expire).

---

### **Summary Comparison**

| Feature | Amazon EC2 | Stripe API |
| :--- | :--- | :--- |
| **Resource** | CPU Cycles | HTTP Requests |
| **Refill Rate** | Instance Size (T3.micro) | User Plan (Free/Pro) |
| **Bucket Size** | 24 Hours of Credits | ~Seconds/Minutes of traffic |
| **When Empty** | Throttles to Baseline CPU | Returns HTTP 429 |
| **Implementation** | Hypervisor (C/C++) | Distributed Redis (Lua) |