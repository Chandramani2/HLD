# System Design: Stock Exchange Platform

## 1. Requirements

### Functional
* **Place Order:** Limit Orders ("Buy 100 AAPL at $150") and Market Orders ("Buy 100 AAPL Now").
* **Cancel Order:** Cancel an open order before it executes.
* **Matching Engine:** Match Buy and Sell orders automatically.
* **Market Data:** Broadcast real-time price updates (Ticker).
* **Order History:** View past trades and portfolio.

### Non-Functional
* **Ultra-Low Latency:** Execution must happen in **microseconds** ($\mu s$). Standard DBs are too slow.
* **High Throughput:** Handle millions of orders per second (especially at Market Open/Close).
* **ACID Compliance:** Financial data must be strictly consistent. No money can disappear.
* **Fairness:** First-Come-First-Serve (FIFO) within the same price price point.

## 2. Estimations (The High-Frequency Scale)

* **Users:** Millions of retail, but **High-Frequency Trading (HFT) bots** generate 80% of volume.
* **Throughput:** 1 Million orders per second (peak).
* **Latency Target:** End-to-end processing < 500 $\mu s$ (microseconds).
* **Storage:**
    * Active Order Book: Fits in **RAM** (In-Memory).
    * Historical Trades: **Petabytes** (Write Once, Read Many).

---

## 3. High-Level Architecture



[Image of stock exchange system architecture]


The architecture is split into the **Transactional Path** (Trading) and the **Data Path** (Ticker).

| Component | Tech Choice | Reason |
| :--- | :--- | :--- |
| **Order Gateway** | **C++ / Java / Rust** | Validation and Protocol Translation (FIX to Internal Binary). |
| **Matching Engine** | **In-Memory (Single Threaded)** | The Core. No database locks. Uses **LMAX Disruptor** pattern. |
| **Market Data** | **UDP Multicast** | Broadcasting prices to millions of listeners efficiently. |
| **Persistence** | **Append-Only Log (WAL)** | Write to disk strictly for recovery (Kafka/Journal). |
| **History DB** | **TimescaleDB / KDB+** | Specialized Time-Series databases for financial queries. |

---

## 4. Schema & Data Structures (The "Secret Sauce")

Traditional SQL/NoSQL is **not** used for the live Order Book. It is too slow.

### A. The Limit Order Book (In-Memory Data Structure)
We need to sort orders by **Price** (Best Price First) and then by **Time** (FIFO).

* **Bids (Buy Orders):** Max-Heap or TreeMap (Highest price at top).
* **Asks (Sell Orders):** Min-Heap or TreeMap (Lowest price at top).
* **Best Practice:** **Red-Black Tree** or **Skip List**.
    * *Why?* Heaps are fast ($O(1)$) to peek top, but slow ($O(N)$) to cancel an order in the middle. Trees allow $O(\log N)$ search, insert, and delete.

### B. Persistent Schema (Post-Trade)
Once a trade happens, it is flushed to a database (PostgreSQL/Timescale) for reporting.

```sql
-- Executed Trades
CREATE TABLE trade_history (
    trade_id BIGSERIAL PRIMARY KEY,
    symbol VARCHAR(10),
    buyer_order_id BIGINT,
    seller_order_id BIGINT,
    price DECIMAL(18, 4), -- NEVER use Float for money
    quantity INT,
    executed_at TIMESTAMP
);

-- Wallet / Balance (Strong ACID)
CREATE TABLE balances (
    user_id BIGINT,
    currency VARCHAR(10),
    amount DECIMAL(18, 4),
    locked_amount DECIMAL(18, 4), -- Locked for open orders
    PRIMARY KEY (user_id, currency)
);
```

---

## 5. Senior Interview Topics & Logic

### Q1: "How do you handle Floating Point errors?"
**The Trap:** "I use `float` or `double`." -> **Immediate Fail.**
**The Senior Answer:**
"Computers cannot represent `0.1` accurately in binary. In Finance, we use:
1.  **Decimal Data Types:** `BigDecimal` (Java) or `DECIMAL` (SQL).
2.  **Integers (Micros):** Store everything as integers.
    * Example: \$150.25 is stored as `1502500` (scaled by 10,000). All math is integer math."

### Q2: "How do you achieve Microsecond Latency (Lock-Free Concurrency)?"
**The Trap:** "I use Multithreading with Locks." (Context switching kills performance).
**The Senior Answer:** **LMAX Disruptor / Single-Threaded Event Loop.**
* **Logic:** The Matching Engine runs on a **Single Thread** pinned to a specific CPU Core.
* **Why?** No Context Switching, No OS Scheduling delays, No CPU Cache misses.
* **Throughput:** A single thread can handle 5M+ orders/sec if it never waits for I/O.
* **Persistence:** It writes to an asynchronous Ring Buffer (Disruptor) which a secondary thread writes to disk.

### Q3: "Explain Price-Time Priority Matching."
**Algorithm:**
1.  **Incoming Buy Order:** "Buy 100 shares at \$100".
2.  **Check Ask Queue:** Is there a Seller at $\le \$100$?
    * *Yes:* Match immediately.
    * *No:* Add Buyer to the **Buy Tree**.
3.  **Tie-Breaking:** If two people want to buy at \$100, the one who arrived **earlier** gets priority (FIFO).

---

## 6. API & Protocol Design

REST is too slow for HFT.

### A. FIX Protocol (Financial Information eXchange)
The industry standard for communicating with exchanges.
* **Format:** Tag-Value (e.g., `8=FIX.4.2|35=D|55=AAPL|54=1`...)
* **Transport:** Raw TCP (Socket).

### B. WebSocket (For Retail Clients)
For a generic React/Mobile app (like Robinhood).

* **Subscribe:**
    ```json
    { "type": "subscribe", "symbol": "AAPL" }
    ```
* **Push Update:**
    ```json
    { "symbol": "AAPL", "price": 150.05, "ts": 16788888 }
    ```

### C. Internal Binary Protocol
Inside the data center, services talk via compact binary formats (Protobuf or Struct packing) to save bandwidth.

---

## 7. Reliability & Disaster Recovery

**Problem:** The matching engine is in RAM. What if power fails?
**Solution:** **Event Sourcing (Snapshot + Replay).**

1.  **WAL (Write Ahead Log):** Every incoming order is written to a sequential log file *before* it touches the memory.
2.  **Recovery:** On restart, the engine reads the Log and "replays" every order to rebuild the In-Memory Tree state.
3.  **Active-Passive HA:**
    * **Primary Node:** Matches trades.
    * **Secondary Node:** Reads the same input stream (multicast) and does the same matching but *outputs nothing*.
    * **Failover:** If Primary dies, Secondary takes over instantly because its memory is already hot and synced.

---

## 8. Summary Checklist

* [ ] **Core:** Single-Threaded In-Memory Matching Engine.
* [ ] **Structure:** Red-Black Trees for Order Book.
* [ ] **Math:** Integers/Decimal (No Floats).
* [ ] **Protocol:** FIX over TCP.
* [ ] **Persistence:** Event Sourcing / Append-Only Logs.
* [ ] **Latency:** Avoid Locks, minimize Context Switches.