# System Design: Google Docs & Collaborative Editor

## 1. Requirements & Scale

### Functional Requirements
* **CRUD Operations:** Create, update, and delete documents.
* **Real-time Collaboration:** Multiple users editing the same document simultaneously.
* **Presence & Awareness:** View other users' cursor positions and presence.
* **Version Control:** Document versioning and history rollback.

### Non-Functional Requirements (NFRs)
* **High Availability:** System must stay up for millions of users.
* **Low Latency:** Real-time updates should reflect in **< 100ms**.
* **Consistency:** * **Offline:** Availability over Consistency.
    * **Online:** Consistency over Availability (Operational Transformation).
* **Scale:** Support for billions of documents.

---

<img width="2366" height="1462" alt="Image" src="https://github.com/user-attachments/assets/8aed0523-cf2b-41dc-be1c-9cad1374eeac" />

## 2. Database Schema (Polyglot Persistence)

The system uses different databases for metadata, document snapshots, and edit operations.

### Metadata DB (NoSQL - Cassandra/DynamoDB)
*Focus: Fast lookups and massive scalability for doc info.*
* **Document Table:** `docId (PK)`, `title`, `ownerId`, `s3Url (latest blob)`, `createdAt`, `lastModifiedTime`.
* **Version Table:** `versionId (PK)`, `docId`, `modifiedBy`, `timestamp`, `s3Url (snapshot)`.

### Operations DB (Relational/Time-series)
*Focus: Strict ordering of every single keystroke/edit.*
* **Operations Table:** `opId`, `docId`, `userId`, `timestamp`, `type (insert/delete)`, `data (char)`, `position`.

---

## 3. Redis Usage: The Canonical Copy
In a collaborative editor, Redis doesn't just cache data; it acts as the **source of truth for active sessions**.

* **The Canonical Copy:** When users are actively editing, the document's current state is kept in Redis to avoid hitting the main DB for every keystroke.
* **TTL Management:** If no user is editing (determined by a TTL), the document is flushed from Redis to the main database and S3.
* **Presence Data:** * **Key:** `presence:{docId}`
    * **Value:** Set of `{userId, cursor_pos, color_code}`.
* **Locking (if applicable):** While this design uses OT, Redis can manage distributed locks for heavy administrative actions (like deleting the doc).

---

## 4. Kafka & Operational Transformation (OT)
Kafka acts as the **sequencing engine** to ensure all edits are processed in the order they were received.

1. **Ordering:** Kafka partitions edits based on `docId`. This ensures all operations for a single document arrive at the **Operation Consumer Service** in strict chronological order.
2. **OT Processing:** The backend OT engine receives an operation (e.g., "Insert 'A' at pos 5"). It checks if other operations have shifted the document state since that user last synced. It **transforms** the index of the operation to ensure the final state is consistent across all clients.
3. **Replay & Reconciliation:** If a client goes offline and returns, the **Reconciliation Service** pulls the "missing" events from Kafka/DB and replays them to catch the client up to the canonical state.

---

## 5. Senior Software Engineer Interview Q&A (15-20 Questions)

### Architecture & Strategy
1. **Q: Why use WebSockets instead of HTTP Long Polling?**
    * **A:** Keystrokes are frequent and small. WebSockets provide a persistent, full-duplex connection that reduces the overhead of repeatedly opening HTTP headers.

2. **Q: What is the difference between OT (Operational Transformation) and CRDTs?**
    * **A:** OT requires a central server to transform operations for consistency. CRDTs (Conflict-free Replicated Data Types) allow clients to merge changes independently without a central server but are much more complex to implement and memory-intensive.

3. **Q: How do you handle a user editing while offline?**
    * **A:** The client maintains a local buffer of operations. Upon reconnecting, it sends these to the Reconciliation Service, which integrates them using OT logic or prompts the user if the conflict is too large.

4. **Q: Why use Cassandra for document metadata?**
    * **A:** Cassandra provides high write throughput and easy horizontal scaling, which is necessary when managing billions of document records.

5. **Q: How do you ensure "Cursor Presence" doesn't overwhelm the network?**
    * **A:** Cursor movements are throttled on the client side (e.g., sent every 50-100ms) and often broadcast via a lightweight pub/sub mechanism like Redis or dedicated WebSocket rooms.

### Concurrency & Consistency
6. **Q: If two users insert a character at the exact same position at the same time, how does OT resolve this?**
    * **A:** The OT server assigns a priority (usually based on timestamp or User ID). It transforms the second operation to be `index + 1` so both characters are preserved rather than one overwriting the other.

7. **Q: How do you prevent the Operations DB from growing infinitely?**
    * **A:** We use **Snapshoting**. Every 100 operations or every 10 seconds, the current state is saved to S3 as a blob, and old operations are archived or truncated.

8. **Q: What happens if the Operation Consumer Service (OT Engine) crashes?**
    * **A:** Since Kafka stores the message offset, a new instance can spin up, read from the last committed offset, and resume processing without data loss.

9. **: How do you handle "Large Document" performance?**
    * **A:** Use **Pagination** or **Lazy Loading** for the UI. In the backend, split the document into "Chunks" or "Nodes" so the OT engine doesn't have to re-evaluate the entire 500-page doc for one character change.

### Scalability & Infrastructure
10. **Q: How would you scale the WebSocket servers?**
    * **A:** Use a Load Balancer with **Sticky Sessions** (Session Affinity) to ensure a user stays connected to the server that has their doc's context, or use a distributed Pub/Sub (Redis) to sync messages between WebSocket nodes.

11. **Q: Why is Kafka partitioned by `docId`?**
    * **A:** To ensure that all edits for a specific document are processed by the same consumer instance, maintaining strict order.

12. **Q: How do you handle global latency for a user in London editing a doc hosted in NYC?**
    * **A:** Use **Edge Computing** or **Global Accelerators**. While the OT source of truth remains in one region to avoid split-brain, the WebSocket connection can be terminated at a closer edge location.

13. **Q: How do you implement "Version History"?**
    * **A:** By storing the "Snapshots" in S3 and keeping a pointer to the offset in the Operations DB. Reverting involves loading a snapshot and replaying ops up to a specific timestamp.

14. **Q: What is the role of the CDN in this design?**
    * **A:** The CDN serves the static Javascript/React application and the read-only versions of public documents to offload the main servers.

15. **Q: How do you protect against "Write Heavy" users or bots?**
    * **A:** Implement **Rate Limiting** at the API Gateway level based on `userId` and `docId`.

### Reliability
16. **Q: How do you ensure data isn't lost if Redis clears?**
    * **A:** Redis should be configured with AOF (Append Only File) persistence, and the system should perform regular "Manual Saves" or "Autosaves" to S3/Postgres every few seconds.

17. **Q: How do you handle document permissions/ACL?**
    * **A:** The Metadata Service checks permissions against a User-Doc-Access table before the API Gateway upgrades the connection to a WebSocket.

18. **Q: Why is "Operational Transformation" considered complex to implement?**
    * **A:** It requires handling $N^2$ transformation permutations (Insert vs Insert, Insert vs Delete, etc.) and ensuring mathematical convergence across all possible edge cases.