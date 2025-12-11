# 📈 The Senior Stock Brokerage System Playbook

> **Target Audience:** Senior Architects & FinTech Leads  
> **Goal:** Design a system with **ACID Compliance**, **Micro-second Latency**, and **Regulatory Audit Trails**.

In a senior interview, you must trade "Scalability" for "Integrity." Eventual consistency is forbidden in the ledger. You must demonstrate knowledge of **Double-Entry Bookkeeping**, **FIX Protocol**, and **Real-Time WebSockets**.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The "Opening Bell" Problem](#-part-1-requirements--the-opening-bell-problem)
2. [Part 2: High-Level Architecture (The 4 Pillars)](#-part-2-high-level-architecture)
3. [Part 3: The Data Plane (The Holy Ledger)](#-part-3-the-data-plane-the-holy-ledger)
4. [Part 4: Real-Time Market Data (Streaming)](#-part-4-real-time-market-data-streaming)
5. [Part 5: Execution & Settlement (The FIX Protocol)](#-part-5-execution--settlement)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The "Opening Bell" Problem

### 1. Requirements
* **Functional:** Buy/Sell Stocks, View Portfolio, Real-time Charts, Add Funds.
* **Non-Functional:**
    * **Strong Consistency:** Balances must be accurate.
    * **Low Latency:** Order placement < 100ms.
    * **Security:** 2FA, Encryption, Fraud Detection.

### 2. The "Opening Bell" Traffic Pattern
Stock apps do not have flat traffic.
* **9:29 AM:** 1M users connected. Low traffic.
* **9:30 AM (Market Open):** **100x spike** in 5 seconds.
* **Implication:** Auto-scaling (AWS ASG) is too slow (takes 3-5 mins to boot servers).
* **Senior Strategy:** **Scheduled Scaling**. Pre-provision 500 servers at 9:00 AM. Spin them down at 11:00 AM.

---

## 🏗️ Part 2: High-Level Architecture

<img width="2816" height="1536" alt="Image" src="https://github.com/user-attachments/assets/95ca0262-78d3-47f8-a5f0-5d5e0893f3a3" />

We segregate the system into **Transactional** (Ordering) and **Streaming** (Data).

### 1. The Core Services
1.  **Order Management System (OMS):** Validates orders, checks balances, holds state.
2.  **Execution Management System (EMS):** Talks to external exchanges (NASDAQ/NYSE).
3.  **Market Data Service:** Ingests price feeds and pushes to clients.
4.  **Ledger Service:** Manages user funds (The Bank).



[Image of Stock Trading System Architecture Diagram]


### 2. The Communication Protocols
* **Client -> Gateway:** HTTPS (REST/gRPC) for Orders.
* **Client <- Gateway:** **WebSockets (WSS)** for Live Prices.
* **Internal Microservices:** gRPC (Low latency).
* **Gateway -> Exchange:** **FIX Protocol** (Financial Information eXchange) over TCP.

---

## 📒 Part 3: The Data Plane (The Holy Ledger)

You never store a user's balance as a simple number: `Balance = $100`.

### 1. Double-Entry Bookkeeping
* **Rule:** Every transaction has a Source (Debit) and Destination (Credit). Sum must be Zero.
* **Table Structure:**
    * `TransactionID`: UUID
    * `Entry 1`: Account: User_Cash | Debit: $100
    * `Entry 2`: Account: Broker_Clearing | Credit: $100
* **Why?** If the DB crashes halfway, the transaction fails. You can never "create" money out of thin air.

### 2. Database Choice
* **Ledger/Orders:** **PostgreSQL / Oracle**.
    * *Must support:* Row-level locking (`SELECT FOR UPDATE`), Foreign Keys, ACID transactions.
* **Historical Data (Charts):** **TimescaleDB / InfluxDB**.
    * Stock prices are time-series data. We need to query "Apple Stock, 5-minute candles, Last Year." NoSQL/Time-Series DBs are optimized for this.

---

## 📡 Part 4: Real-Time Market Data (Streaming)

How do you push Apple's price to 10 million users simultaneously?

### 1. The Pipeline
1.  **Source:** Exchange (NYSE) sends raw UDP multicast feed.
2.  **Ticker Plant:** Internal service that normalizes the data (converts binary to JSON/Protobuf).
3.  **Pub/Sub:** Redis Pub/Sub or Kafka.
4.  **WebSocket Server (Push):** Subscribes to Redis, pushes to User.

### 2. Conflation (Throttling)
* **Problem:** High-Frequency Trading (HFT) generates 100,000 updates/sec. The mobile app cannot render 60fps stock updates (human eye can't see it, battery dies).
* **Senior Solution:** **Conflation**.
    * The backend holds the stream.
    * It sends a snapshot to the user **every 200ms**.
    * If 50 price changes happen in that 200ms, the user only sees the *latest* one.



---

## ⚡ Part 5: Execution & Settlement (The FIX Protocol)

We don't just "hit an API" to buy stocks. We use the industry standard.

### 1. The FIX Protocol
* A text-based protocol over TCP used by every bank/exchange since the 90s.
* **Tag-Value format:** `8=FIX.4.2 | 35=D (New Order) | 55=AAPL | 54=1 (Buy) | 38=100 (Qty)`
* **Why?** Extremely compact, session-based reliability (Sequence numbers ensure no lost packets).

### 2. Settlement (T+1 / T+2)
* When you buy a stock, you don't own it instantly.
* **Trade Date:** The deal is struck.
* **Settlement Date:** Money and Shares actually swap hands (usually 1 or 2 days later).
* **Senior Design:** Your system must handle "Unsettled Cash."
    * `BuyingPower = SettledCash + UnsettledCash`.
    * You allow users to trade with Unsettled Cash (good UX), but you don't let them Withdraw it (Risk Management).

---

## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: The "Negative Balance" Race Condition
**Interviewer:** *"User has $100. They open two tabs. They hit 'Buy $100 Apple' on both tabs at the exact same millisecond. How do you prevent them from buying $200 worth of stock?"*

* ❌ **Junior Answer:** "Check `if (balance >= 100)` then subtract." (Race condition: Both checks pass before subtraction happens).
* ✅ **Senior Answer:** "**Pessimistic Locking (Select for Update).**"
    ```sql
    BEGIN;
    SELECT balance FROM wallets WHERE user_id = 1 FOR UPDATE; 
    -- Database physically locks this row. No other transaction can read/write it.
    -- App calculates: 100 - 100 = 0.
    UPDATE wallets SET balance = 0 ...;
    COMMIT;
    -- Lock released. Second transaction finally runs, sees $0 balance, fails.
    ```

### Scenario B: Idempotency (The Network Timeout)
**Interviewer:** *"I submit a Buy Order. The spinner spins. My internet dies. I don't know if it went through. I click 'Buy' again."*

* **Risk:** The user buys the stock twice.
* ✅ **Senior Answer:** "**Idempotency Key.**"
    1.  Client generates a UUID (`Order-123-ABC`) *before* sending the request.
    2.  Server receives request. Checks Redis/DB: "Have I seen `Order-123-ABC`?"
    3.  **If No:** Process order. Save Key.
    4.  **If Yes:** Return the *previous* success response immediately. Do not execute again.

### Scenario C: Market Closure & Queuing
**Interviewer:** *"A user places an order at 2:00 AM. The market is closed. What happens?"*

* ✅ **Senior Answer:** "**After-Hours Queuing State Machine.**"
    1.  Order is accepted by OMS but marked as `PENDING_OPEN`.
    2.  Order is stored in a durable queue (Kafka/Database).
    3.  **Scheduler:** At 9:30 AM, a "Market Open" event fires.
    4.  The system drains the queue and fires all pending orders to the Exchange via FIX.
    * *Risk:* Price might have changed overnight. We must warn the user: "Market Order executes at the Opening Price, which may differ from yesterday's close."

### Scenario D: Fraud Detection (Pattern Matching)
**Interviewer:** *"How do we spot a user trying to manipulate the market (Pump and Dump)?"*

* ✅ **Senior Answer:** "**Async Stream Analysis (CEP).**"
    * We cannot slow down the trading path.
    * Sidecar pattern: Send a copy of every order to a **Kafka Topic**.
    * **Apache Flink / Kinesis Analytics:** Consumes the stream. Looks for patterns (e.g., "User bought small-cap stock X, then posted on social media, then sold").
    * Flag account for manual review (AML Compliance).

---

### **Final Summary for the Interview**
To win the FinTech interview, remember this mantra:
1.  **Postgres** for Money (ACID).
2.  **WebSockets** for Prices (Speed).
3.  **FIX Protocol** for Exchanges (Standard).
4.  **Idempotency** for Safety (Network reliability).

**Would you like to simulate a specific problem, like "Designing the Margin Trading System"?**