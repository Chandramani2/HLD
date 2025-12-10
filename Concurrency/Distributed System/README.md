# 🌐 Distributed Concurrency Control: The Deep Dive

> **The Problem:** In a single JVM, `synchronized` works because all threads share the same memory space.
> In a Microservices architecture, you have 10 instances of "Order Service" running on different servers. A `synchronized` block on Server A does **not** stop Server B from modifying the same data.

**Distributed Concurrency Control** is the art of coordinating access to shared resources across network boundaries to prevent **Race Conditions**, **Dirty Writes**, and **Lost Updates**.

---

## 1. The Three Main Strategies ⚔️

There is no "one size fits all." You choose based on Contention (how often collisions happen).

### A. Optimistic Locking (Versioning) 🟢
**Philosophy:** "Conflict is rare. I'll proceed, but verify before saving."
* **Mechanism:** Add a `version` column to the database row.
* **Logic:** `UPDATE table SET val = new, ver = 2 WHERE id = 1 AND ver = 1`
* **Result:** If someone else updated it to `ver = 2` already, the row count is 0. Throw exception.
* **Best For:** Low contention (User profiles, unconnected settings).

### B. Pessimistic Locking (Database Locks) 🔴
**Philosophy:** "Conflict is likely. I'll lock the door before I enter."
* **Mechanism:** Use the database's internal locking engine.
* **Logic:** `SELECT * FROM table WHERE id = 1 FOR UPDATE`.
* **Result:** The DB blocks any other transaction trying to read/write this row until you commit.
* **Best For:** High contention, strict financial consistency.

### C. Distributed Locks (External Coordinator) 🔒
**Philosophy:** "We need a global traffic cop."
* **Mechanism:** Use a high-speed store (Redis, ZooKeeper, Etcd) to hold a "lease".
* **Logic:** Service asks Redis "Can I have key `lock:order:1`?"
* **Result:** Only one service holds the key. Others wait or fail.
* **Best For:** Long-running tasks, non-database resources (e.g., generating a report, accessing a file).



---

## 2. Implementation: Java & Spring Boot 💻

### Strategy A: Optimistic Locking (JPA)

Spring Data JPA handles this automatically with `@Version`.

```java
@Entity
public class Product {
    @Id private Long id;
    private int stock;
    
    @Version // <--- The Magic
    private Long version;
}

@Service
public class InventoryService {
    @Autowired private ProductRepository repo;

    public void decreaseStock(Long id) {
        try {
            Product p = repo.findById(id).orElseThrow();
            p.setStock(p.getStock() - 1);
            repo.save(p);
        } catch (ObjectOptimisticLockingFailureException e) {
            // Handle the collision: Retry or fail
            throw new RuntimeException("Someone else bought this item first!");
        }
    }
}
```

### Strategy B: Distributed Lock (Redis + Redisson)

Do **not** write your own Redis lock using `SETNX`. It is buggy (no automatic renewal, no wait logic). Use **Redisson**.

**The "Watchdog" Feature:** If your code is slow, Redisson automatically extends the lock time-to-live (TTL) so it doesn't expire while you are working.

```java
@Service
public class ReportService {

    @Autowired private RedissonClient redisson;

    public void generateComplexReport(String reportId) {
        // 1. Get a lock object based on ID
        RLock lock = redisson.getLock("lock:report:" + reportId);

        try {
            // 2. Try to lock (Wait up to 10s, Auto-unlock after 60s if app crashes)
            // Ideally, leave leaseTime -1 to enable Watchdog auto-extension
            boolean isLocked = lock.tryLock(10, 60, TimeUnit.SECONDS);

            if (isLocked) {
                try {
                    // Critical Section: Only one server runs this at a time
                    System.out.println("Generating PDF...");
                    Thread.sleep(5000); 
                } finally {
                    lock.unlock(); // Always unlock!
                }
            } else {
                throw new RuntimeException("Could not acquire lock, try later");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## 3. The "Fencing Token" Problem 🤺

**The Scenario:**
1.  **Client A** gets a lock from Redis.
2.  **Client A** pauses (GC Pause) for 5 minutes. Lock expires.
3.  **Client B** gets the lock and writes to Storage.
4.  **Client A** wakes up and writes to Storage, **overwriting Client B**.

**The Solution:** The Storage must check a **Fencing Token** (a monotonically increasing number).
1.  Redisson returns a token `33` to Client A.
2.  Redisson returns a token `34` to Client B.
3.  Storage rejects write from A because `33 < 34`.

---

# 🎓 Senior Developer Interview Q&A (15-20 Questions)

These questions test your understanding of distributed consensus, failure modes, and CAP theorem trade-offs.

### 🔥 Section 1: Architecture & Strategy

**Q1: Optimistic vs. Pessimistic Locking: How do you choose?**
* **A:**
    * **Optimistic:** High throughput, low contention. Use when retries are cheap. (e.g., Edit Profile).
    * **Pessimistic:** Strict consistency, high contention. Use when retries are expensive or impossible. (e.g., Deducting Balance).

**Q2: Why can't I just use `synchronized` in a clustered environment?**
* **A:** `synchronized` locks the monitor of the object inside the specific JVM heap. It has zero visibility into other JVMs running on other servers.

**Q3: What are the downsides of using a Database for distributed locking?**
* **A:**
    1.  **Performance:** DB connections are expensive and limited. Holding connections for locks reduces throughput.
    2.  **No TTL:** If the app crashes holding a DB lock, you need manual DBA intervention to kill the session (unless you implement complex timeout logic).

**Q4: Explain the "Lost Update" problem.**
* **A:** Transaction A reads row (val=10). Transaction B reads row (val=10). A writes 11. B writes 11. The correct value should be 12. B's write "lost" A's update. Prevented by locking.

---

### 🗝️ Section 2: Redis & Redlock

**Q5: How does the basic Redis lock (`SETNX`) work? Why is it insufficient?**
* **A:** `SET key value NX PX 10000`. It sets the key only if it doesn't exist, with an expiry.
    * *Insufficient because:* (1) No reentrancy. (2) No spin-lock (wait) mechanism. (3) If the task takes longer than expiry, the lock releases early (unsafe). (4) No fencing token.

**Q6: What is the "Redlock" Algorithm?**
* **A:** A distributed locking algorithm for Redis *Clusters*.
    * It tries to acquire the lock on N independent master nodes.
    * It succeeds if it acquires the lock on N/2 + 1 nodes within a small time window.
    * *Controversy:* Martin Kleppmann (DDIA author) argues Redlock is not safe for hard consistency due to clock drift issues.

**Q7: What is the "Watchdog" in Redisson?**
* **A:** It is a background thread that starts when a lock is acquired without a specific `leaseTime`. It periodically renews the lock's expiration (e.g., every 10s) as long as the thread holding the lock is still alive. This prevents the lock from expiring if the task takes longer than expected.

**Q8: How do you handle "Clock Drift" in distributed locking?**
* **A:** If Server A's clock runs fast, it might think a lock expired before Server B does.
    * *Mitigation:* Use relative time (TTL) rather than absolute timestamps. Use Fencing Tokens for storage validation.

---

### 🐘 Section 3: ZooKeeper & CP Systems

**Q9: Why is ZooKeeper considered "safer" than Redis for locking?**
* **A:** ZooKeeper is a **CP** system (Consistent, Partition Tolerant). Redis is generally **AP** (Available) or eventually consistent.
    * ZK uses **Ephemeral Nodes**. If the client disconnects (session dies), the node is deleted automatically by ZK. It guarantees strict ordering.

**Q10: Explain the "Thundering Herd" problem in ZooKeeper locking.**
* **A:** If 1000 clients watch a lock node, and it gets deleted, all 1000 get notified and try to grab it.
    * *Fix:* Use **Sequential Ephemeral Nodes**. Client N only watches Client N-1. When N-1 is deleted, only N is notified.

---

### ☠️ Section 4: Edge Cases & Failure Modes

**Q11: What happens if the Redis node holding the lock crashes?**
* **A:**
    * *Single Node:* The lock is lost. Another client might acquire it immediately, violating mutual exclusion.
    * *Master-Slave:* If the lock wasn't replicated to the Slave yet, and failover happens, the new Master doesn't have the lock. Mutual exclusion is violated. (This is why Redlock exists).

**Q12: How do you ensure a Lock is released if the application crashes?**
* **A:** Always set a **TTL (Time To Live)** or Lease Time. The lock system (Redis/ZK) will auto-delete the key after X seconds.

**Q13: What is a "Fencing Token" and why is it needed?**
* **A:** It handles the case where a Lock expires due to a GC Pause, but the application wakes up and tries to write data anyway. The Storage rejects the write because the token (e.g., 33) is lower than the current token (34) generated by the new lock holder.

**Q14: Deadlock in Distributed Systems: How to detect?**
* **A:** Extremely hard to detect across services.
    * *Prevention:* Always acquire locks in a consistent order (Resource A then Resource B). Use timeouts (`tryLock(time)`) so threads don't wait forever.

**Q15: Isolation Levels: Read Committed vs. Repeatable Read.**
* **A:**
    * *Read Committed:* You can see data committed by others while your transaction is running. (Susceptible to Non-Repeatable Reads).
    * *Repeatable Read:* You see a snapshot of data as it was at the start of your transaction. (Standard for most financial logic).

**Q16: Can you implement a Distributed Semaphore?**
* **A:** Yes. Using Redis `INCR` or ZK counters. Useful for Rate Limiting (e.g., "Only allow 5 concurrent report generations across the entire cluster").

**Q17: What is "Split Brain" and how does it affect locking?**
* **A:** Network partition divides the cluster into two. If both sides elect a leader/master, you might have two clients holding the "same" lock.
    * *Fix:* Quorum-based systems (ZooKeeper/Etcd) require a majority vote (N/2 + 1) to function, ensuring only one side survives.