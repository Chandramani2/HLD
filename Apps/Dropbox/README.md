# System Design: Dropbox/GDrive (Cloud Storage & Sync)

## 1. Requirements & Scope

### Functional Requirements
* **File Upload/Download:** Support for basic file operations.
* **Automatic Sync:** Seamless file synchronization across multiple devices.
* **Sync Strategy:** Support for both local changes (uploading to remote) and remote changes (pulling to local).

### Non-Functional Requirements (NFRs)
* **Availability over Consistency:** Ensure the system is always available for users to access their files.
* **Low Latency:** Optimized for high-speed uploads and downloads.
* **Large File Support:** Ability to handle files up to 50GB using chunking.
* **Reliability:** Support for resumable uploads and high data integrity (sync accuracy).

---

## 2. Back-of-the-Envelope Calculations

### Chunking & Upload Time
* **Large File Size:** 50GB.
* **Transfer Speed:** At 100Mbps, a 50GB file takes approximately 1 hour and 12 minutes to transfer.
* **Chunking Strategy:** Files are split into **5MB chunks**.
* **Fingerprinting:** Each chunk is identified by a hash of its bytes (fingerprint) to identify duplicate or changed segments.

---

## 3. High-Level Design (HLD)


### Components
* **Client App:** Features a **Local DB** to identify changes and a **Local Folder** that uses OS-level APIs (Windows: FileSystemWatcher; macOS: FSEvents) to monitor file movements.
* **LB & API Gateway:** Manages authentication, rate limiting, SSL termination, and request routing.
* **File Service:** Handles file metadata requests and provides **Pre-signed URLs** to allow clients to upload directly to S3.
* **Sync Service:** Manages the logic for polling for changes and reconciling local/remote versions.
* **Blob Storage (S3):** Stores raw file bytes using multi-part uploads.
* **Event Bus (Kafka):** Used for sync notifications, partitioned by `userId` and `folderId`.

---

<img width="2200" height="1007" alt="Image" src="https://github.com/user-attachments/assets/f1ddeb84-cd8f-46c8-9d82-7032da422e81" />

## 4. Database Schema (DynamoDB)

The system uses **DynamoDB** to store file metadata for high scalability.

| Entity | Fields |
| :--- | :--- |
| **File Metadata** | `fileId`, `folderId`, `fileName`, `MimeType`, `size`, `ownerId`, `s3Link`, `createdAt`, `updatedAt` |
| **Chunks** | List of: `{ id (fingerprint), status, s3Link, updatedAt(sync) }` |
| **Folder** | `cursor` (used for sync tracking) |

---

## 5. Sync Logic & Optimization
The system employs **Delta Sync** to ensure speed and consistency.

* **Fast Sync:** Achieved through adaptive polling and fetching only the specific changed "chunks" (Delta Sync).
* **Consistent Sync:** Uses an event bus with cursors and reconciliation logic to ensure the database and client remain in sync.
* **Optimized Upload:** Clients request a pre-signed URL from the File Service to upload directly to S3, bypassing the application server to reduce load.

---

## 6. System Design Interview Q&A

1. **Q: Why split files into 5MB chunks?**
    * **A:** It facilitates resumable uploads (if a connection drops, you only restart the failed chunk) and enables **Delta Sync**, where only modified parts of a file are re-uploaded.

2. **Q: How does the system detect changes on a user's computer?**
    * **A:** By utilizing native OS APIs: `FileSystemWatcher` for Windows and `FSEvents` for macOS to listen for file system events.

3. **Q: What is the purpose of the Pre-signed URL?**
    * **A:** It allows the client to upload file bytes directly to S3, which is more efficient than routing large files (up to 50GB) through the File Service API.

4. **Q: How is the Event Bus (Kafka) partitioned?**
    * **A:** It is partitioned by `userId` and `folderId` to ensure that sync events for a specific user/folder are processed in the correct order.

5. **Q: How does "Adaptive Polling" work?**
    * **A:** If no changes are detected, the polling frequency decreases; when frequent changes occur, the frequency increases to keep the sync "live".

6. **Q: Why use DynamoDB for metadata?**
    * **A:** Metadata needs to be highly available and scale horizontally. DynamoDB's key-value structure is ideal for looking up file attributes and chunk lists by `fileId`.

7. **Q: How does the system handle concurrent edits to the same file?**
    * **A:** The system uses a **Sync Cursor** and reconciliation logic. If a conflict occurs, it can use versioning or create a "conflicted copy" of the file.

8. **Q: What happens when a file is deleted?**
    * **A:** The File Service marks the metadata as deleted. A background worker eventually cleans up the orphaned chunks in S3 (Soft Delete vs. Hard Delete).

9. **Q: How is high data integrity maintained?**
    * **A:** Through **Fingerprinting**. Hashing each chunk ensures that the bytes received by the server exactly match the bytes sent by the client.

10. **Q: How does the system handle offline changes?**
    * **A:** The Local DB tracks changes made while offline. Once a connection is restored, the Sync Service pulls remote changes and pushes local ones, reconciling any overlaps.