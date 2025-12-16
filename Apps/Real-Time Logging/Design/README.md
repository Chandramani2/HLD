# System Design: Distributed Logging System (ELK / Splunk)

## 1. Requirements

### Functional
* **Ingestion:** Collect logs from thousands of services (API servers, DBs, Load Balancers).
* **Search:** Developers must be able to search logs with specific queries (e.g., `level:ERROR AND service:payment`).
* **Real-time:** Logs should be visible within seconds of generation.
* **Alerting:** Trigger notifications on specific patterns (e.g., "500 Internal Server Error" rate > 1%).
* **Visualization:** Dashboards for trends (Volume over time).

### Non-Functional
* **High Throughput:** Handle massive bursts of data during outages.
* **Reliability:** Zero data loss (Log entries are critical for debugging).
* **Scalability:** Horizontal scaling for both storage and ingestion.
* **Cost Efficiency:** Logs are voluminous; storing everything on SSDs is too expensive.

## 2. Estimations (The "Firehose")

* **Scale:** 10,000 Microservices.
* **Volume:**
    * Average: 1 Million logs/sec.
    * Log Size: 1KB per line.
    * Throughput: 1GB/sec (~86 TB/day).
* **Storage:**
    * Retention: 30 days.
    * Total: 86 TB * 30 ≈ **2.5 Petabytes**.
* **Challenge:** Indexing 1GB/sec in real-time is CPU intensive.

---

## 3. High-Level Architecture

[Image of distributed logging architecture ELK stack]

The architecture follows a standard **Pipeline** pattern: Collect -> Buffer -> Process -> Store.

| Component | Tech Choice | Reason |
| :--- | :--- | :--- |
| **Log Shipper** | **Filebeat / Fluentd** | Lightweight agents running on every server (Sidecar/DaemonSet). |
| **Buffer (The Dam)** | **Apache Kafka** | Essential. Decouples fast producers from slower indexers. Prevents data loss during spikes. |
| **Log Processor** | **Logstash / Vector** | Parses raw text, masks PII (passwords), and enriches data (adds GeoIP). |
| **Storage (Hot)** | **Elasticsearch / OpenSearch** | Inverted Index allows $O(1)$ search on any field. |
| **Storage (Cold)** | **AWS S3 / Glacier** | Long-term archival for compliance (cheap). |
| **UI** | **Kibana / Grafana** | Query interface. |

---

## 4. Data Storage & Schema Strategy

We do not use a single huge table. We use **Time-Based Indices**.

### A. Index Management (ILM)
Instead of one index `logs_all`, we create a new index every day (or hour).
* `logs-payment-2023-10-01`
* `logs-payment-2023-10-02`

**Why?**
1.  **Deleting Old Data:** Deleting rows in a database is slow (fragmentation). Dropping an entire file (Index) is instant.
2.  **Performance:** Searching "Last 15 minutes" only hits today's index, ignoring the other 29 days.

### B. Document Schema (JSON)
Logs must be structured. Raw text is useless for analytics.
```json
{
  "@timestamp": "2023-10-27T10:00:00Z",
  "level": "ERROR",
  "service": "checkout-service",
  "trace_id": "abc-123-xyz", // Distributed Tracing ID
  "message": "Payment gateway timeout",
  "meta": {
    "user_id": 555,
    "region": "us-east-1"
  }
}
```

---

## 5. Senior Interview Topics & Logic

### Q1: "How do you handle a 'Thundering Herd' (Traffic Spike)?"
**Scenario:** A major outage occurs. Every service starts throwing errors. Log volume jumps 100x. Elasticsearch cannot index this fast and crashes.
**Solution:** **Kafka as a Shock Absorber.**
1.  **Producers (Services):** Push logs to Kafka topics. Kafka is optimized for sequential disk writes and can handle millions of msg/sec easily.
2.  **Consumers (Logstash):** Read from Kafka at their own pace.
3.  **Result:** The logs might be delayed by 5 minutes (Lag), but **no data is lost**, and the DB stays alive.

### Q2: "How do we reduce cost for Petabytes of logs?"
**Solution:** **Hot-Warm-Cold Architecture.**
1.  **Hot (Day 1-3):** Stored on NVMe SSDs (High CPU/RAM nodes). Fastest search.
2.  **Warm (Day 4-14):** Moved to HDD. Merged segments (Force Merge). Slower search.
3.  **Cold (Day 15+):** Snapshot to S3 (Object Storage). Index is closed/frozen.
    * *To search:* You must "thaw" (restore) the index first.
    * *Cost Savings:* 90% cheaper than keeping everything on SSD.

### Q3: "How do you search across distributed systems?" (Correlation)
**Problem:** A user request hits Frontend -> API -> Auth -> Database. An error happens in Database. How do you find the Frontend log?
**Solution:** **Distributed Tracing (Trace IDs).**
1.  **Entry:** The Load Balancer generates a unique `trace_id` (UUID).
2.  **Propagation:** This ID is passed in HTTP Headers (`X-Request-ID`) to every downstream service.
3.  **Logging:** Every log statement automatically includes this ID.
4.  **Query:** `trace_id: "abc-123"` reveals the full journey of that request across all services.

---

## 6. Implementation Details (Agents)

How do logs get off the server?

### Pattern A: Sidecar (Container World)
* **Setup:** Every App container has a tiny "buddy" container (Filebeat) in the same Pod.
* **Pros:** Isolation. Can configure specific rules for that app.
* **Cons:** High resource overhead (100 apps = 100 log agents).

### Pattern B: DaemonSet (Node Level) - *Preferred*
* **Setup:** One Log Agent runs per Physical Machine (Node). It mounts `/var/log/containers/` and reads logs from all apps on that machine.
* **Pros:** Efficient. Only 1 agent per server.
* **Cons:** Parsing complex multi-line logs (like Java Stack Traces) from mixed apps can be messy.

---

## 7. Reliability (The "At Least Once" Guarantee)

1.  **Buffering on Agent:** If Kafka is down, Filebeat buffers logs in memory/file on the host. If the buffer fills, it pauses (Backpressure) or drops (Data Loss - depending on config).
2.  **Kafka Replication:** Factor of 3. Even if a broker dies, the log exists on 2 other nodes.
3.  **Dead Letter Queue (DLQ):** If a log is malformed (bad JSON) and crashes the parser, do not block the pipeline. Move it to a "Dead Letter" S3 bucket for manual inspection.

---

## 8. Summary Checklist

* [ ] **Shipper:** Fluentd / Filebeat (DaemonSet).
* [ ] **Buffer:** Kafka (Mandatory for reliability).
* [ ] **Storage:** Elasticsearch (Hot/Warm tiers).
* [ ] **Schema:** Time-partitioned Indices per day.
* [ ] **Cost:** Aggressive retention policies + S3 archiving.
* [ ] **Tracing:** Standardize `trace_id` across all microservices.