# 🛡️ Idempotent Consumer: Handling Duplicate Messages

> **The Problem:** The Outbox Pattern guarantees that a message is delivered *at least once*. This means your consumer might receive the same "Payment Deducted" event twice. If you process it blindly, you charge the user double.

**Idempotency** ensures that $f(f(x)) = f(x)$. No matter how many times you process the same message, the side effect only happens once.

Here are the two industry-standard ways to implement this: **The Redis Lock (Fast)** and **The Database Deduplication (Safe)**.

---

## 1. Strategy A: Redis `SETNX` (High Performance) ⚡

Use this for high-volume, non-financial data (e.g., sending emails, notifications).

**The Trick:** Use the Redis command `SETNX` (Set if Not Exists). It is atomic.
* If it returns `1` (True): You are the first. Process it.
* If it returns `0` (False): Someone else handled it. Ignore.

### 🐍 Python Implementation (Redis)

```python
import redis

# Connect to Redis
r = redis.Redis(host='localhost', port=6379, db=0)

def process_event(event_id, data):
    # 1. The Atomic Check
    # key: "processed:{event_id}"
    # value: "1"
    # ex: Expires in 24 hours (cleanup)
    # nx: True (Only set if Key does NOT exist)
    is_new = r.set(f"processed:{event_id}", "1", ex=86400, nx=True)

    if not is_new:
        print(f"🛑 Duplicate detected: {event_id}. Skipping.")
        return

    # 2. Process Business Logic
    print(f"✅ Processing event: {data}")
    send_email(data)
```

**⚠️ The Risk:** If your code crashes *after* setting the Redis key but *before* finishing the work, the message is lost forever (Ghost Message).

---

## 2. Strategy B: Database Unique Constraint (Strong Consistency) 🏦

Use this for financial transactions or critical data.

**The Trick:** Use a dedicated table called `processed_messages` with a **Unique Constraint** on the `message_id`. Wrap the check and the business logic in the **same ACID transaction**.

### The Schema (SQL)
```sql
CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT NOW()
);
```

### ☕ Java Implementation (Spring Boot)

This approach creates a "hard barrier" at the database level.

```java
@Service
public class PaymentConsumer {

    @Autowired private PaymentRepository paymentRepo;
    @Autowired private JdbcTemplate jdbcTemplate;

    @Transactional  // <--- ACID Transaction
    public void handlePaymentEvent(PaymentEvent event) {
        String eventId = event.getId();

        try {
            // 1. Try to insert the ID barrier
            // If ID exists, DB throws DuplicateKeyException immediately
            jdbcTemplate.update(
                "INSERT INTO processed_messages (message_id) VALUES (?)", 
                eventId
            );

            // 2. If we survived line 1, process logic
            paymentRepo.deductBalance(event.getUserId(), event.getAmount());
            System.out.println("✅ Processed payment: " + eventId);

        } catch (DuplicateKeyException e) {
            // 3. Catch the duplicate
            System.out.println("🛑 Duplicate detected via DB Constraint: " + eventId);
            // Do nothing, just ack the message
        }
    }
}
```

**✅ The Benefit:** Because of `@Transactional`, if `deductBalance` fails and rolls back, the `processed_messages` insert *also* rolls back. The message remains "unprocessed" and can be retried safely.

---

## 3. Comparison Table 📊

| Feature | Redis (`SETNX`) | Database (Unique Key) |
| :--- | :--- | :--- |
| **Speed** | 🚀 Very Fast (Memory) | 🐢 Slower (Disk I/O) |
| **Consistency** | Weak (Possible Ghost Messages) | Strong (ACID Guaranteed) |
| **Implementation** | Easy | Medium |
| **Best For** | Notifications, Analytics, Logs | Payments, Orders, Inventory |

---

## 💡 Pro Tip: The "Business Key"
Sometimes a technical `message_id` isn't enough (what if the producer sends a *new* message ID for the *same* user action?).

For true safety, use a **Business Key** for idempotency.
* **Bad Key:** `uuid-gen-by-kafka-123`
* **Good Key:** `user_101_order_505_deduction` (A composite key of User + Order).