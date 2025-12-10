# 🛡️ Resiliency Patterns: Keeping Systems Alive

> **The Problem:** In a distributed system, failures are inevitable. A network cable is cut, a database locks up, or a downstream service gets overloaded.
>
> If Service A calls Service B, and Service B is slow, Service A waits... and waits... until it runs out of threads and crashes too. This is called **Cascading Failure**.

**The Solution:** Implement patterns that allow your system to fail gracefully, recover automatically, and prevent one bad service from taking down the whole platform.

Here are the four pillars of resiliency: **Circuit Breaker**, **Retry**, **Bulkhead**, and **Rate Limiter**.

---

## 1. Circuit Breaker Pattern 🔌

**Concept:** Just like an electrical circuit breaker prevents a house fire when there is a short circuit, this pattern prevents your app from wasting resources calling a dead service.



### The Three States:
1.  **CLOSED (Normal):** Traffic flows freely. We count failures.
2.  **OPEN (Tripped):** Failures exceeded the threshold (e.g., 50% failure rate). **All requests are blocked immediately** (Fail Fast) without calling the backend.
3.  **HALF-OPEN (Testing):** After a timeout (e.g., 30s), we let *one* request through to test if the service is back.
    * *Success?* -> Go back to **CLOSED**.
    * *Failure?* -> Go back to **OPEN**.

### Java Example (Resilience4j)
```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackInventory")
public String getInventory(String productId) {
    return inventoryClient.check(productId); // If this fails 50% of the time...
}

// The fallback runs instantly when Circuit is OPEN
public String fallbackInventory(String productId, Throwable t) {
    return "Stock Information Currently Unavailable";
}
```

---

## 2. Retry Pattern 🔄

**Concept:** Sometimes failures are transient (a network blip). Trying again immediately might fix it.

**The Strategy:**
* **Don't retry indefinitely:** You will DDoS your own service.
* **Exponential Backoff:** Wait longer between each try.
    * Attempt 1: Wait 1s
    * Attempt 2: Wait 2s
    * Attempt 3: Wait 4s

### Configuration Example
```yaml
resilience4j:
  retry:
    instances:
      backendA:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        retryExceptions:
          - java.io.IOException # Only retry network errors, not 404s/400s
```

**⚠️ Warning:** Never wrap a **Time-Consuming Transaction** in a retry loop without careful thought. You might make the congestion worse.

---

## 3. Bulkhead Pattern 🚢

**Concept:** In a ship, the hull is divided into watertight compartments (bulkheads). If a torpedo hits one section, water is contained there, and the ship doesn't sink.

In software, we isolate resources (Thread Pools) so that one heavy feature doesn't starve the rest of the app.



### The Scenario:
* Endpoint A: `getImages()` (Slow, high latency).
* Endpoint B: `getUserProfile()` (Fast, critical).

If `getImages` gets 1000 requests, it might use up all Tomcat threads. Now `getUserProfile` cannot be served.

### The Solution:
Assign strict Thread Pool limits.
* `ThreadPool-A` (Images): Max 10 threads.
* `ThreadPool-B` (Profile): Max 20 threads.

If Image Service fills up, only Image requests fail. Profile requests keep working perfectly.

---

## 4. Rate Limiter Pattern 🚦

**Concept:** Control the throughput of traffic sent to or received by a service. It protects your service from getting overwhelmed by a sudden spike (Slashdot effect).

**The Algorithm:** Token Bucket.
* You have a bucket with 10 tokens.
* Every request takes 1 token.
* Tokens replenish at 1 token per second.
* If the bucket is empty, the request is rejected (`429 Too Many Requests`).

### Java Example (Resilience4j)
```java
@RateLimiter(name = "backendA")
public void processRequest() {
    // Only allows N requests per second
    // Excess requests wait or fail immediately based on config
}
```

---

## 5. Timeouts ⏱️

**Concept:** Never wait forever.

The most basic yet most effective pattern. Every external call (Database, API, Cache) **must** have a timeout.

* **Connection Timeout:** Time to establish the handshake (usually fast, e.g., 2s).
* **Read Timeout:** Time to wait for the data payload (depends on logic, e.g., 5s).

**Rule of Thumb:** Your timeout should always be shorter than the timeout of the service calling *you*.

---

## 📝 Summary Comparison

| Pattern | Goal | Implementation |
| :--- | :--- | :--- |
| **Circuit Breaker** | Stop cascading failures. | Fail fast when backend is dead. |
| **Retry** | Handle transient glitches. | Try again with backoff. |
| **Bulkhead** | Resource isolation. | Separate thread pools per feature. |
| **Rate Limiter** | Traffic control. | Cap requests per second. |
| **Timeout** | Prevent hanging threads. | Cut connection after N seconds. |