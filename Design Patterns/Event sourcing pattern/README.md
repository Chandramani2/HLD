# 🎞️ Event Sourcing: The Ultimate Source of Truth

> **The Problem:** In a traditional CRUD database, you only store the **Current State**.
> * User A changes address from "NY" to "LA".
> * The Database overwrites "NY" with "LA".
> * **Data Loss:** You have lost the information that the user *used to live* in NY. You cannot answer "Where did they live on Jan 1st?".

**The Solution:** Do not store the current state. Store **every change** (Event) that has ever happened. The Current State is just a calculation derived by replaying these events.



---

## 1. The Bank Account Analogy 🏦

This is how accountants have worked for 500 years.

* **CRUD Way:** A single column `Balance: $100`.
* **Event Sourcing Way:** A ledger of transactions.
    1.  `AccountCreated` ($0)
    2.  `Deposited` (+$50)
    3.  `Deposited` (+$50)
    4.  `Withdrawn` (-$20)
    5.  **Current State:** $0 + 50 + 50 - 20 = **$80**.

**Key Rule:** The Event Store is **Append-Only**. You never update or delete an event. You only add new ones (e.g., `TransactionReversed`).

---

## 2. The Architecture 🏗️



### A. The Event Store 🗄️
The database that holds the immutable log of events. It replaces your typical SQL tables.
* **Structure:** `AggregateID`, `EventType`, `Timestamp`, `Payload`, `Version`.

### B. The Aggregate (Domain Logic) 🧠
The Java/Python object that applies business rules. It does **not** hold state in a database row. It builds its state in memory by replaying events.

### C. The Projection (Read Model) 📽️
Since replaying 1 million events every time is slow, we create "Projections" (Read Views).
* **Process:** Listen to events -> Update a flat SQL/NoSQL table.
* **Result:** Fast reads (CQRS).

---

## 3. Implementation Logic (Java) 💻

The implementation relies on two core methods: `process()` (Validate command) and `apply()` (Update state).

```java
public class BankAccount {
    
    // Internal State (Transient - rebuilt from memory)
    private String id;
    private int balance = 0;
    
    // 1. Replay History (The Magic)
    public BankAccount(List<Event> history) {
        for (Event e : history) {
            apply(e); // Rebuild state from scratch
        }
    }

    // 2. Handle Command (Validate Business Rules)
    public List<Event> withdraw(int amount) {
        if (this.balance < amount) {
            throw new RuntimeException("Insufficient Funds!");
        }
        // Create the event (Do not change state yet!)
        return List.of(new MoneyWithdrawn(this.id, amount));
    }

    // 3. Apply Event (State Transition)
    private void apply(Event event) {
        if (event instanceof MoneyDeposited) {
            this.balance += ((MoneyDeposited) event).amount;
        } else if (event instanceof MoneyWithdrawn) {
            this.balance -= ((MoneyWithdrawn) event).amount;
        }
    }
}
```

---

## 4. The Performance Fix: Snapshots 📸

Replaying 10,000 events to get a balance is slow.
**Solution:** Every 100 events, save a **Snapshot** of the current state.

* **Load Logic:**
    1.  Load latest Snapshot (Version 100).
    2.  Load Events *after* Version 100 (101, 102, 103).
    3.  Apply (Snapshot + 3 Events).
    4.  **Result:** Massive speedup.

---

## 5. Pros and Cons ⚖️

| Feature | Event Sourcing | Traditional CRUD |
| :--- | :--- | :--- |
| **Audit Log** | Native (100% History). | Hard (Requires separate logging). |
| **Debugging** | Time Travel (Replay to exact moment of bug). | Impossible (State is lost). |
| **Performance** | fast Writes (Append only). Slow Reads (Replay). | Fast Reads. Slow Writes (Locks). |
| **Complexity** | 🔴 Very High. | 🟢 Low. |
| **Schema Changes** | Hard (Events are immutable). | Easy (Alter Table). |

---

## 6. When to use it? ✅

* **Financial Systems:** Ledgers, Crypto exchanges (Auditability is law).
* **Version Control:** Git is essentially an event-sourced file system.
* **Legal/Medical:** Where "who changed what and when" is critical.
* **Gaming:** Replaying a match from the inputs.

**Do NOT use for:** Simple CRUD apps (Blogs, E-commerce profiles). It is over-engineering.

---

## 📝 Summary Checklist

| Component | Responsibility |
| :--- | :--- |
| **Event** | Immutable fact in the past (`UserMoved`). |
| **Command** | Request to do something (`MoveUser`). |
| **Aggregate** | The logic that decides if a Command is valid. |
| **Replay** | The process of rebuilding state from events. |
| **Snapshot** | Optimization to prevent slow replays. |