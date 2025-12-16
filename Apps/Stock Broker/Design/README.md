# System Design: Stock Broker App (Robinhood / Zerodha)

## 1. Requirements

### Functional
* **User Onboarding:** KYC (Know Your Customer) verification.
* **Trading:** Place Buy/Sell orders (Market, Limit, Stop-Loss).
* **Market Data:** Real-time stock prices (Tickers) and Historical Charts (Candlesticks).
* **Portfolio:** View Holdings, Current Value, and Profit/Loss (P&L).
* **Funds:** Deposit/Withdraw money (Payment Gateway Integration).

### Non-Functional
* **Consistency:** **ACID** is non-negotiable. If a user buys a stock, their cash balance *must* decrease instantly.
* **Security:** High standard (2FA, Encryption, Fraud Detection).
* **Reliability:** The system cannot go down during market hours (9:00 AM - 3:30 PM).
* **Latency:** Low latency for price updates, but order execution depends on the Exchange.

## 2. Estimations (The "Market Open" Surge)

* **Users:** 10 Million active users.
* **Traffic Pattern:** Massive spike at **9:15 AM** (Market Open) and **3:15 PM** (Market Close).
* **Throughput:** 100k orders per second (peak).
* **Market Data:** The Exchange sends millions of price ticks per second. We must filter this before sending it to mobile phones (Data Conflation).

---

## 3. High-Level Architecture

The system is split into two distinct pipelines: **Transactional (Trading)** and **Data Streaming (Market Feed)**.



| Component | Tech Choice | Reason |
| :--- | :--- | :--- |
| **API Gateway** | **Kong / Nginx** | Rate limiting is critical to prevent DDOS during market volatility. |
| **Order Management (OMS)** | **Java / Go** | Validates funds/holdings before routing to the Exchange. |
| **Exchange Connector** | **C++ / Java (FIX Protocol)** | The bridge that talks to Nasdaq/NYSE via TCP/IP. |
| **Market Data Ingest** | **Kafka** | Consumes the massive firehose of price ticks from the Exchange. |
| **Ticker Service** | **Redis + WebSocket** | Broadcasts live prices to millions of connected apps. |
| **Chart Service** | **TimescaleDB / InfluxDB** | Specialized DBs for storing historical price candles. |
| **User/Wallet DB** | **PostgreSQL (Sharded)** | ACID compliance for money and holdings. |

---

## 4. Schema Design (ACID is King)

We use a Relational Database (RDBMS). NoSQL is generally avoided for the core ledger due to eventual consistency risks.

### A. Wallet & Ledger (Double Entry Bookkeeping)
We don't just "update" a balance. We insert a transaction record.

```sql
-- The Source of Truth
CREATE TABLE wallets (
    user_id BIGINT PRIMARY KEY,
    currency VARCHAR(3),
    balance DECIMAL(18, 4),
    locked_balance DECIMAL(18, 4), -- Money locked in open orders
    updated_at TIMESTAMP
);

-- Audit Trail
CREATE TABLE transactions (
    txn_id UUID PRIMARY KEY,
    user_id BIGINT,
    type VARCHAR(20), -- 'DEPOSIT', 'WITHDRAW', 'BUY_DEDUCT', 'SELL_CREDIT'
    amount DECIMAL(18, 4),
    reference_order_id BIGINT
);
```

### B. Holdings (Portfolio)
```sql
CREATE TABLE holdings (
    user_id BIGINT,
    symbol VARCHAR(10),
    quantity INT,
    avg_buy_price DECIMAL(18, 4),
    PRIMARY KEY (user_id, symbol)
);
```

### C. Orders
```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id BIGINT,
    symbol VARCHAR(10),
    type VARCHAR(20), -- 'MARKET', 'LIMIT'
    side VARCHAR(4), -- 'BUY', 'SELL'
    quantity INT,
    price DECIMAL(18, 4),
    status VARCHAR(20), -- 'PENDING', 'SENT_TO_EXCHANGE', 'FILLED', 'REJECTED'
    exchange_order_id VARCHAR(50) -- ID returned by Nasdaq/NYSE
);
```

---

## 5. Senior Interview Topics & Logic

### Q1: "How do you prevent a user from spending the same money twice?" (Double Spending)
**Scenario:** User has $100. They send two "Buy $100 AAPL" requests simultaneously via API.
**Solution:** **Pessimistic Locking (`SELECT ... FOR UPDATE`).**

1.  **Transaction Start:**
2.  `SELECT balance FROM wallets WHERE user_id = 1 FOR UPDATE;` (Locks the row).
3.  **Check:** Is `balance >= 100`?
4.  **Update:** `UPDATE wallets SET balance = balance - 100, locked_balance = locked_balance + 100`.
5.  **Insert:** Create Order record.
6.  **Commit:** Release lock.
7.  **Result:** The second request waits for the lock. When it acquires it, the balance is 0. It fails.

### Q2: "The Exchange sends 100,000 ticks/sec. How do we show this on a mobile screen?" (Data Conflation)
**Problem:** A mobile phone cannot render 60 updates per second. It drains battery and bandwidth.
**Solution:** **Throttling / Conflation.**

1.  **Ingest:** Kafka receives all 100k ticks.
2.  **Processor:** A service aggregates ticks per second.
    * *Logic:* "If AAPL changes from 150.00 -> 150.01 -> 150.05 in 100ms, only send 150.05".
3.  **Push:** Send updates to the Client WebSocket **at most every 200ms-500ms**.

### Q3: "What happens if the Broker crashes after sending an order, but before getting confirmation?"
**Problem:** Order is "FILLED" at Nasdaq, but "PENDING" in your DB. User thinks order failed and tries again.
**Solution:** **Reconciliation (The "Rec" Process).**

1.  **Idempotency:** When sending to Exchange, attach a unique `Client_Order_ID` (UUID).
2.  **Recovery:** On restart, query the Exchange: "What is the status of Order UUID-123?"
3.  **Daily Cron:** Every night, run a script that downloads the "Trade Report" from the Exchange and compares it row-by-row with the internal DB to fix mismatches.

---

## 6. API & Protocol Design

### A. Placing an Order (Idempotency is Key)
* **Method:** `POST /v1/orders`
* **Header:** `X-Idempotency-Key: uuid-v4` (Prevents duplicate orders on network retries).
* **Body:**
    ```json
    {
      "symbol": "TSLA",
      "side": "BUY",
      "quantity": 10,
      "type": "LIMIT",
      "price": 200.50
    }
    ```

### B. Market Data Stream
* **Protocol:** WebSocket.
* **Optimization:** Use **Binary Protobuf** instead of JSON to reduce bandwidth by ~60%.

---

## 7. Security (The "Fintech" Standard)

1.  **Two-Factor Auth (2FA):** Mandatory (TOTP/SMS).
2.  **Encryption:** TLS 1.3 for data in transit; AES-256 for PII (Personally Identifiable Information) at rest.
3.  **Rate Limiting:** Strict limits per API key to prevent bots from crashing the OMS.
4.  **Pinning:** SSL Pinning in mobile apps to prevent Man-in-the-Middle attacks.

---

## 8. Summary Checklist

* [ ] **Database:** RDBMS (Postgres) for core ledger.
* [ ] **Consistency:** Strong (Pessimistic Locking) for funds.
* [ ] **Concurrency:** Idempotency keys for every order.
* [ ] **Market Data:** Conflation (Throttle updates) to save bandwidth.
* [ ] **Protocol:** FIX (to Exchange) / REST + WebSocket (to Client).