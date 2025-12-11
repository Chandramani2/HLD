# Elasticsearch Architecture & Internals (Senior Developer Edition)

## 1. How It Works: The Deep Dive
At its core, Elasticsearch is a distributed document store built on top of **Apache Lucene**.

### The Physical Layer
* **Indices are Logical:** An index is just a namespace/grouping.
* **Shards are Physical:** The actual data lives in **Shards**. Every shard is a self-contained **Lucene Instance** (a fully functional search engine).
* **Immutability:** Once a document is written to a Lucene segment, it is **immutable**. You cannot change a file; you can only write a new one and mark the old one as "deleted" (similar to Log-Structured Merge trees).

### The Write Path (Data Durability Flow)
*This is critical for questions regarding data loss and NRT (Near Real-Time).*

1.  **Memory Buffer:** A write request hits the Primary Shard. The doc is written to an in-memory **Indexing Buffer**.
2.  **Translog (Transaction Log):** Simultaneously, it is appended to the `Translog` on disk. This ensures data is not lost if the node crashes before the segment is written to disk.
3.  **Refresh (Make Searchable):** By default every `1s`, the buffer is "refreshed." This creates a new **Lucene Segment**.
    * *State:* The document is now **searchable**.
    * *Note:* It is *not* yet fully committed to the main disk storage.
4.  **Flush (Make Durable):** Periodically (or when the Translog gets full), a "Flush" occurs. Segments are `fsync`'ed to the physical disk, and the Translog is cleared.

> The Write Path (Indexing)
1.  **Coordination:** Request hits a node; node routes it to the correct **Primary Shard**.
2.  **Memory Buffer:** Document written to memory buffer and **Translog**.
3.  **Refresh:** Every 1s, buffer is written to a new **Segment** (searchable but not durable).
4.  **Flush:** Translog is committed, and segments are `fsync`ed to disk.

### The Read Path (Scatter-Gather)
1.  **Scatter:** The coordinating node identifies which shards (primary or replica) hold the data.
2.  **Gather:** Each shard executes the query locally and returns **only Document IDs and Scores** (lightweight) to the coordinating node.
3.  **Fetch:** The coordinating node sorts the global results, requests the actual `_source` (JSON content) from the specific shards, and returns the final response.

---

## 2. Key Data Structures
*Senior devs must know that ES uses different structures depending on the data type.*

| Data Structure | Used For | Technical Details |
| :--- | :--- | :--- |
| **Inverted Index** | Text Search | Maps terms to Doc IDs (e.g., "Kafka" $\to$ `[Doc1, Doc5]`). Best for `match` queries. |
| **BKD Trees** | Numerics, Dates, Geo | Block KD-Trees. Optimized for multi-dimensional range queries. Faster than inverted indices for numbers. |
| **Doc Values** | Sorting, Aggregations | **Columnar Storage** on disk. Stores `DocID -> Value`. Inverted index is slow for sorting (requires un-inverting); Doc Values allow fast columnar access (OS Cache friendly). |
| **Fielddata** | Aggs on `Text` fields | **Warning:** Loads the inverted index into **Heap Memory**. Major cause of OOM. Always use `keyword` type (Doc Values) for sorting strings. |

---

## 3. Scaling Strategies

### A. Scaling for Write-Heavy Workloads (Log Ingestion)
* **Refresh Interval:** Increase `index.refresh_interval` from `1s` to `30s` or `60s`. This reduces the overhead of creating tiny Lucene segments.
* **Bulk API:** Never index single documents. Use Bulk requests (target 5MB–15MB payload size).
* **Auto-Generated IDs:** Let ES generate the `_id`. If you provide an ID, ES must check if it exists (Read-before-Write cost). If ES generates it, it blindly appends.
* **Async Translog:** Set `index.translog.durability: async`. You risk losing the last few milliseconds of data on a crash, but write throughput increases significantly.

### B. Scaling for Read-Heavy Workloads (Search)
* **Replica Scaling:** Increase the number of **Replicas**. Reads can be parallelized across replicas.
* **Force Merge:** For time-series data (e.g., last month's logs) that is no longer being written to, run `_forcemerge` to reduce segments to 1. This speeds up lookups drastically.
* **Circuit Breakers:** Configure strict memory circuit breakers to prevent complex queries from crashing the node.

---

## 4. Senior Level Interview Q&A

### Q1: We are facing "Split Brain" issues. How do we fix it?
> **Context:** Network partition causes two nodes to both believe they are the Master.

**Senior Answer:**
"In older versions (<7.x), we had to manually set `discovery.zen.minimum_master_nodes` to `(N/2) + 1`. In modern Elasticsearch (7.x+), this is handled automatically via **Voting-Only nodes** and a formal **Quorum** based consensus. If it happens today, it is likely a misconfiguration in `cluster.initial_master_nodes` during bootstrap or severe network latency exceeding the discovery timeout."

### Q2: How do you handle "Deep Pagination" (e.g., Page 5,000)?
> **Context:** User wants to jump to result #50,000.

**Senior Answer:**
"Standard pagination (`from` + `size`) is O(N). To get result 50,000, the shard must fetch 50,000 docs, sort them in memory, and discard 49,999.
1.  **For Scrolling:** Use **`search_after`**. It uses the sort value of the *last* result as a cursor for the *next* page. It is stateless and efficient.
2.  **For Exports:** Use **Point-in-Time (PIT)** snapshots to ensure consistency during long exports."

### Q3: What causes a `CircuitBreakingException`?
**Senior Answer:**
"This protects the JVM Heap. If a query attempts to load too much data into memory (e.g., aggregating on a high-cardinality `text` field using Fielddata), ES calculates the estimated RAM usage *before* execution. If it exceeds the limit (default 60% of Heap), it breaks the circuit to prevent an OutOfMemory (OOM) crash. The fix is usually data modeling (using `keyword` instead of `text`) or optimizing the query scope."

### Q4: A node is running out of disk space. What happens?
**Senior Answer:**
"Elasticsearch relies on **Watermarks**:
1.  **Low (85%):** Stops allocating *new* shards to the node.
2.  **High (90%):** Attempts to migrate *existing* shards off the node.
3.  **Flood Stage (95%):** Enforces a **Read-Only block** (`index.blocks.read_only_allow_delete`) on all indices on that node. Even if you free space, you often have to manually remove this block setting via API."

### Q5: Why is search slow even though we added more nodes?
**Senior Answer:**
"You might be suffering from **Oversharding**. If you have an index with 50 shards but only 10GB of data, every search request forces the coordinating node to manage 50 threads of Scatter/Gather overhead.
* **Rule of Thumb:** Keep shard size between **20GB – 40GB**.
* **Fix:** Use the **Shrink API** to reduce the shard count to an appropriate number."

# Elasticsearch Internals: Data Structures & Scaling (Deep Dive)

## Part 1: Data Structures Under the Hood
Elasticsearch does not use a single data structure. It uses specialized structures for different data types to optimize for storage, search speed, and aggregation speed.

### 1. The Inverted Index (For Text Search)
The Inverted Index is the heart of full-text search. It maps **Terms** to **Documents**. However, physically, it is more complex than a simple Hash Map.

#### Components of the Inverted Index:
1.  **Term Index (The Roadmap):**
    * **Problem:** The Term Dictionary (list of all words) is too big to fit in RAM.
    * **Solution:** ES uses an **FST (Finite State Transducer)**. An FST is like a generic Trie but much more compressed. It maps "prefixes" of words to their location on disk.
    * **Location:** Always loaded into **Heap Memory**.
    * **Function:** It tells ES, "The word 'Apple' starts at offset `0x1A2B` in the file on disk."

2.  **Term Dictionary (The Words):**
    * **Structure:** A sorted list of all unique terms in the dataset.
    * **Location:** Stored on **Disk** (but relies heavily on OS File System Cache).

3.  **Postings List (The Data):**
    * **Structure:** A list of Document IDs containing the term.
    * **Optimization:** It doesn't store raw IDs (e.g., `1, 5, 8, 20`). It uses **Frame of Reference (FOR)** compression (Delta encoding). It stores the *difference* between IDs (e.g., `1, +4, +3, +12`) to save space.
    * **Intersection:** For queries like `match: "fast" AND "car"`, Lucene uses **Roaring Bitmaps** or Skip Lists to quickly find the intersection of two Postings Lists.

Used for `text` and `keyword` fields.
* **Concept:** Maps terms to the documents containing them (like a book index).
* **Structure:**
    1.  **Term Dictionary:** Sorted list of all terms.
    2.  **Postings List:** List of Doc IDs containing the term.
    3.  **FST (Finite State Transducer):** An in-memory graph structure that maps term prefixes to their location on disk, reducing memory usage significantly compared to a standard Map.

---

### 2. BKD Trees (For Numerics, Dates, Geo)
*Why not use Inverted Index for numbers?* Because an Inverted Index treats numbers as strings. Searching for `price > 50` requires looking up `51`, `52`, `53`... which is slow.

* **Structure:** **Block KD-Trees (BKD)**.
* **How it works:** It divides the data space into multi-dimensional blocks (like binary space partitioning).
* **Performance:** It allows Elasticsearch to discard huge chunks of data instantly. If you search for `price > 100`, the BKD tree knows exactly which blocks on disk contain values over 100 and ignores the rest.
* **Usage:** Used for `long`, `integer`, `double`, `date`, `geo_point`.

* **Concept:** A Block K-Dimensional tree. It adapts the K-D tree logic (splitting space into halves) but optimizes it for disk blocks.
* **Usage:** Extremely efficient for Range Queries (e.g., `age > 20`) and Geo-spatial queries (e.g., "within 5km").
---

## 3. Architecture & Components

| Component | Description |
| :--- | :--- |
| **Node** | A single server instance. |
| **Shard** | A single Lucene instance. An index is split into primary shards. |
| **Replica** | A copy of a primary shard for high availability and read scaling. |
| **Segment** | Immutable files on disk that make up a shard. |
| **Translog** | (Transaction Log) An append-only file ensuring data durability before fsync. |

---

### 4. Doc Values (For Sorting & Aggregations)
The Inverted Index is great for finding *Docs* from *Terms*. It is terrible at finding *Values* from *Docs* (which is needed for Sorting/Aggregations).

* **The Problem:** To sort by Price, ES would have to "un-invert" the index (scan every term to see if Doc #1 is in it). This is memory-intensive.
* **The Solution:** **Doc Values**. This is a **Columnar Store** (similar to Apache Parquet or Cassandra).
* **Structure:** A fast lookup map of `DocID -> Value`.
* **Location:** Stored on disk, mapped into memory via the OS Cache.
* **Benefit:** Allows sorting and aggregating on billions of rows with minimal Heap memory usage.

---

## 5. Configuration (`elasticsearch.yml`)
* `cluster.name`: Logical name of the cluster.
* `node.roles`: Define if a node is `master`, `data`, or `ingest`.
* `bootstrap.memory_lock`: **Crucial.** Set to `true` to disable swapping.
* `discovery.seed_hosts`: Initial list of nodes for cluster formation.
* `indices.memory.index_buffer_size`: % of heap used for buffering writes (default 10%).

---

## Part 2: Advanced Scaling Strategies

Scaling isn't just "adding nodes." It requires managing three resources: **Heap (RAM)**, **CPU**, and **Disk I/O**.

### 1. Scaling Ingestion (Write Throughput)
When your write queue is filling up, or CPU is 100% during ingestion:

* **A. Optimize Segment Merging (The Silent Killer):**
    * *Concept:* Every second (refresh), a new segment is created. Background threads constantly merge small segments into big ones. This consumes massive I/O and CPU.
    * *Strategy:* If doing a bulk load, **disable refreshes** (`refresh_interval: -1`). Enable it only when the load is done. This prevents the "thundering herd" of merge threads.
* **B. Translog Tuning:**
    * *Concept:* Every write goes to disk (Translog) for safety.
    * *Strategy:* Set `index.translog.durability: async`. ES will buffer writes in memory and only touch the disk every 5 seconds. You risk 5s of data loss, but gain 20-30% indexing speed.
* **C. ID Generation:**
    * *Strategy:* **Never use custom IDs** (like DB Primary Keys) if possible. If you send `id: "user_123"`, Lucene must check if "user_123" exists (Disk Read) before writing (Disk Write). If you let ES generate the ID, it skips the check (Write Only).

### 2. Scaling Search (Read Latency)
When queries are slow or timeout:

* **A. Custom Routing (The "Shard Reduction" Trick):**
    * *Concept:* By default, ES hashes the ID to decide which shard gets the data. To find it, you must query *all* shards.
    * *Strategy:* Use **`_routing`**. If your data is multi-tenant (e.g., `company_id`), save data using `routing=company_id`. When searching, provide the same routing key. ES will query **only 1 shard** instead of 50.
* **B. Node Query Cache vs. Request Cache:**
    * **Node Query Cache:** Caches the *results of filter contexts* (e.g., bitsets for `status: "active"`). Very efficient.
    * **Shard Request Cache:** Caches the *full JSON result* of aggregations. Use this if you have a dashboard where users keep hitting the same "Sales Report" aggregation.
* **C. Force Merge (Cold Data):**
    * *Strategy:* For daily log indices (e.g., `logs-2023-10-01`), once the day is over, run a `_forcemerge` to reduce segments to **1**. This eliminates the overhead of checking multiple segments during a search.

### 3. Architecture Scaling (ILM - Index Lifecycle Management)
To scale to Petabytes, you cannot keep all data on expensive NVMe SSDs.

* **Hot Nodes:** High CPU, NVMe SSDs. Handles **Ingestion** and **Recent Search**.
* **Warm Nodes:** High Disk density, HDD/SATA. Handles **Read-Only** data (last 7-30 days).
* **Cold/Frozen Nodes:** S3/Blob Store backed (Searchable Snapshots). Massive storage, very cheap, slower search.

---

## Part 3: Senior Interview Q&A (Deep & Scenario Based)

### Q1: "We added 10 new nodes, but indexing speed didn't increase. Why?"
**Answer:**
"This is likely due to **Primary Shard Saturation**.
* Write throughput is limited by the *number of Primary Shards*. Replicas only help with *Read* scaling.
* If an index has 5 Primary Shards, and you have 20 Data Nodes, only 5 nodes are doing the heavy lifting of ingestion at any specific time. The other 15 are just waiting to be Replicas.
* **Fix:** You must increase the number of Primary Shards (requires reindexing) or use the Rollover API to create new indices with more shards."

### Q2: "What is a 'Mapping Explosion' and how do we prevent it?"
**Answer:**
"A Mapping Explosion occurs when you ingest JSONs with random keys (e.g., dynamic logging). Each new key updates the **Cluster State**. The Cluster State is propagated to *all* nodes.
* If you have 10,000 unique fields, the Cluster State becomes huge (MBs). Every update blocks the master node, causing the whole cluster to hang.
* **Fix:** set `dynamic: strict` or `dynamic: false` in the mapping. Use **Flattened** data type for objects where you don't need deep search inside specific keys."

### Q3: "How does Garbage Collection (GC) affect Elasticsearch?"
**Answer:**
"Elasticsearch is a Java application. If Heap is under pressure (e.g., complex Aggregations or huge Fielddata loading), the JVM triggers **Stop-The-World** GC pauses.
* If the pause is long (e.g., > 30 seconds), the Master node thinks the Data node is dead.
* The Master kicks the node out of the cluster.
* This triggers **Shard Rebalancing** (moving TBs of data), which causes an I/O storm, leading to *more* GC pauses on other nodes. It's a cascading failure loop.
* **Prevention:** Monitor JVM Heap usage. Never allocate more than 30GB to Heap (due to Compressed Oops pointers). Rely on OS Cache for data storage, not Heap."

# Senior Backend Developer Interview Q&A: Elasticsearch

### Internals & Data Structures
**Q1: What is the difference between an Inverted Index and a Doc Value?**
**A:**
* **Inverted Index:** Maps Terms → Document IDs. Optimized for **Search** (finding docs).
* **Doc Values:** Maps Document IDs → Terms. Columnar storage optimized for **Sorting, Aggregations, and Scripting**.

**Q2: Why are segments immutable, and how does this affect updates?**
**A:** Immutability allows for lock-free reads and efficient caching. However, it means "updates" are actually "delete + re-index." The old document is marked in a `.del` file, and the new one is written to a new segment. The old data is only physically removed during **Segment Merging**.

**Q3: Explain the "Split Brain" problem and how to prevent it.**
**A:** Split Brain occurs when a cluster divides into two, and both sub-clusters elect a master, leading to data divergence. In modern ES (7.x+), this is handled automatically via `cluster.initial_master_nodes` and a quorum-based voting system. In older versions, `discovery.zen.minimum_master_nodes` (N/2 + 1) was required.

**Q4: What is the role of the Translog?**
**A:** Lucene commits are expensive. The Translog (Write Ahead Log) ensures durability. If a node crashes before the in-memory buffer is flushed to disk (fsynced), the Translog is replayed upon restart to recover the data.

**Q5: How does the BKD Tree improve range query performance over Inverted Indexes?**
**A:** Inverted Indexes are discrete (term-based). For a range `1 to 100`, an inverted index might have to look up 100 separate terms. A BKD tree acts like a spatial index, allowing ES to quickly drill down and select a "block" of numbers, making it significantly faster for numeric ranges.

### Performance & Scaling
**Q6: Why is Deep Paging (e.g., page 10,000) slow in Elasticsearch?**
**A:** To fetch results 10,000 to 10,010, each shard must fetch its top 10,010 results. The coordinator node must then collect `number_of_shards * 10,010` results, sort them in memory, and discard the first 10,000. This kills heap memory. **Solution:** Use `search_after` or Scroll API.

**Q7: What is the difference between `keyword` and `text` data types?**
**A:**
* **`text`:** Analyzed (tokenized, lowercased). Good for full-text search. Cannot be used for sorting/aggregations (unless `fielddata` is enabled, which is expensive).
* **`keyword`:** Stored exactly as is. Good for exact filtering, sorting, and aggregations.

**Q8: How do you handle "Hot/Warm" architecture?**
**A:** Use **Index Lifecycle Management (ILM)**.
* **Hot:** High-spec SSD nodes for ingestion and recent search.
* **Warm:** Cheaper HDD nodes for older logs.
* You use "Node Attributes" (e.g., `node.attr.box_type: hot`) and configure the index to move between them as it ages.

**Q9: What happens when you change the Mapping of an existing field?**
**A:** You **cannot** change the mapping of an existing field (e.g., changing `integer` to `text`) because the Inverted Index is already built. You must create a new index with the correct mapping and **Reindex** the data.

**Q10: Why would you set `refresh_interval` to `-1` or `30s` during bulk indexing?**
**A:** The default is `1s`. Creating a new segment every second is expensive. Increasing this interval reduces I/O pressure and segment count, significantly speeding up bulk ingestion.

### Troubleshooting
**Q11: What does "Red" cluster status mean vs "Yellow"?**
**A:**
* **Yellow:** All Primary shards are active, but some Replicas are unassigned (data is available, but not redundant).
* **Red:** At least one Primary shard is missing. Data is effectively lost or unavailable.

**Q12: How does a "Term Query" differ from a "Match Query"?**
**A:**
* **Term:** Exact match. Does **not** analyze the input (looks for exact token in inverted index).
* **Match:** Analyzes the input (tokenizes it) before searching. Used for full-text search.

**Q13: What is the `_source` field?**
**A:** It stores the original JSON body sent by the client. It is not indexed but is stored to be returned in search results. Disabling it saves disk space but prevents reindexing or retrieving the original doc.

**Q14: How does the "Coordinator Node" impact memory?**
**A:** It performs the "Scatter-Gather." If a client requests a huge aggregation or large response size, the Coordinator node must hold all that data in heap before returning it. It is often the OOM bottleneck.

**Q15: What is the "Circuit Breaker" in Elasticsearch?**
**A:** It is a mechanism to prevent OutOfMemory errors. If a query (e.g., a massive aggregation) is estimated to use more RAM than the limit (e.g., 70% of Heap), the Circuit Breaker trips and aborts the query immediately to save the node.

### Advanced
**Q16: How does Fuzzy Search work internally?**
**A:** It uses Levenshtein Distance (Edit Distance). It builds a "Levenshtein Automaton" and intersects it with the Term Dictionary (FST) to efficiently find terms within $N$ edits of the search term.

**Q17: Explain `routing` in Elasticsearch.**
**A:** By default, ES distributes docs based on `hash(id)`. If you provide a custom `routing` value (e.g., `user_id`), all docs for that user go to the same shard. This makes searching for that user extremely fast (you only query 1 shard instead of all), but can lead to "Data Skew" (Hot Shards).

**Q18: What is the difference between a Filter and a Query context?**
**A:**
* **Query:** "How well does this match?" Calculates a **Relevance Score**.
* **Filter:** "Does this match?" (Yes/No). **No scoring**. Results are **Cached**, making filters faster.

**Q19: What are Nested Objects vs. Parent-Child?**
**A:**
* **Nested:** Stores objects as separate hidden documents in the same Lucene block. Fast query, but reindexing the parent requires reindexing all children.
* **Join (Parent-Child):** Documents are stored separately on the same shard. Slower query (join overhead), but children can be updated independently of the parent.

**Q20: How does NRT (Near Real-Time) search work?**
**A:** It relies on the filesystem cache. When a segment is written (refreshed), it is opened by a file handle. Even though it hasn't been `fsync`ed to the physical disk platter, the OS cache allows it to be read. This bridges the gap between the memory buffer and the physical disk commit.