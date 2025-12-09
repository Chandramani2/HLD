# 💬 The Senior System Design Playbook: Design WhatsApp

> **Target Audience:** Senior Backend Engineers & Architects  
> **Goal:** Design a messaging platform with **2 Billion Users**, **Real-Time Delivery**, and **End-to-End Encryption**.

In a senior interview, "Design WhatsApp" is not about a REST API. It is about **WebSockets vs Long Polling**, **The "Fanout" Problem for Groups**, and **Handling Offline Messages** efficiently.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The "Billions" Scale](#-part-1-requirements--the-billions-scale)
2. [Part 2: High-Level Architecture (The Connection Manager)](#-part-2-high-level-architecture-the-connection-manager)
3. [Part 3: 1-on-1 Messaging Flow (The Happy Path)](#-part-3-1-on-1-messaging-flow-the-happy-path)
4. [Part 4: Group Messaging (The Fanout Explosion)](#-part-4-group-messaging-the-fanout-explosion)
5. [Part 5: Status Synchronization (Sent/Delivered/Read)](#-part-5-status-synchronization-sentdeliveredread)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The "Billions" Scale

### 1. Requirements
* **Functional:** 1-on-1 Chat, Group Chat, Sent/Delivered/Read Receipts, Online Status, Image/Video Support.
* **Non-Functional:**
    * **Low Latency:** Real-time experience.
    * **Consistency:** Order of messages matters (Msg A before Msg B).
    * **Durability:** Messages must be delivered eventually, even if the user is offline.

### 2. Back-of-the-Envelope Math
* **DAU:** 2 Billion.
* **Messages:** 100 Billion per day.
* **Peak Traffic:** ~1.2 Million messages/second.
* **Insight:** We cannot open a new HTTP connection for every message. The handshake overhead would kill the servers. We need **Persistent Connections**.

---

## 🏗️ Part 2: High-Level Architecture

The core of a chat app is the **Gateway Service** that holds open connections.

### 1. The Gateway (Chat Server)
* **Protocol:** **WebSocket** (TCP).
    * *Why?* Bi-directional communication. The server needs to *push* messages to the receiver without them asking.
* **Stateful:** This service is **Stateful**.
    * It knows "User A is connected to Box 1".
    * It maintains the `Session Map: { UserID -> ConnectionID }`.

### 2. The Session Manager (Redis / Zookeeper)
* Since we have 10,000 Gateway Servers, how do we know where User B is connected?
* **Distributed Cache:** **Redis** stores the mapping.
    * `User_A -> Gateway_01`
    * `User_B -> Gateway_99`



---

## 📩 Part 3: 1-on-1 Messaging Flow (The Happy Path)

**Scenario:** User A sends "Hello" to User B.

1.  **User A** sends message via WebSocket to **Gateway 01**.
2.  **Gateway 01** calls **Message Service** to persist the message (Cassandra/HBase).
3.  **Gateway 01** queries **Session Manager (Redis)**: "Where is User B?"
    * *Result:* "User B is on **Gateway 99**."
4.  **Gateway 01** forwards message to **Gateway 99** (via internal RPC/gRPC).
5.  **Gateway 99** pushes message down the WebSocket to **User B**.
6.  **User B** sends ACK ("Received") back up the chain.

---

## 📢 Part 4: Group Messaging (The Fanout Explosion)

**Scenario:** User A sends a message to a Group with 500 people.

### 1. The Naive Approach (Client-Side Fanout)
* User A's phone loops 500 times and sends the message to 500 people individually.
* **Fail:** Kills User A's battery and bandwidth.

### 2. Server-Side Fanout (The Senior Solution)
* User A sends **1 message** to the server ("Group ID: 123").
* **Group Service:** Looks up members of Group 123.
* **Message Queue (Kafka):** Pushes the message to a "Group Dispatch" topic.
* **Workers:** Read from Kafka, look up the Gateway for each of the 500 members, and push the message.
* **Optimization:**
    * **Small Groups (< 200):** Direct push (Online fanout).
    * **Mega Groups (> 5k):** Do not push. Send a "New Message Notification" (Lightweight). Client pulls the message when they open the app.

---

## 🚦 Part 5: Status Synchronization (Sent/Delivered/Read)

The checkmarks (✓, ✓✓, Blue ✓✓) are a complex distributed problem.

1.  **Sent (1 Grey Check):** Server receives message, persists to DB, sends ACK to User A.
2.  **Delivered (2 Grey Checks):**
    * Server pushes message to User B.
    * User B's App (background) sends an ACK packet automatically.
    * Server updates DB status -> Pushes "Delivered" event to User A.
3.  **Read (2 Blue Checks):**
    * User B opens the chat UI.
    * App sends "Read" event.
    * Server updates DB -> Pushes "Read" event to User A.

**Storage Optimization:** Do not store a row for every ACK. Update the `LastReadMessageID` in the `Conversation` table.

---

## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: Offline Handling (The "Inbox" Pattern)
**Interviewer:** *"User B is on a flight. User A sends 10 messages. How do we ensure delivery when User B lands?"*

* ✅ **Senior Answer:** "**Temporary Storage + Cursor.**"
    * If User B is disconnected, Gateway writes messages to a **"Unread" Table** (or HBase/Cassandra).
    * When User B reconnects (WebSocket Handshake):
        1.  User B sends `LastReceivedMessageID = 100`.
        2.  Server queries DB: "Fetch messages for User B where ID > 100".
        3.  Server dumps the messages down the socket.
    * This ensures no data loss and correct ordering.

### Scenario B: Database Choice (Chat History)
**Interviewer:** *"We generate billions of messages. MySQL can't handle the write load. What DB do we use?"*

* ✅ **Senior Answer:** "**Wide-Column Store (Cassandra / HBase) or Key-Value (DynamoDB).**"
    * **Why?**
        1.  **Write Speed:** LSM Trees handle massive write ingestion better than B-Trees.
        2.  **Access Pattern:** We always query by `ChatID` + `Timestamp` (Range Query).
    * **Schema:**
        * `Partition Key`: ChatID (Groups conversation together).
        * `Clustering Key`: Timestamp (Sorts messages by time).

### Scenario C: End-to-End Encryption (E2EE)
**Interviewer:** *"How do we ensure even our engineers can't read the messages?"*

* ✅ **Senior Answer:** "**The Signal Protocol (Double Ratchet).**"
    * Encryption happens **on the device**.
    * **Public Keys:** Stored on the server.
    * **Flow:**
        1.  User A fetches User B's Public Key from server.
        2.  User A encrypts message.
        3.  Server routes the encrypted blob (ciphertext). Server sees garbage.
        4.  User B decrypts with Private Key.
    * **Constraint:** Server-side search is impossible. Search must happen locally on the phone (SQLite).

### Scenario D: Last Seen / Online Status
**Interviewer:** *"How do we track who is Online?"*

* ❌ **Bad Answer:** "Update the database every second." (Writes will kill the DB).
* ✅ **Senior Answer:** "**Heartbeats + Redis TTL.**"
    * **Active:** Client sends heartbeat every 30s.
    * **Server:** Updates `User:123:Status` in Redis with `TTL = 40s`.
    * **Check:** If `Redis.get(User:123)` exists, they are Online. If null, they are Offline.
    * **Lag:** Users might appear online for 30-40s after disconnect (acceptable trade-off).

---

### **Final Checklist**
1.  **Protocol:** WebSockets for bidirectional low latency.
2.  **DB:** Cassandra/HBase for Write-Heavy chat logs.
3.  **Groups:** Async Fanout using Message Queues.
4.  **Offline:** Cursor-based synchronization on reconnect.

**This concludes the Chat Application Playbook.**