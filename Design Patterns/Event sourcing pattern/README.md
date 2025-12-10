# ⚡ CQRS: Command Query Responsibility Segregation

> **The Problem:** In a traditional architecture (CRUD), the same data model is used to **Read** and **Write**.
> * Complex queries require many `JOINs`, slowing down the database.
> * Writing complex business logic requires strict validation, slowing down updates.
> * Scaling is hard: You have to scale the whole database even if your app is 90% reads and 10% writes.

**The Solution:** Split your application into two distinct parts: **Command** (Write) and **Query** (Read).



---

## 1. The Core Concept: Splitting the Stack 🪓

CQRS separates the "Write Side" from the "Read Side". They can even use **different databases**.

### A. The Command Side (Writes) ✍️
* **Goal:** Enforce business rules and ensure data integrity.
* **Database:** Optimized for transactions (e.g., PostgreSQL, Oracle) – Normalized (3NF).
* **Input:** "Commands" (Imperative verbs: `CreateOrder`, `ShipItem`).
* **Output:** Void (or just an ID/Status). No data is returned.

### B. The Query Side (Reads) 📖
* **Goal:** Fetch data as fast as possible.
* **Database:** Optimized for reading (e.g., Elasticsearch, MongoDB, Redis) – Denormalized (Flat).
* **Input:** "Queries" (Questions: `GetOrderById`, `SearchItems`).
* **Output:** DTOs (Data Transfer Objects).

---

## 2. Synchronization: Eventual Consistency ⏳

If you use two different databases (e.g., Postgres for writing, Elastic for reading), they will get out of sync. You need a synchronization mechanism.

**The Flow:**
1.  **User** sends `CreateOrderCommand`.
2.  **Command Handler** validates and saves to **Write DB**.
3.  **Command Handler** publishes an event: `OrderCreated`.
4.  **Event Handler** listens to the event and updates the **Read DB** (e.g., inserts a document into Elasticsearch).



* **Lag:** There is a tiny delay (milliseconds to seconds) between the Write and the Read being updated. This is **Eventual Consistency**.

---

## 3. Implementation Example (Java/Spring) 💻

### The Command (Write)
Notice it only contains data needed to change state.

```java
// 1. The Command Object
public class BookRoomCommand {
    public String roomId;
    public String userId;
    public LocalDate date;
}

// 2. The Handler
@Service
public class RoomCommandHandler {
    
    @Autowired private RoomRepository writeRepo; // JPA/Postgres
    @Autowired private KafkaTemplate kafka;

    @Transactional
    public void handle(BookRoomCommand cmd) {
        // Validation logic
        if (writeRepo.isBooked(cmd.roomId, cmd.date)) {
             throw new RuntimeException("Room occupied!");
        }
        
        // Write to SQL
        RoomBooking booking = new RoomBooking(cmd.roomId, cmd.userId);
        writeRepo.save(booking);
        
        // Sync to Read Side
        kafka.send("room-events", new RoomBookedEvent(cmd.roomId));
    }
}
```

### The Query (Read)
Notice it goes straight to the optimized NoSQL store, bypassing complex logic.

```java
@Service
public class RoomQueryService {

    @Autowired private MongoTemplate readRepo; // MongoDB

    public RoomView getRoomDetails(String roomId) {
        // Super fast read, no joins, pre-calculated view
        return readRepo.findById(roomId, RoomView.class);
    }
}
```

---

## 4. When to use CQRS? (The Checklist) ✅

CQRS adds significant complexity. Do **not** use it for simple CRUD apps.

| Scenario | Use CQRS? | Reason |
| :--- | :--- | :--- |
| **Simple Admin Panel** | ❌ NO | CRUD is sufficient. Over-engineering. |
| **High Read/Write Disparity** | ✅ YES | e.g., A Tweet (Written once, Read 1M times). |
| **Complex Business Logic** | ✅ YES | Keep domain logic clean in the Command side. |
| **Different Teams** | ✅ YES | One team optimizes Search, another optimizes Logic. |

---

## 5. CQRS vs. Event Sourcing 🤝

People often confuse them. They are best friends but not the same thing.

* **CQRS:** Splitting Read and Write models.
* **Event Sourcing:** Storing the *state* as a sequence of events (e.g., `AccountCreated`, `MoneyDeposited`, `MoneyWithdrawn`) instead of just the current balance (`$50`).

**Combined Power:**
If you use Event Sourcing, the "Write DB" is just an Event Store. The "Read DB" is a projection built by replaying those events. This is the ultimate form of CQRS.

---

## 📝 Summary Checklist

| Component | Responsibility | Technology Example |
| :--- | :--- | :--- |
| **Command Model** | Behavior, Validation, Consistency. | Java/Postgres |
| **Query Model** | Speed, Projection, Search. | Node/Elasticsearch |
| **Synchronizer** | Glue between the two models. | Kafka/RabbitMQ |
| **UI Client** | Knows to read from B and write to A. | React/Angular |