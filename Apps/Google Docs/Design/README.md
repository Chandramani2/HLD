# System Design Interview Guide: Google Docs

## Phase 1: Clarify Requirements (The "What")
*Goal: Define the scope so you don't build the wrong thing.*

**Candidate:** "Before designing, I want to align on the scope. Are we focusing on the entire Google Drive suite or just the document editing experience?"

**Interviewer:** "Focus on the real-time editing of a text document."

### 1. Functional Requirements (MVP)
* **Document Management:** Users can create, open, and delete documents.
* **Real-time Collaboration:** Multiple users (e.g., up to 10) can edit simultaneously.
* **Conflict Resolution:** Edits must not overwrite each other.
* **History:** Ability to undo/redo or view version history.

### 2. Non-Functional Requirements (Quality Attributes)
* **Latency:** Typing must feel instant. Updates from others should appear within ~200ms.
* **Consistency:** Eventual consistency is okay, but all users must converge to the *same* state.
* **Concurrency:** The system needs to handle high write throughput per document.

---

## Phase 2: Estimations (The "Scale")
*Goal: Determine if we need sharding or special storage optimization.*

**Candidate:** "I'll do a quick back-of-the-envelope estimation to guide my storage choice."

* **Traffic:**
    * Assume **10 million** Daily Active Users (DAU).
    * On average, a user makes **5,000 characters** of edits per day.
    * Total Writes: $10^7 \text{ users} \times 5,000 \text{ chars} = 50 \text{ billion ops/day}$.
* **Storage:**
    * If each operation (timestamp, user, char, pos) takes approx 100 bytes.
    * Daily Storage: $50 \times 10^9 \times 100 \text{ bytes} \approx 5 \text{ TB/day}$.
    * **Conclusion:** This is write-heavy. A standard SQL DB might struggle with the log volume. We need a NoSQL store (like Cassandra/DynamoDB) for the edit logs and a cache (Redis) for active sessions.

---

## Phase 3: API Design
*Goal: Define how clients talk to the server.*

**Candidate:** "We need two protocols here: HTTP for static actions and WebSockets for real-time."

### 1. REST API (Document Management)
* `POST /api/documents` (Create new doc)
* `GET /api/documents/{docId}` (Get metadata like title, owner)

### 2. WebSocket API (Editing)
* **Connect:** `ws://api.googledocs.com/collab/{docId}`
* **Payload (Send Edit):**
    ```json
    {
      "type": "insert",
      "pos": 120,
      "char": "a",
      "version": 45,
      "userId": "u123"
    }
    ```

---

## Phase 4: High-Level Design (The "Blueprints")
*Goal: Draw the boxes and connections.*

**Candidate:** "I will divide the architecture into a 'Management Layer' (stateless) and a 'Real-time Layer' (stateful)."

1.  **Client:**
    * Has a local copy of the DOM/Text.
    * Has an **OT Engine** to handle conflicts locally before sending.
2.  **Load Balancer:**
    * Needs **Sticky Sessions** (Session Affinity). If User A and User B are editing Doc #1, they should ideally be routed to the same application server to minimize latency.
3.  **Real-Time Service (App Server):**
    * Maintains an open WebSocket connection.
    * Holds the "In-Memory" state of the document.
4.  **Message Queue (Kafka):**
    * Asynchronously saves edits to the database so the user doesn't have to wait for disk I/O.
5.  **Database (Persistence):**
    * **Metadata DB (PostgreSQL):** User info, document titles, permissions.
    * **Log DB (NoSQL - Cassandra):** Stores the immutable log of every edit.

---

## Phase 5: Deep Dive (The "Hard Parts")
*Goal: Demonstrate senior-level depth on the critical bottleneck.*

**Interviewer:** "How do you actually handle the conflict if two users type at the same time?"

**Candidate:** "This is the core of the problem. We use **Operational Transformation (OT)**."

### 1. The Concurrency Algorithm: OT
* **Scenario:**
    * Text: `"CAT"`
    * User A: Adds "S" at index 3 -> `"CATS"`
    * User B: Deletes "C" at index 0 -> `"AT"`
* **The Conflict:** If B's delete arrives at the server *after* A's insert, the index changes. B intended to delete index 0, but if A inserted something at index 0, B might delete the wrong letter.
* **The Server's Role:**
    * The server acts as the **source of truth**.
    * When an operation arrives, the server checks the version number.
    * If the version is old, the server **transforms** the incoming operation against all recent operations it has already processed.
    * *Mathematical Logic:* `T(Op_new, Op_old) = Op_transformed`

### 2. Data Storage Schema
Instead of saving the whole text file every time, we use **Event Sourcing**.

* **Table: Operations_Log**
    * `doc_id` (Partition Key)
    * `version_id` (Sort Key)
    * `operation` (blob: insert/delete)
    * `user_id`

* **Optimization (Snapshots):**
    * Replaying 50,000 logs to load a document is too slow.
    * Every 1000 edits, a background worker compiles the logs into a full file and saves it to **Object Storage (S3)**.
    * **Read Path:** Load latest Snapshot from S3 + Replay only the logs created *after* that snapshot.

### 3. Real-Time Propagation
* **Problem:** What if the document is so popular (e.g., a public doc) that it exceeds the capacity of one server?
* **Solution:** **Redis Pub/Sub**.
    * If User A is on Server 1 and User B is on Server 2.
    * Server 1 publishes the edit to a Redis channel `channel_doc_123`.
    * Server 2 subscribes to `channel_doc_123`, receives the payload, and pushes it down the WebSocket to User B.

---

## Phase 6: Final Review (Bottlenecks)
*Goal: Critique your own design.*

1.  **Reliability:** If a WebSocket server crashes, users lose connection. The client must implement **automatic retries** with exponential backoff.
2.  **Security:** Since WebSockets are long-lived, we need to periodically re-validate the auth token (e.g., every 15 minutes) without dropping the connection.
3.  **Data Integrity:** If the OT algorithm fails, the document becomes corrupted. We need **checksums** (hashes) of the document content sent periodically by clients to verify everyone is seeing the same text.