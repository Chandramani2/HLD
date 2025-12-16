# System Design: Chat System (WhatsApp / Telegram)

## 1. Requirements

### Functional
* **1-on-1 Chat:** Send text, images, videos.
* **Group Chat:** Up to 1024 members.
* **Status (Stories):** Ephemeral content (24h).
* **Sent/Delivered/Read Receipts:** The "Ticks" system.
* **Last Seen / Online Status.**

### Non-Functional
* **Low Latency:** Messages must appear "instant" (< 500ms).
* **High Availability:** 99.999% uptime.
* **Consistency:** Message ordering is critical (Msg A must appear before Msg B).
* **Security:** End-to-End Encryption (E2EE).

## 2. Estimations (Scale)

* **Users:** 2 Billion MAU.
* **Messages:** 100 Billion messages per day.
* **Bandwidth:**
    * Text is small.
    * Media is huge. If 10% messages are images (500KB), storage = Petabytes/day.
* **Constraint:** WhatsApp does **not** store messages permanently on their server (unless undelivered). Once delivered, it is deleted from the server (Store-and-Forward).

---

## 3. High-Level Architecture



WhatsApp uses a **Store-and-Forward** architecture. It acts as a router, not a vault.

| Component | Database/Tech Choice | Reason |
| :--- | :--- | :--- |
| **User Service** | **Mnesia (Erlang DB) / MySQL** | fast user profile lookups. WhatsApp is famous for using Erlang. |
| **Message Queue** | **Custom / Kafka** | Buffer messages if the receiver is offline. |
| **Chat Server** | **Erlang / Elixir (Ejabberd)** | Massive concurrency. Handles millions of open WebSocket connections per node. |
| **Media Storage** | **CDN + S3** | Encrypted blobs (images/video). The server only sends a "thumbnail + link". |

---

## 4. Schema Design

Since messages are deleted after delivery, the database footprint is smaller than Facebook Messenger.

### A. User Table (MySQL/Mnesia)
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY, -- Phone Number
    public_key VARCHAR(512), -- For E2EE
    push_token VARCHAR(256), -- For GCM/APNS notifications
    status_msg VARCHAR(255),
    last_seen TIMESTAMP
);
```

### B. Message Table (Ephemeral Storage)
Used *only* when the receiver is offline.
```sql
CREATE TABLE messages_queue (
    message_id UUID PRIMARY KEY,
    sender_id BIGINT,
    receiver_id BIGINT,
    payload BLOB, -- Encrypted content
    created_at TIMESTAMP,
    expiry TIMESTAMP -- Auto-delete if not delivered in 30 days
);
```

### C. Group Metadata (Cassandra/HBase)
Group info is persistent.
```sql
CREATE TABLE groups (
    group_id UUID PRIMARY KEY,
    created_by BIGINT,
    created_at TIMESTAMP
);

CREATE TABLE group_members (
    group_id UUID,
    user_id BIGINT,
    role VARCHAR(20), -- 'ADMIN', 'MEMBER'
    PRIMARY KEY (group_id, user_id)
);
```

---

## 5. Senior Interview Topics & Logic

### Q1: "How does the 'Sent/Delivered/Read' Tick system work?"
This is an **Acknowledgment (ACK)** protocol over WebSocket.

1.  **User A sends Msg:** Server receives it -> Sends ACK to A (**1 Tick**).
2.  **Server pushes to User B:** User B's app receives it -> Sends ACK to Server.
3.  **Server updates A:** Server notifies A "Delivered" (**2 Gray Ticks**).
4.  **User B opens Chat:** App sends "Read" ACK to Server.
5.  **Server updates A:** Server notifies A "Read" (**2 Blue Ticks**).

**Senior Insight:** "For Group Chats, the server tracks ACKs from *all* participants. The 'Blue Tick' only appears when `count(read_acks) == count(group_members)`."

### Q2: "How do we handle Last Seen / Online Status efficiently?"
**The Problem:** 2 Billion users updating "I am online" every second is a DDoS attack on your DB.
**The Solution:** Heartbeats & Lazy Updates.

1.  **Connection:** User maintains a persistent WebSocket connection.
2.  **Heartbeat:** Client pings server every X seconds. If ping is missed, mark `offline`.
3.  **Lazy Fetch:** We do not broadcast "User A is online" to everyone. We only fetch status when User B *opens* the chat window with User A.

### Q3: "Explain End-to-End Encryption (E2EE) in this context."
**Protocol:** **Signal Protocol** (Double Ratchet Algorithm).

* **Registration:** User A publishes Public Keys to the server.
* **Session Setup:** When A messages B, A fetches B's Public Key.
* **Encryption:** Message is encrypted on Device A. The Server *cannot* read it. It only sees binary garbage.
* **Implication:** We cannot "Search" messages on the server side. Search must happen locally on the user's phone (SQLite).

---

## 6. API & Protocol Design

WhatsApp does **not** use standard HTTP for chatting. It is too slow (overhead of headers).

### Protocol: XMPP (Extensible Messaging and Presence Protocol) or MQTT
* Modified version of XMPP is often used.
* **Binary Protocol:** Lightweight, persistent connection.
* **Format:** Protocol Buffers (Protobuf) instead of JSON (smaller payload).

### Critical Flow: Sending Media
1.  **User A uploads** encrypted image to Media Server (HTTP POST).
2.  **Server returns** a Hash/URL (e.g., `cdn.whatsapp.net/img_123`).
3.  **User A sends** a text message via WebSocket containing the **Thumbnail + URL**.
4.  **User B receives** the text, shows the blurred thumbnail.
5.  **User B clicks** (or auto-download), fetching the image from the CDN.

---

## 7. Scaling Logic (The "C10K Problem" on steroids)

**Problem:** How does one server handle 1 Million open connections?
**Solution:** **Erlang/Elixir (BEAM VM)**.
* Standard servers (Tomcat/Node) struggle with threads.
* Erlang uses "Lightweight Processes" (Actors).
* A single machine can handle ~2 Million concurrent connections.
* This is why WhatsApp had only ~50 engineers for 900M users.

---

## 8. Summary Checklist

* [ ] **Protocol:** Custom XMPP/WebSocket (Persistent Connection).
* [ ] **Database:** Mnesia/MySQL for Users; Cassandra for Group metadata.
* [ ] **Storage:** Ephemeral (Delete after delivery).
* [ ] **Encryption:** Signal Protocol (E2EE).
* [ ] **Media:** Upload to Blob Store -> Send Link in Chat.