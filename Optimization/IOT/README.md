# High-Throughput IoT Ingestion Architecture (2M RPS)

This document outlines the diagnosis and architectural resolution for a system handling 10 million IoT sensors generating 2 million POST requests per second.

---

## 1. The Problem: Data Jitter and Gaps
The "jittery" data intervals and missing points on the frontend indicate that the system is suffering from **Resource Exhaustion** and **Network Backpressure**.

### Why TCP is a Major Factor
**TCP (Transmission Control Protocol)** is built for reliability, but at a scale of 2M RPS, its strengths become weaknesses:
* **The 3-Way Handshake:** Every request requires a `SYN` -> `SYN-ACK` -> `ACK` cycle. This consumes massive CPU and network overhead.
* **Head-of-Line (HoL) Blocking:** If one packet is lost, TCP stops all subsequent data from being processed until the lost packet is retransmitted. This causes the "gaps" in your graph.
* **TIME_WAIT State:** Servers have a limited number of ephemeral ports. Rapidly opening and closing TCP connections for POST requests leads to port exhaustion, where new requests are flatly rejected.

---

## 2. Deep Dive: Bottleneck Identification

### A. Database Write Contention (The "Wall")
MySQL is an ACID-compliant relational database. At 2M RPS, it faces:
* **Disk I/O Saturation:** Writing millions of rows per second to B-Tree indexes causes massive "IO Wait."
* **Locking Mechanisms:** Row-level locks and transaction logs (Undo/Redo) create a massive bottleneck.

### B. Connection Exhaustion
If every sensor attempt creates a direct or even pooled connection to MySQL, you will exceed the `max_connections` limit instantly, resulting in `500 Internal Server Errors`.

### C. Application Layer "Stop-the-World"
In a synchronous model (Request -> DB Write -> Response), the API waits for the DB. If the DB lags by even 10ms, your API worker threads saturate, and the server stops accepting new traffic.

---

## 3. The Solution: Step-by-Step Evolution



### Step 1: Decouple with a Message Queue (Asynchronous Ingestion)
Move from a synchronous write to an asynchronous buffer using **Apache Kafka** or **AWS Kinesis**.
* **Action:** The API validates the packet and immediately pushes it to a Kafka Topic.
* **Benefit:** Kafka can ingest millions of events per second as an append-only log, shielding the database from direct traffic spikes.

### Step 2: Swap MySQL for a Time-Series Database (TSDB)
Standard MySQL is not the right tool for this volume of timestamped data.
* **Recommendation:** Move to **InfluxDB**, **TimescaleDB**, or **ClickHouse**.
* **Benefit:** These use **LSM Trees** (Log-Structured Merge-trees) that turn random writes into sequential disk I/O, allowing for 10x–100x higher write throughput.

### Step 3: Implement Micro-Batching
Do not insert one row at a time.
* **Action:** Write a **Consumer Service** (in Java/Go) that pulls data from Kafka and performs **Bulk Inserts** (e.g., 5,000 rows per single `INSERT` statement).
* **Benefit:** Reduces the overhead of transaction commits and network roundtrips significantly.

---

## 4. Comparison Table

| Component | Current State | Target Architecture |
| :--- | :--- | :--- |
| **Ingestion** | Synchronous POST | Asynchronous (Kafka/Kinesis) |
| **Database** | Standard MySQL | Time-Series (ClickHouse/Influx) |
| **Strategy** | 1 Insert per Request | Micro-batching (Bulk Inserts) |
| **Protocol** | Heavy HTTP/TCP | Consider MQTT or UDP |

---

## 5. Next Steps: Implementation
To begin this transition, we will focus on building the **High-Performance Producer** in Java.

* **Objective:** Create a non-blocking Kafka producer using `linger.ms` and `batch.size` tuning.
* **Optimization:** Implement Snappy or LZ4 compression to reduce network bandwidth.

---

# Deep Dive: TCP Bottlenecks in High-Scale IoT Ingestion

When a system is pushed to **2 million Requests Per Second (RPS)**, the protocols that usually ensure reliability become the primary sources of failure. This document explains why TCP is contributing to your "jitter" and data loss.

---

## 1. What is TCP?
**TCP (Transmission Control Protocol)** is a connection-oriented protocol. Unlike UDP (which is "fire and forget"), TCP is built for absolute reliability. It ensures that every packet arrives in the correct order and without errors using the **Three-Way Handshake**:

1.  **SYN**: Sensor says, "I want to talk."
2.  **SYN-ACK**: Server says, "I'm ready, let's talk."
3.  **ACK**: Sensor says, "Great, here is the data."



---

## 2. Why TCP Fails at 2M RPS
At this scale, the "reliability" features of TCP create significant overhead and failure points:

### A. Connection Backlog (SYN Backlog)
When 2 million sensors hit the server simultaneously, the Operating System places these `SYN` packets into a "backlog" queue while waiting for the application to accept them.
* **The Problem**: If the API is slow (stalled by MySQL), this queue fills up.
* **The Result**: The OS starts dropping new `SYN` packets. To the sensor, it looks like the server is offline. This is the primary cause of **missing data**.

### B. The "TIME_WAIT" Accumulation
After a sensor finishes a POST request and closes the connection, the socket enters a `TIME_WAIT` state for 60–240 seconds to ensure stray packets are cleared.
* **The Problem**: Linux has a limited number of ephemeral ports (approx. 28,000 to 65,000).
* **The Result**: If you open/close 2M connections/sec, you exhaust available ports in less than a second. The server becomes unable to open new sockets, causing a total ingestion freeze.

### C. TCP Congestion Control & Retries
TCP automatically slows down when it detects dropped packets (congestion).
* **The Problem**: If a packet is dropped, the sensor employs **Exponential Backoff** (waiting longer with each retry).
* **The Result**: This creates **jitter**. Data intended for 12:00:01 might not land until 12:00:05. If the retry limit is hit, the data is lost forever.

### D. Head-of-Line (HoL) Blocking
If one packet in a stream is lost, TCP holds all subsequent packets in the buffer until the missing one is retransmitted.
* **The Result**: This creates **"bursty" data patterns**. Your graphs show a flat line for 5 seconds, followed by a sudden vertical spike of data arriving all at once.

---

## 3. Network-Level Fixes
Before optimizing application code, these "Performance Tuning" steps are required:

| Strategy | Mechanism | Benefit |
| :--- | :--- | :--- |
| **Keep-Alive** | Reuse one connection for multiple POSTs. | Eliminates the 3-way handshake overhead for every request. |
| **TCP Fast Open (TFO)** | Send data during the initial handshake. | Reduces latency by one full Round Trip Time (RTT). |
| **L4 Load Balancing** | Use Layer 4 (Transport) instead of Layer 7. | Bypasses heavy HTTP header inspection at the entry point. |
| **Kernel Tuning** | Increase `tcp_max_syn_backlog` and `somaxconn`. | Allows the OS to hold more pending connections during spikes. |

---

## 4. Moving Forward
Understanding the network layer is critical before writing the producer. Our next step is to implement a **Java-based Kafka Producer** that utilizes **persistent connections** and **batching** to bypass these TCP-level bottlenecks.