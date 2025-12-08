# 🧮 The Senior Back-of-the-Envelope (BotE) Playbook

> **Target Audience:** Senior Engineers, Tech Leads  
> **Goal:** Validate architectural decisions with "napkin math" before writing a single line of code.

In a senior interview, you don't calculate to get the *exact* number. You calculate to determine the **Order of Magnitude**.
* *Is this feasible on one server?*
* *Do we need a cluster?*
* *Will the database disk fill up in a month or a decade?*

---

## 📖 Table of Contents
1. [Part 1: The Magic Numbers (Memorize These)](#-part-1-the-magic-numbers-memorize-these)
2. [Part 2: The Golden Formulas](#-part-2-the-golden-formulas)
3. [Part 3: Latency Numbers (Jeff Dean's List)](#-part-3-latency-numbers-latency-at-scale)
4. [Part 4: The Estimation Process](#-part-4-the-estimation-process-step-by-step)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🔮 Part 1: The Magic Numbers (Memorization)

You cannot pull out a calculator in an interview. You must use "Engineering Approximations."

### 1. Powers of Two (Approximation)
The most important trick: **$2^{10} \approx 10^3$ (1,000)**.

| Power | Exact Value | Approximation | Unit |
| :--- | :--- | :--- | :--- |
| $2^{10}$ | 1,024 | **1 Thousand** ($10^3$) | 1 KB |
| $2^{20}$ | 1,048,576 | **1 Million** ($10^6$) | 1 MB |
| $2^{30}$ | 1,073,741,824 | **1 Billion** ($10^9$) | 1 GB |
| $2^{40}$ | ~1 Trillion | **1 Trillion** ($10^{12}$) | 1 TB |
| $2^{50}$ | ~1 Quadrillion | **1 Quadrillion** ($10^{15}$) | 1 PB |

### 2. Time Constants
Simplify division by rounding the seconds in a day.

* **Seconds in a Day:** Actual: `86,400`. **Interview Math:** `100,000` ($10^5$).
* **Requests per Month:** If QPS is 1, requests/month $\approx 2.5$ Million.

### 3. Data Types Sizes
* **Char:** 1 Byte (ASCII) / 2 Bytes (Unicode).
* **Integer/Long:** 4 Bytes / 8 Bytes.
* **UUID:** 16 Bytes (standard).
* **Reference/Pointer:** 8 Bytes (64-bit machine).

---

## 📐 Part 2: The Golden Formulas

### 1. QPS (Queries Per Second)
How much load is hitting your API?
$$QPS = \frac{\text{Daily Active Users (DAU)} \times \text{Requests Per User}}{10^5 \text{ (Seconds in a day)}}$$

> **Senior Tip (Peak Traffic):** Average QPS is useful for storage. **Peak QPS** is useful for capacity planning.
> * Rule of Thumb: **Peak QPS $\approx 2 \times$ Average QPS**.

### 2. Storage Estimation
How much disk space do we need for X years?
$$\text{Total Storage} = \text{Daily Writes} \times \text{Size of Object} \times \text{Days} \times \text{Replication Factor}$$

### 3. Bandwidth Estimation
How big is the network pipe?
$$\text{Bandwidth} = \text{QPS} \times \text{Size of Request}$$

### 4. Memory (Cache) Estimation
What percentage of data is "hot"?
* **Pareto Principle (80/20 Rule):** 20% of the data generates 80% of the traffic.
* **Cache Size:** $\text{Total Daily Data} \times 0.20$.

---

## 🐢 Part 3: Latency Numbers (Latency at Scale)

You must know *why* reading from disk is bad. (Numbers updated for modern hardware, approx).

| Operation | Time | Visual Scale (If 1 CPU cycle = 1 sec) |
| :--- | :--- | :--- |
| **L1 Cache Reference** | 0.5 ns | Heartbeat |
| **Mutex Lock** | 100 ns | Brushing teeth |
| **Main Memory (RAM) Ref** | 100 ns | Brushing teeth |
| **Compress 1KB (Snappy)** | 10 $\mu$s | Buying a coffee |
| **Read 1 MB from RAM** | 250 $\mu$s | Reading a short article |
| **Read 1 MB from SSD** | 1 ms | Taking a lunch break |
| **Packet Round Trip (Same DC)** | 500 $\mu$s | Walking to a neighbor's house |
| **Packet Round Trip (CA to Netherlands)** | 150 ms | **4 Years** |



---

## 🏗️ Part 4: The Estimation Process (Step-by-Step)

Do not just shout out numbers. Show your work.

1.  **Clarify Requirements:** "Are we estimating storage for 1 year or 5 years?"
2.  **State Assumptions:** "I assume 100M DAU. I assume average tweet size is 1KB."
3.  **Calculate (Round Aggressively):** $100M \times 1KB = 100GB$.
4.  **Sanity Check:** "100GB per day is manageable on a single hard drive, but we need sharding for throughput."

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Twitter (Text Storage)
**Interviewer:** *"Estimate the storage required for Twitter for 1 year."*

* **Step 1: Traffic Assumptions**
    * 300M DAU.
    * Each user tweets 2 times/day (Average).
    * Total Tweets/Day = $600M$ tweets.
* **Step 2: Size Assumptions**
    * `tweet_id` (8 bytes) + `user_id` (8 bytes) + `text` (140 bytes) + `metadata` (media links/timestamps) $\approx$ **1 KB** total per tweet.
* **Step 3: Math**
    * Daily Storage = $600M \times 1KB = 600GB$.
    * Yearly Storage = $600GB \times 365 \approx 200TB$ (raw data).
* **Step 4: Senior Adjustment (Replication)**
    * Standard replication is 3x.
    * Total = $200TB \times 3 = 600TB$.
* **Conclusion:** 600TB fits easily into a modern Petabyte-scale cluster (e.g., Cassandra/HDFS).

### Scenario B: Instagram (Bandwidth)
**Interviewer:** *"Design a photo upload system. How much bandwidth do we need?"*

* **Step 1: Assumptions**
    * Total Uploads: 10M photos/day (Write-heavy).
    * Avg Photo Size: 2MB.
* **Step 2: Math**
    * Total Daily Data = $10M \times 2MB = 20TB$.
    * Seconds in Day $\approx 10^5$.
    * Throughput = $20TB / 10^5 = 200MB/s$.
* **Step 3: Senior Analysis**
    * 200MB/s is **Write** speed.
    * Read speed is usually much higher (e.g., 100x Reads).
    * Read Bandwidth $\approx 20GB/s$.
* **Conclusion:** We definitely need a CDN and multiple Edge servers. One server NIC (usually 1Gbps or 10Gbps) cannot handle 20GB/s.

### Scenario C: The "Number of Servers" Check
**Interviewer:** *"How many web servers do we need for 1 Million QPS?"*

* **Step 1: Single Server Capacity**
    * How much can a Tomcat/NodeJS server handle?
    * *Simple API (RAM fetch):* ~10k QPS per box.
    * *Heavy API (Complex logic):* ~1k QPS per box.
* **Step 2: Math**
    * Target: 1,000,000 QPS.
    * Assume Heavy API (safe bet): 1,000 QPS/server.
    * Servers Needed = $1,000,000 / 1,000 = 1,000$ Servers.
* **Step 3: Senior Adjustment (Headroom)**
    * We run servers at 50% CPU to handle spikes.
    * Total = **2,000 Servers**.

---

## 📝 Part 6: Common Pitfalls to Avoid

1.  **Precision Trap:** Don't say "124.56 GB". Say "approx 125 GB". It shows confidence.
2.  **Ignoring Metadata:** Storing a 1KB image isn't just 1KB. You have inodes, DB rows, and indexes. Add overhead (usually +10-20%).
3.  **Confusing "Throughput" with "Latency":**
    * *Throughput:* How many cars pass the bridge (QPS).
    * *Latency:* How long it takes one car to cross (Milliseconds).
    * *Tip:* Async processing improves throughput, but usually hurts latency slightly.
