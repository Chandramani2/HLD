# 🔄 The SAGA Pattern: Transactions in Microservices

> **The Problem:** In a Monolith, you have a single database. You can start a transaction, update 5 tables, and if one fails, the database auto-magically **Rolls Back** everything. ACID is easy.

In Microservices, you have **Database per Service**. You cannot "Rollback" a transaction that happened in a different service's database.

**The Solution:** The SAGA Pattern breaks a big transaction into a sequence of local transactions. If one fails, you must execute **Compensating Transactions** to undo the work already done.

---

## 1. The Core Concept: Forward vs. Backward ⏩⏪

A Saga is defined by two flows:
1.  **The Happy Path (Forward):** executing transactions $T_1, T_2, T_3$.
2.  **The Failure Path (Backward):** executing compensations $C_2, C_1$ to undo changes.

### The "Undo" Logic (Compensation)
You cannot simply "delete" the data because other users might have already seen it. You must apply a logical fix.

| Action ($T$) | Compensation ($C$) |
| :--- | :--- |
| **Create Order** | Cancel Order (State = REJECTED) |
| **Reserve Stock** | Release Stock |
| **Charge Credit Card** | Refund Credit Card |

---

## 2. Approach A: Choreography (The Dance) 💃

**Concept:** There is no central leader. Services talk to each other via **Events** (Kafka/RabbitMQ).
* "I did my part, now I'm telling everyone."

### The Workflow:
1.  **Order Service:** Creates Order (Pending). Publishes `OrderCreated`.
2.  **Inventory Service:** Listens to `OrderCreated`. Reserves items. Publishes `StockReserved`.
3.  **Payment Service:** Listens to `StockReserved`. Tries to charge card.
    * **IF SUCCESS:** Publishes `PaymentSuccess`. Order Service listens and updates state to `CONFIRMED`.
    * **IF FAIL:** Publishes `PaymentFailed`.

### The Compensation (Rollback):
If `PaymentFailed` is published:
1.  **Inventory Service** listens to it -> **Releases Stock**.
2.  **Order Service** listens to it -> **Cancels Order**.

### ✅ Pros & ❌ Cons
* ✅ **Simple:** Good for small flows (2-3 steps).
* ✅ **Decoupled:** Services don't need to know who calls them.
* ❌ **Spaghetti Logic:** In complex flows, it's hard to track "Who listens to what?" Cyclic dependencies are common.

---

## 3. Approach B: Orchestration (The Conductor) 🎼

**Concept:** A central **Coordinator** (State Machine) tells services what to do via **Commands** (REST/gRPC/Queues).

### The Workflow:
1.  **Order Service** starts the `OrderSagaOrchestrator`.
2.  **Orchestrator** sends command `ReserveStock` to Inventory Service.
3.  **Inventory** replies "Success".
4.  **Orchestrator** sends command `ChargePayment` to Payment Service.
5.  **Payment** replies "Failed" (Insufficient Funds).

### The Compensation (Rollback):
The Orchestrator knows exactly where it stopped. It looks at its history: "I successfully did Step 1, but failed Step 2."
1.  **Orchestrator** sends command `ReleaseStock` to Inventory.
2.  **Orchestrator** updates Order Status to `FAILED`.

### ✅ Pros & ❌ Cons
* ✅ **Central Logic:** You can look at one file and see the whole flow.
* ✅ **Management:** Easier to handle timeouts and retries.
* ❌ **Single Point of Failure:** The Orchestrator must be highly available.

---

## 4. Implementation Example (Java/Pseudo-code) 💻

Here is how an **Orchestrator** is typically structured using a "Command" pattern.

```java
public class OrderSagaOrchestrator {

    public void processOrder(Order order) {
        // Step 1: Inventory
        try {
            inventoryClient.reserve(order.getItems());
        } catch (Exception e) {
            // Failed at step 1: Nothing to compensate
            orderService.failOrder(order.getId());
            return;
        }

        // Step 2: Payment
        try {
            paymentClient.charge(order.getTotalAmount());
        } catch (Exception e) {
            // Failed at step 2: MUST compensate Step 1
            inventoryClient.release(order.getItems()); // <--- Compensation
            orderService.failOrder(order.getId());
            return;
        }

        // Step 3: Success
        orderService.completeOrder(order.getId());
    }
}
```

*Note: In production, you wouldn't use simple `try-catch`. You would use a State Machine library (like Spring Statemachine, Camunda, or Netflix Conductor) to persist the state in case the server crashes in the middle.*

---

## 5. Summary: Which one to choose? 📊

| Feature | Choreography | Orchestration |
| :--- | :--- | :--- |
| **Coupling** | Low (Event-based) | High (Command-based) |
| **Complexity** | Increases with more services | Stable |
| **Visibility** | Low (Hard to debug) | High (Central dashboard) |
| **Best For** | Simple flows (3-4 services) | Complex flows (Banking, Enterprise) |