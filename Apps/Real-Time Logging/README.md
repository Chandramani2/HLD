# System Design: Real-Time Logging System (Distributed Log Aggregator)

## 1. Requirements Analysis
**Goal:** Design a system like Splunk, ELK Stack (Elasticsearch, Logstash, Kibana), or Datadog that ingests, indexes, and visualizes logs from millions of services.

### Functional Requirements
* **Log Ingestion:** Collect logs from distributed servers/containers.
* **Search & Query:** Search by keywords, time range, and specific tags (e.g., `level:ERROR`).
* **Real-time Visualization:** Dashboards for trends (e.g., "500 Internal Server Error" counts).
* **Alerting:** Trigger notifications on specific patterns or thresholds.
* **Retention:** Configurable retention (e.g., Hot for 7 days, Cold for 1 year).

### Non-Functional Requirements
* **High Throughput:** Handle massive write volumes (e.g., 50k logs/sec).
* **Low Latency:** Logs should be searchable within seconds (< 5s).
* **Scalability:** Must scale horizontally as log volume grows.
* **Reliability:** Zero data loss (At-least-once delivery).

---

## 2. High-Level Architecture

The standard industry pattern is: **Collect $\rightarrow$ Buffer $\rightarrow$ Process $\rightarrow$ Store $\rightarrow$ Visualize**.



### Components Overview

1.  **Log Agent (Sidecar/DaemonSet):**
    * *Examples:* Filebeat, Fluentd, Vector.
    * Runs on the application server. It tails log files or listens to stdout.
    * **Role:** Lightweight filtering and forwarding. It adds basic metadata (Host IP, Service Name) and pushes to the backend.

2.  **Load Balancer (LB):**
    * The entry point for all agents. Distributes traffic to the ingestion layer.

3.  **Message Queue (The Buffer):**
    * *Examples:* Apache Kafka, RabbitMQ.
    * **Critical Role:** Decouples producers (Apps) from consumers (Indexers).
    * **Backpressure:** If the database slows down, logs accumulate here instead of crashing the application servers.

4.  **Log Processor (Stream Processor):**
    * *Examples:* Logstash, Apache Flink.
    * Reads from Kafka.
    * **Tasks:**
        * **Parsing:** Convert raw text to JSON (grok patterns).
        * **Enrichment:** Add GeoIP based on user IP.
        * **Scrubbing:** Remove PII (Credit Cards, Passwords).

5.  **Indexer & Storage (The "Brain"):**
    * *Examples:* Elasticsearch, ClickHouse, OpenSearch.
    * **Hot Storage (SSD):** Stores recent logs (e.g., 7 days) for fast searching.
    * **Cold Storage (S3/Glacier):** Archives old logs for compliance.

6.  **Visualization UI:**
    * *Examples:* Kibana, Grafana.
    * Queries the Indexer and renders charts.

---

## 3. Detailed System Design Concepts

### A. Pull vs. Push Model
* **Push (Agent-based):** Agents send logs to the collector. *Pros:* Real-time, better for short-lived containers. *Cons:* Can DDoS the collector. **(Preferred for Logging)**.
* **Pull:** Collector scrapes endpoints. *Pros:* Central control. *Cons:* High latency, complex networking.

### B. Indexing Strategy (Sharding)
To handle massive data, we use **Time-Based Indices**.
* **Daily Indices:** `logs-service-2023.10.27`.
* **Why?** Deleting old data is a cheap $O(1)$ operation (`DROP INDEX`) rather than an expensive $O(N)$ operation (`DELETE WHERE date < X`).
* **Rollover:** If an index gets too big (e.g., >50GB), roll over to a new index regardless of the time.

### C. Handling High Throughput (Bulk Indexing)
* **Problem:** Inserting logs one by one kills database performance due to disk I/O and network overhead.
* **Solution:** **Batching**. The Log Processor accumulates logs (e.g., 2000 logs or every 5 seconds) and sends a single **Bulk Insert** request.

---

## 4. Senior Developer Interview Q&A

**Q1: The system is receiving a massive spike in logs (e.g., DDoS attack), and the database is struggling. How do you handle this?**
> **Answer:**
> 1.  **Buffer:** Rely on Kafka to absorb the shock. Ensure Kafka has sufficient retention (disk space).
> 2.  **Sampling:** Implement "Dynamic Sampling." If traffic > threshold, drop `DEBUG` and `INFO` logs, keeping only `ERROR` and `WARN`.
> 3.  **Circuit Breaker:** If the DB is unresponsive, the Processor should stop retrying aggressively and back off.

**Q2: How do you design for Multi-Tenancy (SaaS Logging Product)?**
> **Answer:**
> * **Option A (Logical Isolation):** All customers in one index. Add a `tenant_id` field to every document. Query filter: `WHERE tenant_id = 'X'`. *Pros:* Cheap. *Cons:* "Noisy Neighbor" effect.
> * **Option B (Physical Isolation):** One Index per customer (`index_customer_A`). *Pros:* Performance isolation. *Cons:* High resource overhead (too many open file handles).
> * **Verdict:** Use Option A for small customers, Option B for Enterprise tiers.

**Q3: Searching is slow for wildcard queries (e.g., `*Exception*`). Why?**
> **Answer:**
> * Standard Inverted Indices map full tokens (`NullPointerException` -> Doc1). A wildcard forces the engine to scan the entire term dictionary.
> * **Solution:** Use **N-Grams** during indexing. Break words into fragments (`Exc`, `xce`, `cep`...) to allow fast partial matching, at the cost of larger index size.

**Q4: How do you ensure Zero Data Loss?**
> **Answer:**
> * **At Agent:** Local file buffering. If the network is down, the agent writes to a local disk buffer and retries later.
> * **At Queue:** Use replication in Kafka (Replica factor 3) and `acks=all` to ensure data is persisted before acknowledging the agent.
> * **At Storage:** Use a Write Ahead Log (WAL) or Translog in the database.

---

## 5. Summary Checklist
* **Bottleneck:** Disk I/O (Write heavy) and CPU (JSON parsing).
* **Data Structure:** Inverted Index (Search), LSM Tree (Write throughput).
* **CAP Theorem:** AP (Availability over Consistency). It is acceptable for a log to appear 2 seconds late, but unacceptable to reject logs.