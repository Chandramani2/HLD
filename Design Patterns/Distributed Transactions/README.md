# 🌐 Distributed Transactions: The Complete Patterns Guide

> **The Problem:** In a Monolith, a single database handles ACID transactions (Commit or Rollback). In Microservices, data is spread across multiple databases. How do you ensure data consistency when Service A works but Service B fails?

Here is the structured breakdown of the three major strategies: **Strong Consistency (2PC)**, **Eventual Consistency (Sagas)**, and **Reliability Patterns (Outbox)**.

---

## 1. The "Old School" Way: Two-Phase Commit (2PC) 👮‍♂️

**Type:** Strong Consistency (ACID)  
**Mechanism:** Synchronous / Blocking

This is the traditional approach (XA Transactions). It relies on a central **Coordinator** that talks to all participating services (nodes).



### How it works:
1.  **Phase 1 (Voting / Prepare):** The Coordinator asks all services: *"Can you commit this change?"*
    * Services lock their rows and return `YES` or `NO`.
2.  **Phase 2 (Commit / Rollback):**
    * If **ALL** vote `YES`: Coordinator sends `COMMIT`. Everyone saves.
    * If **ANY** vote `NO` (or timeout): Coordinator sends `ROLLBACK`. Everyone undoes changes.

### ✅ Pros:
* **Data Integrity:** Guarantees strong consistency (Atomic).
* **Simplicity:** From the client's perspective, it looks like one transaction.

### ❌ Cons:
* **Blocking:** If the Coordinator crashes during Phase 1, all services keep their locks held indefinitely (Database locks up).
* **Latency:** The speed is determined by the *slowest* service.
* **Not Cloud Native:** Does not scale well in distributed systems (CAP theorem).

---

## 2. The Modern Standard: The Saga Pattern 🔄

**Type:** Eventual Consistency (BASE)  
**Mechanism:** Asynchronous / Non-Blocking

Instead of one big transaction, a Saga splits the work into a sequence of **Local Transactions**.

**The Golden Rule:** Since you cannot "Rollback" a committed database transaction, you must execute a **Compensating Transaction** (Undo Logic) to reverse the changes.

> **Example:**
> * **Action:** Deduct $100 from Wallet.
> * **Compensation:** Refund $100 to Wallet.

There are two ways to implement Sagas:

### A. Choreography (Event-Driven) 💃
Services talk to each other directly via events (Kafka/RabbitMQ). There is no central manager.



1.  **Order Service** creates order -> Publishes `OrderCreated` event.
2.  **Inventory Service** listens -> Reserves stock -> Publishes `StockReserved` event.
3.  **Payment Service** listens -> Charges card -> Publishes `PaymentProcessed` event.
4.  **Failure:** If Payment fails, it publishes `PaymentFailed`. Other services listen and run their **Compensations** (Release stock, Cancel order).

* **Pros:** Decoupled, simple for small flows.
* **Cons:** Hard to track "who does what" in complex flows (Cyclic dependencies).

### B. Orchestration (Command-Driven) 🎼
A central **Orchestrator** (State Machine) tells services what to do.



1.  **Order Service** sends request to **Orchestrator**.
2.  **Orchestrator** calls Inventory Service ("Reserve Stock").
3.  **Orchestrator** calls Payment Service ("Charge Card").
4.  **Failure:** If Payment fails, the Orchestrator explicitly calls the "Undo" endpoints on Inventory and Order services.

* **Pros:** Central logic, easy to debug, handles complex workflows well.
* **Cons:** The Orchestrator can become a bottleneck (Single Point of Failure).

---

## 3. The "Dual Write" Fix: Transactional Outbox Pattern 📮

**Type:** Reliability Pattern  
**Problem:** In Sagas, you often need to (1) Save to DB and (2) Publish to Kafka. If you save to DB but fail to publish to Kafka, your system breaks.

**The Solution:** Use the Database as a temporary Message Queue.



### Implementation Steps:
1.  **The Transaction:** Inside the *same* database transaction where you update your business data, you insert a record into a dedicated `Outbox` table.
    ```sql
    BEGIN;
    INSERT INTO orders (id, item) VALUES (1, 'Laptop');
    INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', '{...}'); -- Guaranteed to save if Order saves
    COMMIT;
    ```
2.  **The Relay:** A separate process (like **Debezium** or a Poller) reads the `Outbox` table and pushes the message to Kafka.
3.  **The Cleanup:** Once published, delete the row from the `Outbox`.

### ✅ Pros:
* Guarantees **At-Least-Once** delivery.
* Fixes data inconsistencies between DB and Message Broker.

---

## 4. TCC (Try-Confirm-Cancel) 🏨

**Type:** Application-Level 2PC  
**Use Case:** Reservation systems (Hotels, Flights).

1.  **Try:** Reserve the resource (Pending state). The resource is not consumed, just "blocked."
2.  **Confirm:** If all reservations succeed, finalize the booking (Change state to Confirmed).
3.  **Cancel:** If any fail, release the reservation (Delete Pending state).

*Difference from 2PC:* 2PC locks the database row (technical lock). TCC "locks" the resource logically (business status "RESERVED").

---

## 📊 Summary Comparison

| Strategy | Consistency | Scalability | Complexity | Best Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **2PC (XA)** | High (ACID) | Low | Low | Legacy Banking, Monoliths. |
| **Saga (Choreography)** | Eventual | High | Medium | Simple Microservices flows (3-4 steps). |
| **Saga (Orchestration)** | Eventual | High | High | Complex Enterprise workflows. |
| **TCC** | Strong-ish | Medium | High | Booking/Inventory Systems. |