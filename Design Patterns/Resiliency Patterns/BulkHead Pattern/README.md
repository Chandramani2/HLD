# Bulkhead Design Pattern in Java

The **Bulkhead Design Pattern** is a fault-tolerance pattern that prevents failures in one part of a system from cascading to the entire system.

## 1. The Concept

The name comes from the partitions (bulkheads) in a ship's hull. If one compartment is compromised and fills with water, the bulkheads prevent the water from flooding the entire ship, allowing it to stay afloat.

In software, this means **isolating resources** (like thread pools or semaphore counts) for different services. If `Service A` is experiencing high latency or failure, it shouldn't exhaust all the threads/connections available for `Service B`.

**Key Benefits:**
* **Fault Isolation:** Failures are contained within a specific component.
* **Resource Preservation:** Prevents a single localized failure from consuming all system resources (CPU, Thread Pools).

---

## 2. Implementation with Resilience4j

This guide uses **Resilience4j**, the industry-standard fault-tolerance library for Java.

### A. Add Dependencies

If you are using Maven and Spring Boot, add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version> </dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### B. Configuration (application.yml)

Resilience4j offers two types of bulkheads:
1.  **SemaphoreBulkhead (Default):** Limits the number of concurrent calls. Good for high throughput.
2.  **ThreadPoolBulkhead:** Uses a fixed thread pool and a bounded queue. Good for isolating blocking operations.

Here is a standard configuration for a **Semaphore-based** bulkhead:

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 10       # Max amount of parallel executions allowed
        maxWaitDuration: 200ms       # Max time to block a thread if the bulkhead is full
      inventoryService:
        maxConcurrentCalls: 5
        maxWaitDuration: 100ms
```

### C. Java Implementation

You can apply the bulkhead using the `@Bulkhead` annotation.

**The Service Layer:**

```java
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;

@Service
public class OrderService {

    // 1. Apply the Bulkhead using the instance name defined in YAML
    // 2. Define a fallback method to handle rejections
    @Bulkhead(name = "paymentService", fallbackMethod = "paymentFallback")
    public String processPayment(String orderId) {
        // Simulate a slow or blocking external call
        simulateSlowCall();
        return "Payment processed for " + orderId;
    }

    // Fallback method must have the same signature + the Exception parameter
    public String paymentFallback(String orderId, io.github.resilience4j.bulkhead.BulkheadFullException ex) {
        // Log the error
        System.out.println("Bulkhead full! Rejection for order: " + orderId);
        
        // Return a default response or try a different path
        return "Payment service is currently busy. Please try again later.";
    }

    // Fallback for unexpected errors (optional)
    public String paymentFallback(String orderId, Exception ex) {
        return "Generic failure for " + orderId;
    }

    private void simulateSlowCall() {
        try { Thread.sleep(500); } catch (InterruptedException e) { }
    }
}
```

---

## 3. ThreadPool Bulkhead Configuration

If you need complete isolation where a service runs in its own dedicated threads (useful if the service blocks threads for a long time), use the **ThreadPoolBulkhead**.

**Configuration (`application.yml`):**

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      externalApi:
        maxThreadPoolSize: 5         # Max threads in the pool
        coreThreadPoolSize: 2        # Core threads always active
        queueCapacity: 10            # Queue size for waiting tasks
        keepAliveDuration: 20ms      # Time to keep idle threads alive
```

**Java Usage (Must return `CompletableFuture`):**

```java
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import java.util.concurrent.CompletableFuture;

@Service
public class ExternalApiService {

    @Bulkhead(name = "externalApi", type = Bulkhead.Type.THREADPOOL, fallbackMethod = "fallback")
    public CompletableFuture<String> callExternalSystem() {
        return CompletableFuture.supplyAsync(() -> {
            // Expensive operation running in a separate thread pool
            return "External Data";
        });
    }

    public CompletableFuture<String> fallback(io.github.resilience4j.bulkhead.BulkheadFullException ex) {
        return CompletableFuture.completedFuture("System busy");
    }
}
```

---

## 4. Comparison of Approaches

| Feature | Semaphore Bulkhead | ThreadPool Bulkhead |
| :--- | :--- | :--- |
| **Mechanism** | Uses semaphores to count permissions. | Uses a separate `ExecutorService` (Thread Pool). |
| **Overhead** | Low (counting only). | Higher (Thread management & context switching). |
| **Use Case** | Protecting internal resources, high throughput. | Isolating slow/blocking I/O calls (database, external APIs). |
| **Behavior** | Rejects calls immediately if limit reached. | Queues tasks; rejects only if pool *and* queue are full. |

### Summary of Key Parameters

* **`maxConcurrentCalls`**: The absolute ceiling of how many requests can run at once.
* **`maxWaitDuration`**: How long a thread will wait to enter the bulkhead if it is full. If this time passes, a `BulkheadFullException` is thrown.
* **`queueCapacity`** (ThreadPool only): How many requests can wait in line before being rejected.