# System Design: Google Docs (Real-time Collaborative Editor)

## 1. Requirements Analysis
Before designing, we must clarify the scope to ensure we focus on the complex parts of the system.

### Functional Requirements
* **Document Operations:** Create, Read, Update, Delete (CRUD).
* **Real-time Collaboration:** Multiple users editing the same document simultaneously.
* **Conflict Resolution:** Merge concurrent edits without data loss.
* **Offline Support:** Allow editing without an internet connection and sync later.
* **History & Version Control:** View and revert to previous states.

### Non-Functional Requirements
* **Low Latency:** Keystrokes must appear to other users in <200ms.
* **Consistency:** All users must eventually see the same document state (Eventual Consistency).
* **High Availability:** The system should prioritize availability (AP in CAP theorem), utilizing local storage for disconnected operations.
* **Scalability:** Support millions of concurrent active users.

---

## 2. High-Level Architecture

The system is read-heavy generally, but **write-intense** during active collaboration sessions.

### Architecture Components

1.  **Client (Browser/Mobile):**
    * **Rendering:** Uses HTML Canvas or DOM manipulation.
    * **Local Store:** Maintains a local copy of the document model.
    * **OT Engine (Client):** Handles local conflict resolution and acts as a buffer for offline edits.

2.  **Load Balancer (LB):**
    * Distributes traffic across servers.
    * **Critical Requirement:** Must support **Sticky Sessions (Session Affinity)**. Since real-time collaboration relies on memory state, User A and User B editing "Doc #1" should ideally connect to the same server node.

3.  **API Gateway:**
    * Authentication, Rate Limiting, and SSL Termination.
    * Routes:
        * `/api/docs` -> Document Service (Rest API)
        * `/ws/collab` -> Collaboration Service (WebSocket)

4.  **Collaboration Service (The Core):**
    * **Stateful Service:** Maintains the "in-memory" state of active documents.
    * **WebSockets:** Uses bi-directional persistent connections for low-latency updates.
    * **Broadcasting:** Pushes changes to all connected users for a specific Document ID.

5.  **Document Service:**
    * Stateless service handling metadata (titles, folder structure, permissions).

6.  **Message Queue (Kafka/RabbitMQ):**
    * Decouples the high-speed real-time editing from the slower persistent database storage.

7.  **Data Storage:**
    * **Hot Storage (Redis):** Stores active document sessions, user presence, and recent operations.
    * **Cold Storage (NoSQL - Cassandra/DynamoDB):** Stores the **Operation Logs** (every edit is an immutable event).
    * **Blob Storage (S3):** Stores "Snapshots" of documents (e.g., created every 100 edits) to speed up loading times.

---

## 3. Key System Design Concepts

### A. Operational Transformation (OT)
This is the core algorithm ensuring consistency. We cannot use "Last Write Wins" because it destroys concurrent data in text editing.

* **The Problem:** User A inserts "X" at index 0. User B deletes the character at index 5. If B's delete arrives *after* A's insert, the intended character is now at index 6. Deleting index 5 removes the wrong character.
* **The Solution:** The server "transforms" the incoming operation based on operations that have already happened.
    * *Formula:* `Transform(New_Op, Previous_Op) -> Transformed_Op`
* **Comparison:** Unlike **CRDTs (Conflict-free Replicated Data Types)** which are peer-to-peer friendly but memory-heavy, OT is centralized and standard for browser-based editors like Google Docs.

### B. Communication Protocol
* **WebSockets (WSS):** Preferred over HTTP Long Polling to minimize header overhead and latency.
* **Optimization:** Batching. Do not send a request for every single character typed. Batch edits into small bundles (e.g., every 500ms or 5 characters) to reduce network load.

### C. Storage Pattern: Log-Based (Event Sourcing)
Instead of updating a single text field in a database row repeatedly:
1.  **Log It:** Store the mutation: `{ "op": "insert", "char": "a", "pos": 10, "timestamp": 12:00 }`.
2.  **Reconstruct It:** Replay operations to build the document.
3.  **Snapshot It:** Every $N$ operations, save the full document state to S3. When loading, fetch `Snapshot + Remaining Logs`.

---

## 4. Senior Developer Interview Q&A

**Q1: How do you handle a user who goes offline, makes 50 edits, and then comes back online?**
> **Answer:** This relies on the Client-side OT engine.
> 1.  **Buffer:** The client stores 50 edits in a "pending" buffer.
> 2.  **Version Check:** The client knows it was on Server Version v100.
> 3.  **Reconciliation:** On reconnection, it sends the edits. The server notes the current version is v120 (others edited meanwhile).
> 4.  **Transformation:** The server transforms the 50 pending edits against the 20 new server edits.
> 5.  **Ack:** The server commits the transformed edits and pushes the new state back to the client.

**Q2: Your Collaboration Service uses WebSockets. How do you scale this if 10 million users are online? WebSockets are stateful.**
> **Answer:** Scaling stateful layers requires specific strategies:
> * **Sticky Sessions:** Ensure users editing the same doc hit the same server.
> * **Pub/Sub (Redis):** If User A is on Server 1 and User B is on Server 2 (viewing the same doc), the servers communicate via a Redis Pub/Sub channel. Server 1 publishes an update to `channel_docID`, and Server 2 subscribes to it to update User B.

**Q3: How do you implement "Undo" functionality in a multi-user environment?**
> **Answer:** You cannot simply rollback to a previous state timestamp because that might undo *other users'* work.
> * **Inverse Operations:** Calculate the mathematical inverse of the specific user's last operation. If they added "a" at index 5, the undo operation is `delete at index 5`.
> * **Treatment:** This "undo" is treated as a brand new operation that goes through the standard OT pipeline.

**Q4: How would you store images inserted into the document?**
> **Answer:** Never store binary data in the WebSocket stream or the Operation Log database.
> 1.  Upload the image immediately to **Blob Storage (S3)** via a separate HTTP API.
> 2.  Receive a URL reference (CDN link).
> 3.  Send an operation to the Collaboration Server: `insert_image(url: "https://cdn...", pos: 50)`.
> 4.  The document model only contains the reference URL, keeping the payload light.

---

## 5. Summary Checklist
* **Primary Bottleneck:** Database writes (logs) and WebSocket connection limits.
* **Key Data Structure:** Rope (or Gap Buffer) for efficient text insertion/deletion.
* **Consistency Model:** Strong Eventual Consistency.
* **Algorithms:** Operational Transformation (OT).