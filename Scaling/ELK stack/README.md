# 🦌 The Senior ELK Stack (Elastic) Playbook

> **Target Audience:** DevOps, SREs, & Data Engineers  
> **Goal:** Master **Log Aggregation**, **Full-Text Search Internals**, and **Cost-Efficient Storage Strategies**.

In a senior interview, you must treat Elasticsearch not just as a "Log bucket," but as a distributed **NoSQL Search Engine**. You need to understand **Inverted Indices**, **Sharding strategies**, and why "Mapping Explosions" kill clusters.

---

## 📖 Table of Contents
1. [Part 1: The Architecture (Beats vs. Logstash)](#-part-1-the-architecture-beats-vs-logstash)
2. [Part 2: Elasticsearch Internals (Lucene & Shards)](#-part-2-elasticsearch-internals-lucene--shards)
3. [Part 3: Scaling Strategies (Hot-Warm-Cold)](#-part-3-scaling-strategies-hot-warm-cold)
4. [Part 4: The "Mapping Explosion" (Schema Design)](#-part-4-the-mapping-explosion-schema-design)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: The Architecture (Beats vs. Logstash)

The modern stack is actually **BELK** (Beats, Elasticsearch, Logstash, Kibana).

### 1. The Data Flow
1.  **Beats (The Shipper):** Lightweight agents on the edge servers.
    * *Filebeat:* Tails log files.
    * *Metricbeat:* Grabs CPU/RAM stats.
2.  **Buffer (The Safety Net):** **Kafka / Redis**.
    * *Senior Requirement:* Never send from Beats directly to Logstash/ES in high-scale systems. If ES slows down, you lose logs. Use Kafka to handle **Backpressure**.
3.  **Logstash (The Processor):** Heavy Java-based ETL.
    * Parses, filters, anonymizes (PII removal), and GeoIP tagging.
4.  **Elasticsearch (The Storage):** Indexes the data.
5.  **Kibana (The View):** Visualizes the data.



### 2. Beats vs. Logstash
| Feature | Beats (Go) | Logstash (JRuby) |
| :--- | :--- | :--- |
| **Role** | Shipper (Edge) | ETL Processor (Central) |
| **Resource Usage** | Lightweight (MBs) | Heavy (GBs of Heap) |
| **Complexity** | Simple config | Complex Pipelines (Grok) |
| **Senior Rule** | Do simple parsing here if possible. | Only use for complex transformation or multi-output routing. |

---

## ⚙️ Part 2: Elasticsearch Internals (Lucene & Shards)

Elasticsearch is a distributed wrapper around **Apache Lucene**.

### 1. The Inverted Index
How does ES search 1 billion logs in 10ms?
* It does not scan every row (like SQL `LIKE %text%`).
* It builds an **Inverted Index** (like the index at the back of a book).
    * `Term: "Error"` -> `Doc IDs: [1, 5, 9]`
    * `Term: "Login"` -> `Doc IDs: [2, 3]`
* **Tokenization:** The sentence "User login failed" is split into `["user", "login", "failed"]`.

### 2. Shards & Replicas
* **Index:** A logical namespace (e.g., `logs-2023-10-01`).
* **Shard:** A self-contained Lucene instance. An index is split into $N$ shards.
* **Replica:** A copy of a shard. Provides **High Availability** and **Read Throughput**.

> **⚠️ Senior Trap:** **Over-sharding.**
> Each shard has overhead (File descriptors, Heap).
> * **Rule of Thumb:** Keep shard size between **10GB - 50GB**.
> * having 1,000 tiny shards will kill the Cluster State updates.



---

## 🌡️ Part 3: Scaling Strategies (Hot-Warm-Cold)

Logs lose value over time. Yesterday's errors are critical. Last month's are just for audit.

### 1. The Architecture
* **Hot Nodes:**
    * **Hardware:** High CPU, NVMe SSD.
    * **Data:** Today's logs (Ingest & Heavy Search).
* **Warm Nodes:**
    * **Hardware:** High Storage, HDD/SATA SSD.
    * **Data:** Last 7 days. (Read-Only, occasional search).
* **Cold Nodes:**
    * **Hardware:** Cheapest Disk.
    * **Data:** Last 30 days. (Frozen indices).
* **Frozen/Archive:** Snapshot to S3/GCS.

### 2. Index Lifecycle Management (ILM)
Automate the movement:
* **Phase 1:** Ingest to `Hot`.
* **Phase 2:** After 2 days, move to `Warm`, perform **Force Merge** (reduce segment count).
* **Phase 3:** After 30 days, Delete or Snapshot.

---

## 💥 Part 4: The "Mapping Explosion" (Schema Design)

This is the #1 reason ES clusters crash.

### The Problem
* You send a JSON log: `{"user_id": 123}`. ES detects `Long`.
* Next log: `{"user_id": "ABC"}`. ES errors out (Type Mismatch).
* **Dynamic Mapping:** By default, ES creates a new field for every new JSON key it sees.
* **Explosion:** If you log a user-input object `{"dynamic_key_1": "val", "dynamic_key_2": "val"...}`, ES creates thousands of fields in the mapping. The cluster state becomes huge. All nodes crash trying to sync it.

### The Senior Fix
1.  **Strict Mapping:** Define your schema explicitly. Disable dynamic mapping.
2.  **Flattened Data Type:** If you must dump a messy JSON object, map that specific field as `type: "flattened"`. ES treats the whole sub-object as one "bag of keywords" rather than creating unique fields.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Slow Ingestion (Indexing Latency)
**Interviewer:** *"Our ingestion rate is lagging. We can't keep up with the logs coming from Kafka."*

* ✅ **Senior Answer:** "**Tune for Write Speed.**"
    1.  **Refresh Interval:** Increase `index.refresh_interval` from `1s` to `30s`. (ES creates new Lucene segments every second by default. This is expensive).
    2.  **Disable Replicas:** During massive bulk loads, set `replicas=0`. Re-enable them after the load finishes.
    3.  **Bulk Size:** Ensure Logstash/Beats is sending data in batches (e.g., 5-15MB per bulk request), not single documents.

### Scenario B: Deep Pagination (The "Result 10,000" Problem)
**Interviewer:** *"A user wants to see Page 5,000 of the search results. The query times out."*

* **The Reason:** `from: 50000, size: 10` requires each shard to fetch top 50,010 results, sort them, and return them to the coordinating node. It kills the heap.
* ✅ **Senior Answer:** "**Search After / Scroll API.**"
    * Don't use `from/size` for deep paging.
    * Use **`search_after`**: "Give me 10 results *after* the Sort Value of the last result I saw." This is efficient (stateless).

### Scenario C: Split Brain (Network Partition)
**Interviewer:** *"We lost network connectivity between Master nodes. Now we have two clusters."*

* ✅ **Senior Answer:** "**Minimum Master Nodes (Quorum).**"
    * **Legacy (ES < 7):** Set `discovery.zen.minimum_master_nodes = (N/2) + 1`. If you have 3 masters, you need 2 to elect a leader.
    * **Modern (ES 7+):** This is handled automatically by the cluster bootstrapping, but you must ensure you have **3 dedicated Master-eligible nodes** (not 2) to ensure a majority vote is possible.

### Scenario D: "Text" vs. "Keyword"
**Interviewer:** *"When should I map a string as `text` and when as `keyword`?"*

* ✅ **Senior Answer:**
    * **Text:** For Full-Text Search (Analyzed).
        * "Error connection timed out" -> Tokenized to `["error", "connection", "timed"]`.
        * Use for: Message bodies, descriptions.
    * **Keyword:** For Exact Matching/Aggregations (Not Analyzed).
        * "Error connection timed out" -> Stored as exact string.
        * Use for: `Status` ("404"), `Email`, `HostID`.
        * **Crucial:** You cannot run aggregations (Group By) on `Text` fields efficiently. Use `Keyword`.

---