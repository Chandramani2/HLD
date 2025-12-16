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

# Circuit Breaker vs. Bulkhead Pattern in System Design

These two patterns are pillars of **resiliency** in distributed systems. While both prevent catastrophic cascading failures, they solve the problem from completely different angles: **Circuit Breaker** focuses on *stopping requests* to a failing service, while **Bulkhead** focuses on *isolating resources* so one failure doesn't consume the entire system.

## 1. High-Level Comparison

| Feature | Circuit Breaker Pattern | Bulkhead Pattern |
| :--- | :--- | :--- |
| **Primary Goal** | **Fail Fast**: Stop making calls to a service that is already down to prevent waiting and resource waste. | **Containment**: Isolate resources (threads/connections) so a failure in one area doesn't sink the whole system. |
| **Key Mechanism** | **State Switch**: Monitor error rates. If high, "trip" the circuit (open) to block requests instantly. | **Partitioning**: Assign fixed resource pools (e.g., thread pools) to different dependencies. |
| **Real-world Analogy** | **Electrical Fuse**: If the current surges, the fuse blows to save the appliance. | **Ship Bulkheads**: If the hull is breached, watertight doors seal that section so the rest of the ship stays afloat. |
| **Recovery** | **Automatic**: It self-heals by testing the connection (Half-Open state) after a timeout. | **Passive**: It doesn't "heal" the dependency; it just preserves the caller's remaining resources. |
| **Best For** | Handling **downstream outages** or massive latency spikes. | Handling **resource exhaustion** (e.g., stuck threads) caused by a slow dependency. |

---

## 2. Deep Dive: Circuit Breaker
> **Motto:** "Don't beat a dead horse."

The Circuit Breaker prevents your application from repeatedly trying to execute an operation that's likely to fail. If a downstream service (like a Payment Gateway) is down, your service shouldn't waste time waiting for timeouts on every request.

### How it works
It sits between your service and the external call.
1.  **Closed State:** Traffic flows normally.
2.  **Open State:** After a threshold of failures (e.g., 50% error rate), the breaker trips. All future calls fail *immediately* without reaching the external service.
3.  **Half-Open State:** After a "cool-down" period, it lets a few test requests through. If they succeed, it closes; if they fail, it opens again.

**Example:**
Your E-commerce App calls an Inventory Service. If the Inventory Service crashes, the Circuit Breaker trips. Instead of your users waiting 30 seconds for a timeout, they instantly get a "Stock info unavailable" message, keeping the front end snappy.

---

## 3. Deep Dive: Bulkhead
> **Motto:** "Don't put all your eggs in one basket."

The Bulkhead pattern ensures that if one part of your system consumes all available resources (like thread pools or DB connections), the other parts stay functional. It partitions resources so that a fault in one slice doesn't bring down the whole pie.

### How it works
You create separate pools for different dependencies.
* **Scenario:** You have 100 threads total. You assign 20 to Service A and 20 to Service B.
* **Failure:** If Service A hangs and consumes all 20 of its threads, requests to Service A will fail (queue full).
* **Survival:** Crucially, the other 80 threads (including the 20 for Service B) are untouched. Service B continues to work perfectly.

**Example:**
A video streaming app. You have one service for "Video Player" and another for "User Comments." If the "User Comments" service gets stuck and eats up all its assigned threads, the "Video Player" (critical path) keeps working because it uses a completely different thread pool.

---

## 4. Can they be used together?
**Yes, and they should be.**

They are complementary. You use the **Bulkhead** to ensure that a slow service doesn't eat all your threads, and you use the **Circuit Breaker** to stop calling that slow service entirely if it keeps failing.

### Combined Scenario
1.  **Bulkhead** limits concurrent calls to the "Recommendation Service" to 10 threads.
2.  If the Recommendation Service slows down, it fills those 10 threads.
3.  The **Circuit Breaker** notices that requests are timing out or failing.
4.  The Circuit Breaker "trips," intercepting calls before they even try to enter the Bulkhead pool, saving overhead.

## 📝 Summary Comparison

| Pattern | Goal | Implementation |
| :--- | :--- | :--- |
| **Circuit Breaker** | Stop cascading failures. | Fail fast when backend is dead. |
| **Retry** | Handle transient glitches. | Try again with backoff. |
| **Bulkhead** | Resource isolation. | Separate thread pools per feature. |
| **Rate Limiter** | Traffic control. | Cap requests per second. |
| **Timeout** | Prevent hanging threads. | Cut connection after N seconds. |