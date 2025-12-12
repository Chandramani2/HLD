# System Design: Scalable Notification Service

## The Goal
Build a system that sends millions of notifications (SMS, Email, Push) per day. It must handle **User Preferences** (opt-outs), **Deduplication** (don't send twice), and **History** (logs).

---

## 1. The Relational Database (SQL)
**Choice:** PostgreSQL or MySQL  
**Purpose:** Service Metadata & User Preferences.

* **Data:** `UserTable` (Email, Phone), `PreferencesTable` (Opt-in/Opt-out flags).
* **Why SQL?**
  * **Data Integrity:** "Unsubscribe" actions must be ACID compliant.
  * **Complex Relations:** Joining Users with their Preferences is efficient.
  * **Low Write Volume:** Users rarely change their settings, so scaling writes isn't the priority.
  * **Structured Data:** A user’s profile (email, phone number) and their settings (e.g., "Mute notifications after 10 PM") are highly structured.
  * **ACID Compliance:** You don't want to lose a user's "Unsubscribe" action.
  * **Scale:** Reads are high (checking preferences), but writes are low (users rarely change settings). SQL handles this easily with **Read Replicas**.

---

## 2. The Key-Value Store (NoSQL)
**Choice:** Redis  
**Purpose:** Deduplication & Rate Limiting.

* **Data:** Ephemeral keys like `sent_alert_101:user_50` or `rate_limit:user_50`.
* **Why Redis?**
  * **Latency:** Checks must happen in sub-millisecond time before processing.
  * **TTL (Time-To-Live):** Data expires automatically (e.g., preventing duplicate alerts for only 10 minutes).
  * **Atomic Counters:** Efficiently tracking "5 SMS per hour" limits.
  * **Speed:** When a request comes in, we must check instantly: *"Did we already send this alert to User A in the last 5 minutes?"*
  * **TTL (Time To Live):** We can set a key `notification_sent:user_123` with a 5-minute expiry. It auto-cleans itself.
  * **Atomic Counters:** We can use `INCR user_123_count` to ensure a user doesn't get spammed (Rate Limiting).

---

## 3. The Wide-Column Store (NoSQL)
**Choice:** Cassandra or DynamoDB  
**Purpose:** Notification Logs / History.

* **Data:** `LogTable` (MessageID, UserID, Status, Timestamp).
* **Why Wide-Column?**
  * **Write Heavy:** The system generates massive amounts of logs (Write > Read).
  * **Linear Scale:** As traffic grows, we add more nodes; SQL cannot handle this insert volume efficiently.
  * **Search Pattern:** We only ever search logs by `UserID`, making the Partition Key obvious.
  * **Massive Writes:** We might send 10 million notifications a day. Storing every single log entry (Sent 'Hello' to User A at 10:00) in SQL would bloat the database and slow it down.
  * **Access Pattern:** We usually only query logs by one key: `UserID`. We rarely need complex joins on logs.
  * **TTL:** Cassandra allows setting TTL on rows, so logs auto-delete after 6 months to save space.

---

## 4. The Message Queue
**Choice:** Apache Kafka or RabbitMQ  
**Purpose:** Decoupling & Throttling.

### Why?
* **Decoupling:** If the "Email Service" (SendGrid/SES) goes down, we don't want to lose the data.
* **Throttling:** The queue buffers the messages until the service is back up, preventing system crashes during traffic spikes.
* **Reliability:** If a worker crashes, the message remains in the queue to be retried.

## Architecture Flow Summary

1.  **Service** receives a trigger ("Order Placed").
2.  **Service** queries **PostgreSQL** to check if User allows SMS.
3.  **Service** checks **Redis** to ensure we aren't spamming the user.
4.  **Service** pushes job to **Kafka**.
5.  **Workers** pull job and call 3rd party API (Twilio).
6.  **Workers** write status ("Success") to **Cassandra** for logging.