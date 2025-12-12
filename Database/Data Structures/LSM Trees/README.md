# LSM Trees (Log-Structured Merge-Trees) in NoSQL

**LSM Trees** are the backbone of write-heavy databases. While traditional B+ Trees (used in MySQL/PostgreSQL) are optimized for *reads*, LSM Trees are optimized for **massive write throughput**.

* **The Problem with B+ Trees:** Writing to a B+ Tree often requires random I/O (jumping around the disk) to update specific pages. This is slow.
* **The LSM Solution:** LSM Trees turn random writes into **sequential writes** (appending to a file), which is significantly faster on both HDDs and SSDs.

---

## 1. Core Components

An LSM Tree is not a single tree; it is a collection of trees/structures spread across memory and disk.

### 1. WAL (Write Ahead Log) - *On Disk*
* **Purpose:** Crash Recovery.
* **Function:** Before doing anything, the database appends the data to a simple log file on disk. If the power fails, the database replays this log to restore lost data.

### 2. MemTable - *In Memory*
* **Purpose:** Fast buffering and sorting.
* **Function:** This is an in-memory data structure (usually a Red-Black Tree or Skip List).
* **Behavior:** New writes go here first. Because it's in RAM, inserts are instant. The data is kept **sorted** by key.

### 3. SSTables (Sorted String Tables) - *On Disk*
* **Purpose:** Long-term immutable storage.
* **Function:** When the *MemTable* gets full (e.g., reaches 128MB), it is flushed to disk as an **SSTable**.
* **Properties:**
    * **Immutable:** Once written, an SSTable is never modified.
    * **Sorted:** Keys are ordered, allowing for efficient binary searches.

---

## 2. The Working Flow

### Step 1: The Write Path (Fast & Sequential)
When you run `INSERT INTO users (id, name) VALUES (101, "Alice")`:

1.  **Append to WAL:** The command is appended to the log file (Disk) for safety.
2.  **Write to MemTable:** The data `(101, "Alice")` is inserted into the in-memory Red-Black Tree.
3.  **Acknowledge:** The database tells the client "Success."
    * *Note:* No disk seek happened for the main data structure yet. This is why writes are lightning fast.

### Step 2: The Flush (MemTable → SSTable)
When the MemTable reaches a threshold (e.g., 64MB):
1.  The MemTable becomes immutable.
2.  A new MemTable is created for new writes.
3.  The old MemTable is flushed to disk as a new **SSTable (Level 0)**.
    * *Result:* You now have a file on disk where `101: Alice` is stored.

### Step 3: The Read Path (Merging)
When you run `SELECT name FROM users WHERE id=101`:

The system checks in a specific order (Newest to Oldest):
1.  **Check MemTable:** Is `101` currently in memory? (Yes -> Return "Alice").
2.  **Check SSTables (L0):** If not in RAM, check the most recent SSTable file on disk.
3.  **Check SSTables (L1, L2...):** Continue checking older files until found.

### Step 4: Compaction (Housekeeping)
Over time, you will have thousands of small SSTables on disk. This slows down reads (checking 1,000 files is slow).
* **The Process:** A background process wakes up, picks several small SSTables, merges them into one large, sorted SSTable, and deletes the old ones.
* **Handling Updates/Deletes:**
    * If you update "Alice" to "Bob", the new entry `(101, "Bob")` sits in a newer SSTable. The old `(101, "Alice")` still exists in an old SSTable but is ignored because the system finds the new one first.
    * During compaction, the old "Alice" entry is discarded (Garbage Collection).

---

## 3. Visualizing the Architecture

```mermaid
graph TD
    subgraph "Write Path"
        Client[Client Request] --> WAL[(Write Ahead Log)]
        Client --> MemTable{MemTable <br/> (RAM - Sorted)}
    end

    subgraph "Flush Process"
        MemTable -.->|Flush when full| L0[SSTable Level 0]
    end

    subgraph "Compaction Process"
        L0 --> Merger((Compaction))
        L1[SSTable Level 1] --> Merger
        L1_Old[SSTable Level 1] --> Merger
        Merger --> L2[SSTable Level 2 <br/> (Merged & Sorted)]
    end

    classDef memory fill:#f9f,stroke:#333,stroke-width:2px;
    classDef disk fill:#ccf,stroke:#333,stroke-width:2px;
    class MemTable memory;
    class WAL,L0,L1,L1_Old,L2 disk;
```

---

## 4. Senior Developer Q&A: Deep Dives

### Q1: How do we delete data if SSTables are immutable?
**Answer:** You cannot physically remove a row from an SSTable immediately.
Instead, you write a **Tombstone**.
* You insert a special marker: `(101, DELETED)`.
* During a read, the system sees the Tombstone first and returns "Not Found," ignoring the older real data.
* The actual data is only physically removed during **Compaction**, when the Tombstone and the original data meet and cancel each other out.

### Q2: Reading is slow because we have to check many files. How do we optimize this?
**Answer:** We use **Bloom Filters**.
* A Bloom Filter is a small, probabilistic memory structure that tells you if a key *might* exist in a file or *definitely does not*.
* Before checking an SSTable on disk, the database checks the Bloom Filter in RAM.
* If the Bloom Filter says "No," the system skips opening that file entirely. This saves expensive disk I/O.

### Q3: What is "Write Amplification" vs. "Read Amplification"?
**Answer:** These are the primary trade-offs in LSM tuning.
* **Read Amplification:** One logical read requires looking at multiple files (MemTable + L0 + L1...).
    * *Fix:* Aggressive compaction (keeping fewer files).
* **Write Amplification:** One logical write results in multiple physical writes (WAL + Flush + Multiple Compactions moving the same data around).
    * *Fix:* Less aggressive compaction (spares the disk, but hurts read speed).
* **Space Amplification:** Storing multiple versions of the same key (old "Alice" and new "Bob") takes up extra disk space until compaction runs.

### Q4: Leveled Compaction vs. Size-Tiered Compaction?
**Answer:** Different databases use different strategies.
* **Size-Tiered (Cassandra default):** Merges SSTables of similar size. Great for **Write throughput**, but reads can be slower as data is fragmented.
* **Leveled (RocksDB/LevelDB):** Divides SSTables into specific levels (L1 is 10x larger than L0). Guarantees strictly limited number of files to check. Great for **Read heavy** workloads, but burns more CPU/Disk on compaction.