# 🛑 The Senior Rate Limiter Implementation Playbook

> **Target Audience:** Backend Engineers, SREs, & Architects  
> **Goal:** Master **Architectural Placement**, **Distributed State**, and **Atomic Implementation** (Lua).

In a senior interview, "Where do I put the Rate Limiter?" is a trick question. The answer depends on whether you are protecting the **Database** (App Level) or the **Network** (Gateway Level).

---

## 📖 Table of Contents
1. [Part 1: Architectural Placement (The 3 Layers)](#-part-1-architectural-placement-the-3-layers)
2. [Part 2: The Logic (Fixed vs. Sliding Window Code)](#-part-2-the-logic-fixed-vs-sliding-window-code)
3. [Part 3: The "Senior" Way (Atomic Lua Script)](#-part-3-the-senior-way-atomic-lua-script)
4. [Part 4: Infrastructure Level (Nginx Config)](#-part-4-infrastructure-level-nginx-config)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: Architectural Placement (The 3 Layers)

Where does the logic live?

### 1. Application Side (Middleware)
* **Location:** Inside your code (e.g., Express.js Middleware, Spring Boot Filter).
* **Pros:** granular control. Can limit based on complex business logic (e.g., "User is Premium AND requested `/export`").
* **Cons:** **Scaling.** If you have 100 app instances, they must synchronize via Redis. If Redis is slow, every request is slow.

### 2. API Gateway (The Standard)
* **Location:** Kong, AWS API Gateway, Zuul.
* **Pros:** Offloads logic. Protects backend services completely. Language agnostic.
* **Cons:** Single Point of Failure (SPoF). If the Gateway config is bad, no one gets in.

### 3. Sidecar / Service Mesh (Envoy/Istio)
* **Location:** A helper container running alongside the App container.
* **Pros:** **Distributed.** No central bottleneck. Highly scalable.
* **Cons:** High operational complexity (Kubernetes).



---

## 💻 Part 2: The Logic (Fixed vs. Sliding Window Code)

Let's implement this in **Python** using **Redis**.

### Approach A: Fixed Window Counter (The Naive Way)
* **Logic:** `Key = UserID + CurrentMinute`. Increment. If > Limit, Block.
* **Flaw:** **The Edge Case.** 10 requests at 12:00:59 and 10 requests at 12:01:01.
    * 20 requests in 2 seconds. The backend spikes.

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def is_allowed_fixed(user_id, limit=10):
    # Key changes every minute: "rate:user123:1678900"
    minute_timestamp = int(time.time() // 60)
    key = f"rate:{user_id}:{minute_timestamp}"

    # Atomic Increment
    current_count = r.incr(key)
    
    # Set expiry on first request (Cleanup)
    if current_count == 1:
        r.expire(key, 60)

    if current_count > limit:
        return False
    return True
```

### Approach B: Sliding Window Log (The Precise Way)
* **Logic:** Keep a timestamp of *every* request in a Redis Sorted Set (`ZSET`).
* **Flow:**
    1.  Remove timestamps older than 1 minute.
    2.  Count remaining timestamps.
    3.  If count < limit, add Now.
* **Pros:** Perfectly smooth traffic.
* **Cons:** **Memory Heavy.** Storing timestamps for 1M requests takes a lot of RAM.

---

## ⚛️ Part 3: The "Senior" Way (Atomic Lua Script)

**The Problem with Python/Node code:**
Between "Count remaining timestamps" and "Add Now", another request might come in. **Race Condition.**

**The Solution:** Run the Sliding Window logic inside Redis using **Lua**. Redis guarantees the script runs atomically.

### The Lua Script (`rate_limiter.lua`)

```lua
-- KEYS[1]: The Rate Limit Key (e.g., "rate:user:123")
-- ARGV[1]: Current Timestamp (Microseconds)
-- ARGV[2]: Window Size (e.g., 60000 ms)
-- ARGV[3]: Limit (e.g., 10)

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local clear_before = now - window

-- 1. Remove old requests (Sliding the window)
redis.call('ZREMRANGEBYSCORE', key, 0, clear_before)

-- 2. Count current requests
local count = redis.call('ZCARD', key)

-- 3. Check limit
if count < limit then
    -- Allowed: Add current timestamp
    redis.call('ZADD', key, now, now)
    -- Set TTL for the whole key to save memory (Window + 1 sec)
    redis.call('PEXPIRE', key, window + 1000)
    return 1 -- Allowed
else
    return 0 -- Blocked
end
```

### Calling it from Python

```python
with open("rate_limiter.lua", "r") as f:
    lua_script = f.read()

limit_script = r.register_script(lua_script)

def is_allowed_sliding(user_id, limit=10, window_ms=60000):
    now = int(time.time() * 1000)
    key = f"rate:{user_id}"
    
    # 1 Network Call. Atomic execution.
    result = limit_script(keys=[key], args=[now, window_ms, limit])
    
    return result == 1
```

---

## 🛡️ Part 4: Infrastructure Level (Nginx Config)

Sometimes the best code is **no code**. If you just need DDoS protection, use Nginx.

### The Leaky Bucket Config
Nginx uses the Leaky Bucket algorithm. It queues requests and processes them at a steady rate.

```nginx
http {
    # Define a zone named 'api_limit'
    # Storage: 10MB of memory for IP addresses
    # Rate: 10 requests per second
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        location /api/ {
            # Apply the limit
            # burst=20: Allow a temporary spike of 20 requests
            # nodelay: Don't slow them down, just reject if over burst
            limit_req zone=api_limit burst=20 nodelay;
            
            # Custom Error Code
            limit_req_status 429;
        }
    }
}
```

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Headers & UX
**Interviewer:** *"The user gets blocked. How do they know when to try again?"*

* ✅ **Senior Answer:** "**Standard HTTP Headers.**"
    * Don't just return `429 Too Many Requests`.
    * Include:
        * `X-RateLimit-Limit`: 100
        * `X-RateLimit-Remaining`: 0
        * `X-RateLimit-Reset`: 16789999 (Unix Timestamp when the window resets).
    * Clients can read `Reset` and sleep until then.

### Scenario B: Distributed Clock Drift
**Interviewer:** *"Server A thinks it is 12:00:01. Server B thinks it is 12:00:02. Will this break the Redis Lua script?"*

* ✅ **Senior Answer:**
    * If we pass `now` from the Application Server (Python), yes, clock drift matters.
    * **Fix:** Use `redis.call('TIME')` *inside* the Lua script.
    * This uses **Redis's Clock** as the single source of truth.

### Scenario C: Memory Optimization
**Interviewer:** *"Storing timestamps for 1 billion requests (Sliding Window) is too expensive in Redis RAM."*

* ✅ **Senior Answer:** "**Switch to Token Bucket.**"
    * Sliding Window Log = $O(\text{Requests})$ RAM.
    * Token Bucket = $O(1)$ RAM (Just stores 2 integers: `Tokens_Left`, `Last_Refill_Time`).
    * **Trade-off:** Token Bucket is slightly less "smooth" than Sliding Window, but saves 99% RAM at scale.

---

### **Final Checklist**
1.  **Placement:** API Gateway for protection, App for business logic.
2.  **Concurrency:** Use Lua Scripts to ensure atomicity.
3.  **Algorithm:** Sliding Window (Accuracy) vs Token Bucket (Memory).
4.  **Protocol:** Always return `Retry-After` headers.

**This concludes the Rate Limiter Implementation Playbook.**