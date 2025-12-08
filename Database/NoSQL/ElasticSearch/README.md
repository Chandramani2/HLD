# 🔎 The Senior Elasticsearch Internals Playbook

> **Target Audience:** Search Engineers, Data Architects, & Backend Leads  
> **Goal:** Master **Data Modeling**, **Relevance Scoring (BM25)**, and **Cluster Consistency**.

In a senior interview, you don't just "index JSON." You design schemas for **denormalization**, tune **garbage collection** for the JVM, and optimize **Text Relevance** for users.

---

## 📖 Table of Contents
1. [Part 1: The Consensus Model (Zen Discovery)](#-part-1-the-consensus-model-zen-discovery)
2. [Part 2: Data Modeling (Handling Relationships)](#-part-2-data-modeling-handling-relationships)
3. [Part 3: The Write Path (Translog & Segments)](#-part-3-the-write-path-translog--segments)
4. [Part 4: Relevance & Scoring (BM25 & Query Context)](#-part-4-relevance--scoring-bm25--query-context)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🧠 Part 1: The Consensus Model (Zen Discovery)

How does the cluster agree on the state of the world?

### 1. Master vs. Data Nodes
* **Master Node:** Stores the **Cluster State** (Mappings, Settings, Shard locations).
    * *Traffic:* Lightweight.
    * *Senior Rule:* In large clusters, **dedicated master nodes** are mandatory. Don't let your data nodes (under heavy CPU load) be masters.
* **Data Node:** Stores the **Shards** (Data). Handles CRUD and Aggregations.
* **Coordinating Node:** The node that receives the user's HTTP request. It scatters the request to data nodes and gathers the results.

### 2. Split Brain (The Quorum)
What happens if the network splits and two nodes think they are the Master?
* **Pre-7.x:** You had to manually set `discovery.zen.minimum_master_nodes = N/2 + 1`.
* **7.x+:** Elasticsearch now uses a deterministic voting system.
* **Senior Config:** Always have **3** Master-eligible nodes. If you have 2, and one dies, the other cannot form a majority (1 out of 2 is 50%, not >50%), and the cluster goes into Read-Only mode to prevent data corruption.

---

## 🕸️ Part 2: Data Modeling (Handling Relationships)

Elasticsearch is "NoSQL," meaning **No Joins**. But real data has relationships. How do we model `Blog Post` -> `Comments`?

### 1. Object Type (The Default)
* **Structure:** Just nest the JSON.
    ```json
    { "title": "My Post", "comments": [ { "user": "Alice", "stars": 5 } ] }
    ```
* **The Flaw:** Lucene flattens this. `comments.user` and `comments.stars` are stored as parallel arrays. You lose the correlation. (You can search for "Alice" AND "1 Star", even if Alice actually gave 5 stars).

### 2. Nested Type (The Fix)
* **Mechanism:** Internally indexes each comment as a separate "hidden" document near the parent.
* **Pros:** Preserves object boundaries. Accurate queries.
* **Cons:** **Update Cost.** Adding 1 comment requires re-indexing the *entire* Parent + All Comments. Bad for high-write scenarios.

### 3. Join Field (Parent-Child)
* **Mechanism:** Documents are stored separately but live on the **same shard**. Links are resolved at query time.
* **Pros:** Fast writes. You can add a comment without touching the blog post.
* **Cons:** Slow queries (Join overhead).

> **💡 Senior Decision Matrix:**
> * **1-to-1 or Small 1-to-Many?** Use **Object**.
> * **Data needs strict querying but rarely changes?** Use **Nested**.
> * **Data updates frequently (e.g., Logs/Time-series)?** Use **Denormalization** (Duplicate data).
> * **Large 1-to-Many (e.g., Product -> Reviews)?** Use **Parent-Child**.

---

## 💾 Part 3: The Write Path (Translog & Segments)

Why is Elasticsearch called "Near Real-Time" (NRT)?

### 1. The Buffer & Refresh
1.  **Memory Buffer:** Document is written to RAM. (Not searchable yet).
2.  **Refresh (Default 1s):** Buffer is written to a **Lucene Segment** in OS Cache.
    * *Now it is searchable.*
    * This is why you wait ~1s to see your data.

### 2. The Translog (Durability)
If the server crashes before writing to disk, RAM is lost.
* **Translog:** Every write is simultaneously appended to a Transaction Log on disk.
* **Flush:** When the Translog gets big (512MB) or every 30 mins, a **Flush** triggers.
    * Segments are `fsync`'ed to physical disk.
    * Translog is cleared.



---

## 🎯 Part 4: Relevance & Scoring (BM25 & Query Context)

Why did Document A appear before Document B?

### 1. TF-IDF vs. BM25
* **TF-IDF:** Old algo. Score = `Term Frequency` * `Inverse Doc Frequency`.
    * *Problem:* If a document repeats the word "Computer" 1,000 times, it scores artificially high.
* **Okapi BM25 (The Default):**
    * **Saturation:** The score curve flattens. Mentioning "Computer" 5 times is better than 1, but 1,000 times is not much better than 5.
    * **Length Normalization:** Matches in short fields (Title) score higher than matches in long fields (Body).

### 2. Query Context vs. Filter Context
Every Bool query has two clauses. Know the difference.
* **"Must" (Query Context):** "How well does this match?"
    * Calculates Score. Not Cached. Slower.
* **"Filter" (Filter Context):** "Does this match? Yes/No."
    * **Score = 0.**
    * **Cached:** Bitsets are cached in RAM. **Extremely Fast.**
* **Senior Tip:** If you don't care about ranking (e.g., `status: "active"`), ALWAYS use `filter` clause.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Optimizing for Heavy Writes
**Interviewer:** *"We are indexing 100k events/sec. The cluster is stalling."*

* ✅ **Senior Answer:**
    1.  **Increase Refresh Interval:** Set `index.refresh_interval` to `30s` or `-1` (during bulk). Stops creating tiny segments.
    2.  **Use ID generation:** Let ES generate the ID (`POST /index/_doc`). If you provide a custom ID (`PUT /index/_doc/123`), ES must check if `123` exists (Read-before-Write), which kills performance.
    3.  **Translog Async:** Set `index.translog.durability` to `async`. (Risk: lose last 5s of data on crash, but massive speedup).

### Scenario B: The "Circuit Breaker" Exception
**Interviewer:** *"The cluster threw a `Data too large` exception and rejected the query."*

* **The Reason:** A user tried to aggregate on a high-cardinality field (e.g., `UUID`) loading millions of buckets into Java Heap Memory.
* ✅ **Senior Answer:**
    * **Field Data Cache:** The query tried to un-invert the index into RAM.
    * **Solution:** Enable `doc_values` (on-disk columnar store) for that field. Most modern ES versions do this by default.
    * **Defense:** Configure **Circuit Breakers** to limit query memory usage to 40% of Heap.

### Scenario C: Optimistic Concurrency Control
**Interviewer:** *"User A reads a doc. User B reads the same doc. Both update it. The last write overwrites the first. How to prevent this?"*

* ✅ **Senior Answer:** "**OCC with `_seq_no` and `_primary_term`.**"
    * When reading, get the `_seq_no` (Version 10).
    * When writing, send `IF_SEQ_NO=10`.
    * If User B already updated it (Version is now 11), ES rejects User A's write with a `409 Conflict`.
    * User A must re-read and apply changes again.

### Scenario D: Tuning Relevance (Function Score)
**Interviewer:** *"We want 'Recent' articles to show up first, but they still need to match the keyword 'Economy'."*

* ✅ **Senior Answer:** "**Function Score Query with Decay Functions.**"
    * Use a `bool` query to match "Economy".
    * Apply a `function_score`:
        * `gauss`: Field = `publish_date`.
        * `scale`: "7d".
    * This multiplies the relevance score by a factor that decays the older the document gets. Recent hits float to the top; old hits sink, but relevant old hits still appear.

---

### **Final Checklist**
1.  **Data Modeling:** Nested vs. Parent-Child trade-offs.
2.  **Scoring:** Filter Context (Caching) vs. Query Context (BM25).
3.  **Writes:** Refresh Interval & Translog.
4.  **Consensus:** Split Brain & Master Election.

**This concludes the Elasticsearch Internals Playbook.**