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

# Technical Analysis: TCP Bottlenecks at 2M RPS

## Executive Summary
The primary culprit for the observed "jitter" and data loss is not a failure of the TCP protocol itself, but rather a structural limitation of the **HTTP-over-TCP** pattern. Attempting to sustain **2 million Requests Per Second (RPS)** on a standard server configuration using short-lived connections is physically impossible due to OS-level constraints.

---

## Technical Breakdown: Why TCP is Dropping Data

### 1. The "Three-Way Handshake" Overhead
For every `POST` request, your sensors are likely performing a new TCP handshake.

* **The Math:** 2M requests = 6M packets (`SYN`, `SYN-ACK`, `ACK`) just to initiate the conversation.
* **The Problem:** Your CPU is spending ~80% of its cycles managing handshakes rather than processing payload data. When the CPU saturation hits 100%, the kernel begins ignoring new `SYN` packets.
* **Result:** The sensor perceives the server as "down." **Outcome: Direct Data Loss.**

### 2. Ephemeral Port Exhaustion (The 65k Limit)
This is the most common root cause of intermittent "jitter."



* **The Problem:** A Linux server has roughly 65,535 "ports" available for communication. When a TCP connection closes, the port enters a `TIME_WAIT` state for 60 seconds (by default) to ensure no stray packets are misrouted.
* **The Math:** At 2M requests/sec with a 60-second cooldown, you would mathematically require **120 Million ports**. You only have **65,535**.
* **Result:** The server runs out of addresses to assign to new sensor connections. It rejects all incoming traffic until the timer expires on old ports. **Outcome: Jitter and gaps in telemetry graphs.**

### 3. TCP Congestion Control (The "Slow Down" Signal)
TCP is designed to be a "polite" protocol. If the network or the server is even slightly overwhelmed, TCP triggers a **Congestion Window (CWND)** reduction.

* **The Problem:** The protocol intentionally throttles the rate of data transfer to prevent network collapse.
* **Result:** Data queues up in the sensor's local buffer. Because sensors typically have minimal memory, once that buffer is full, the sensor must delete old data to make room for new readings. **Outcome: Missing Data.**

### 4. The "Accept Queue" Overflow
In Linux, the `tcp_max_syn_backlog` setting acts as the "waiting room" for established connections waiting to be handled by the application.

* **The Problem:** At 2M RPS, this waiting room fills up in microseconds.
* **Result:** Once full, the OS kernel executes a **TCP Drop**. It does not send a "busy" signal; it simply ignores the packet entirely.

---

## The Solution: Architectural Shift to Kafka

By moving to the proposed architecture, we resolve these bottlenecks through three specific mechanisms:

1.  **Persistent Connections (Keep-Alive):** Sensors are configured to keep a single TCP connection open for hours, multiplexing thousands of `POST` requests over one handshake. This completely bypasses **Port Exhaustion**.
2.  **Immediate Buffering:** The Java API no longer processes the data immediately. It accepts the TCP packet, hands it to **Kafka** (in-memory ingestion), and sends an `ACK` back to the sensor in <1ms.
3.  **Backpressure Management:** TCP only slows down if the server is slow. Since the API is now a simple "pass-through" to Kafka, it stays fast enough to keep the TCP window wide open at all times.

---

### Critical Kernel Tuning (Stop-gap)
If you must handle high volume before the Kafka migration is complete, apply these `sysctl` optimizations:

```bash
# Allow reuse of sockets in TIME_WAIT state
net.ipv4.tcp_tw_reuse = 1

# Increase the limit of the socket listen queue
net.core.somaxconn = 65535

# Increase the maximum number of remembered connection requests
net.ipv4.tcp_max_syn_backlog = 65535
```

# Phase 2 Analysis: Post-Connection Bottlenecks at 2M RPS

Even if the TCP handshake and port exhaustion issues are resolved (e.g., via Load Balancer arrays or persistent connections), sustaining **2M RPS** introduces hardware and kernel-level "invisible walls."

---

## 1. Interrupt Storms (The Hardware/IRQ Wall)
When a packet arrives at the Network Interface Card (NIC), it triggers an **Interrupt Request (IRQ)**, forcing the CPU to stop its current task to handle the incoming data.

* **The Problem:** At 2M RPS, the CPU is bombarded with millions of interrupts. The processor spends more time "context switching" (moving between the kernel and your application) than actually processing data.
* **The Result:** **Livelock.** The server appears online, but CPU usage is 100% despite zero progress on your Java logic because it is stuck in an endless loop of acknowledging packet arrivals.



## 2. The "SoftIRQ" Single-Core Bottleneck
By default, Linux often handles all network interrupts on **CPU0**.

* **The Problem:** Even if you have a 64-core server, if the kernel is not configured for **RSS (Receive Side Scaling)**, a single CPU core will hit 100% load while the other 63 cores sit idle.
* **The Result:** Packets are dropped at the NIC buffer because CPU0 cannot clear the queue fast enough.

## 3. Context Switching & Thread Contention
In a standard Java/Spring environment, each request or connection typically attempts to claim a thread.

* **The Problem:** Managing 10,000+ active threads causes the Linux Scheduler to struggle. The overhead of "swapping" thread states in and out of the CPU L1/L2 caches becomes a massive tax on performance.
* **The Result:** High **System CPU** usage but low **User CPU** usage. The application feels "jittery" because threads are stuck in the "Ready" queue waiting for a time slice.

## 4. JVM Garbage Collection (GC) "Stop-the-World"
Handling 2M RPS creates a massive volume of short-lived objects (HTTP requests, JSON strings, Byte buffers).

* **The Problem:** This creates immense pressure on the JVM's "Young Generation" memory.
* **The Math:** If the JVM performs a "Minor GC" pause of just **100ms**, you have just backed up **200,000 requests** ($2,000,000 \times 0.1$).
* **The Result:** This massive backup creates a "micro-spike" that overflows the TCP listen queue, leading to the exact same "dropped packet" symptoms as the original TCP issue.



---

## Summary of Secondary Bottlenecks

| Layer | Component | Failure Mode at 2M RPS |
| :--- | :--- | :--- |
| **Hardware** | NIC / IRQ | CPU "Livelock" due to interrupt saturation. |
| **Kernel** | SoftIRQ | Single-core bottleneck (CPU0) ignores incoming packets. |
| **Memory** | Socket Buffers | `rmem` overflows before the App can "read()" the data. |
| **App (JVM)** | Heap / GC | GC pauses create massive backlogs that trigger TCP drops. |

---

## Technical Recommendations (.md Script)

To mitigate these issues, the following system-level optimizations are required alongside the Kafka migration:

### Kernel Optimization (`/etc/sysctl.conf`)
```bash
# Increase network receive/send buffers (16MB)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Increase the number of packets processed per interrupt cycle (NAPI)
net.core.netdev_budget = 600

# Enable Receive Packet Steering (RPS) to distribute load across cores
# (Example for eth0)
# echo "ff" > /sys/class/net/eth0/queues/rx-0/rps_cpus
```