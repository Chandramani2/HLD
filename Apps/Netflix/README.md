# 🎬 The Senior System Design Playbook: Design Netflix

> **Target Audience:** Senior Architects  
> **Goal:** Design a system supporting **200M+ DAU**, **Petabytes of data**, and **Low Latency** streaming.

In a senior interview, "Design Netflix" is **two** distinct interviews:
1.  **The Control Plane:** The App, Search, Recommendations, Billing (Microservices).
2.  **The Data Plane:** The actual video delivery (CDN, Transcoding, Storage).

---

## 📖 Table of Contents
1. [Part 1: Requirements & Back-of-Envelope Math](#-part-1-requirements--back-of-envelope-math)
2. [Part 2: High-Level Architecture (The Big Picture)](#-part-2-high-level-architecture)
3. [Part 3: The Data Plane (Video Storage & Delivery)](#-part-3-the-data-plane-video-storage--delivery)
4. [Part 4: The Control Plane (Microservices & DBs)](#-part-4-the-control-plane-microservices--dbs)
5. [Part 5: Scalability & Fault Tolerance](#-part-5-scalability--fault-tolerance)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & Back-of-Envelope Math

### 1. Requirements
* **Functional:** Upload movies (Admin), Watch movies (User), Search, Recommendations, Save Watch History.
* **Non-Functional:** High Availability (AP over CP), Low Latency (Start time < 200ms), Reliability (No buffering).

### 2. Estimations (The Scary Numbers)
* **Users:** 200 Million DAU.
* **Storage (The "Long Tail"):**
    * Assume 10,000 Titles.
    * Raw 4K Movie = 100GB.
    * *Transcoding:* We need to generate 50 versions of *every* movie (480p, 720p, 1080p, 4k) × (Codecs: H.264, H.265) × (Devices: TV, Mobile).
    * Total per movie $\approx$ 1TB.
    * Total Storage = $10,000 \times 1TB = 10PB$ (Manageable).
* **Bandwidth (The Real Bottleneck):**
    * Peak Traffic: 100M concurrent users.
    * Stream speed: 5 Mbps (HD).
    * Required Bandwidth: $100M \times 5 \text{ Mbps} = 500 \text{ Tbps}$.
    * **Conclusion:** You **cannot** serve this from a central server. You *must* use a CDN.

---

## 🏗️ Part 2: High-Level Architecture

We split the system into three distinct parts to handle the load.

1.  **Client:** TV, Mobile, Web App.
2.  **Backend (AWS/Cloud):** Handles API requests, Metadata, Billing, User Profile (The "Control Plane").
3.  **CDN (Open Connect):** Physical boxes placed inside ISP networks that hold the video files (The "Data Plane").

![Image](https://github.com/user-attachments/assets/ac6601a9-547c-45a1-8466-3241a3c27afb)

![Image](https://github.com/user-attachments/assets/fda00374-98b6-48ff-9a3d-15c548b64b08)

---

## 🎥 Part 3: The Data Plane (Video Storage & Delivery)

How do we turn a raw `.mov` file into a stream on your iPhone?

### 1. Content Onboarding (The Directed Acyclic Graph - DAG)
We don't just "upload". We process.
* **Pattern:** **Choreography / Orchestration (Conductor)**.
* **Flow:**
    1.  Admin uploads Raw File to **AWS S3** (Source of Truth).
    2.  Event triggers the **Transcoding Service**.
    3.  **Parallel Processing:** The movie is sliced into 5-minute chunks.
    4.  **Workers (EC2/Containers):**
        * Worker A: Encodes Chunk 1 to 1080p.
        * Worker B: Encodes Chunk 1 to 720p.
        * Worker C: Extracts Subtitles.
    5.  **Reassembly:** Chunks are merged and validated.
    6.  **Distribution:** Finished files are pushed to CDNs (Warm up the cache).

### 2. The Streaming Protocol (Adaptive Bitrate)
We do not download the file (Download). We stream.
* **Protocol:** **DASH** (Dynamic Adaptive Streaming over HTTP) or **HLS** (Apple).
* **Mechanism:**
    * The video is split into 4-second segments (`.ts` files).
    * A Manifest file (`.m3u8`) lists all segments and available qualities.
    * **Client Logic:** The player detects bandwidth.
        * High Bandwidth -> Fetch `segment_1_4k.ts`
        * Network Drop -> Fetch `segment_2_480p.ts`
    * **Result:** Quality drops, but the video *never pauses*.

### 3. CDN Design (Open Connect)
* Netflix doesn't use Akamai/Cloudfront (too expensive). They built their own.
* **Placement:** Physical server racks are installed *inside* the ISP data centers (e.g., inside Comcast/AT&T hubs).
* **Traffic:** 95% of traffic is served from the ISP's local network (Zero latency). 5% hits the backbone.

---

## 🧠 Part 4: The Control Plane (Microservices & DBs)

This handles everything *except* the video.

### 1. API Gateway (Zuul / Spring Cloud Gateway)
* **Role:** Route traffic, Authentication, Rate Limiting, Load Shedding.
* **Pattern:** **Backends for Frontends (BFF)**.
    * One generic API is bad. The TV app needs 4K assets; the Mobile app needs tiny assets.
    * Use an "Adapter Service" for each device type that formats the response specifically for that device.
* **Microservices:**
  * Small, stateless services like `User Service`, `Billing Service`, `Subscription Service`, and `Recommendation Service`.
  * Communication occurs via **gRPC** or REST.
* **Service Discovery (Eureka):**
  * Acts as a phonebook. When Service A needs to talk to Service B, it asks Eureka for Service B's IP address.
* **Resilience (Hystrix / Resilience4j):**
  * Implements the **Circuit Breaker Pattern**. If the "Ratings Service" fails, the circuit opens, and the app shows the movie without a star rating rather than crashing.
  * **Chaos Monkey:** A tool that randomly terminates instances in production to ensure system resilience.
  
### 2. Database Architecture (Polyglot Persistence)
One DB cannot rule them all.

| Data Type | Database Choice | Reason (Senior Rationale) |
| :--- | :--- | :--- |
| **Billing / Users** | **PostgreSQL / MySQL** | Strong ACID required. Financial data cannot be "eventually consistent." |
| **Search** | **Elasticsearch** | Fuzzy text search, actors, genres. (Inverted Index). |
| **Viewing History** | **Cassandra / HBase** | **Write-Heavy.** Millions of "User X watched Second 45" events per second. LSM Trees handle the write throughput. |
| **Metadata** | **CockroachDB / Spanner** | Movie titles/Cast. Needs global scale but strong consistency. |
| **Caching** | **EVCache (Memcached)** | Netflix prefers Memcached over Redis for pure KV speed and simplicity. |

## 3. The Content Pipeline (Video Processing)

Before a user can click "Play," the raw video goes through an extensive processing pipeline.

1.  **Ingest:** Production houses upload terabytes of raw high-definition video to **AWS S3**.
2.  **Validation:** Automated checks verify file integrity.
3.  **Transcoding (The Encoding Ladder):**
  * The video is broken into small chunks.
  * Chunks are converted into **1,200+ different formats/combinations**.
  * *Purpose:* To support various devices (TVs, mobile, web) and network speeds (Adaptive Bitrate Streaming).
4.  **Packaging:** Digital Rights Management (DRM) is applied.
5.  **Distribution:** Final files are pushed to **Open Connect** servers.

---

## 4. Open Connect (The Data Plane)

This is Netflix's proprietary CDN. Instead of using third-party CDNs, Netflix builds its own global network.

* **OCAs (Open Connect Appliances):** Custom hardware servers placed directly inside Internet Service Providers (ISPs) and Internet Exchange Points (IXPs).
* **Proactive Caching:**
  * Netflix predicts content popularity by region (e.g., *Stranger Things* in the US, Anime in Japan).
  * Content is pushed to local OCAs during off-peak hours (prefetching) rather than waiting for a user to request it.
* **Benefit:** The video stream often does not travel over the internet backbone; it comes from a server inside the user's ISP data center.

---

## 5. Workflow: What Happens When You Click "Play"?

1.  **Request (Client → AWS):**
  * User clicks play. The client sends a request to the **Playback API** on AWS.
2.  **Validation (AWS):**
  * Backend verifies subscription status and licensing rights for the region.
3.  **Steering (AWS → Client):**
  * The **Steering Service** determines the optimal Open Connect Appliance (OCA).
  * It returns a direct URL to the video file (e.g., `https://oca-dallas-isp.netflix.com/movie.mp4`).
4.  **Streaming (Client ↔ Open Connect):**
  * The client connects to the provided OCA IP address.
  * Video streaming begins.
5.  **Adaptive Bitrate (Client Logic):**
  * The client monitors internet speed. If bandwidth drops, it seamlessly switches to a lower bitrate chunk to prevent buffering.
6.  **Telemetry (Client → AWS):**
  * Client sends heartbeats to the Data Pipeline (Kafka) logging quality and viewing position.

### 6. The Recommendation Engine (Spark/Kafka)
* **Online:** Near Real-time. "You just finished Matrix 1, watch Matrix 2."
  * Implementation: **Kafka Stream** processing.
* **Offline:** Batch Processing. "Because you watched Action last month..."
  * Implementation: **Apache Spark** jobs running on Hadoop/S3 Data Lake.

---

## 🛡️ Part 5: Scalability & Fault Tolerance

### 1. Handling Failures (Hystrix / Resilience4j)
* **Circuit Breakers:** If the `Personalization Service` fails, do we show a "500 Error"?
    * **No.** We fallback to a **Static List** (Top 10 Global Movies). The user never knows the recommendation engine is dead.
* **Bulkheads:** The `Upload Service` should not consume threads reserved for the `Play Service`.

### 2. Sharding Strategies
* **User Data:** Shard by `UserID`.
* **Movie Data:** Shard by `MovieID`.
* **Hot Spot Problem:** What if everyone watches "Stranger Things"?
    * The Database isn't the bottleneck (Read-Heavy). The **Cache** is.
    * **Replica Caching:** Replicate the "Stranger Things" metadata key across multiple cache nodes.

---

## Part 6. Technology Stack Summary

| Component | Technology Used |
| :--- | :--- |
| **Cloud Provider** | AWS (EC2, S3, ELB) |
| **CDN** | Netflix Open Connect (FreeBSD + NGINX) |
| **Frontend/API** | React, Node.js, GraphQL (Federated) |
| **Microservices** | Java, Spring Boot, gRPC |
| **Databases** | Cassandra (History), MySQL (Billing), EVCache (Hot Data) |
| **Messaging** | Apache Kafka |
| **Big Data/Analytics** | Apache Spark, Hadoop, Chukwa |
| **Container Orchestration**| Titus (Custom platform), Kubernetes |

## 🧪 Part 7: Senior Level Q&A Scenarios

### Scenario A: Resume Watching (Concurrency)
**Interviewer:** *"I pause the movie on my TV. I open my phone. It should resume at the exact same second. How?"*

* **The Challenge:** Synchronization latency.
* ✅ **Senior Solution:**
  1.  TV sends "Heartbeat" every 10s to **Cassandra** (`User:123, Movie:Matrix, Time:500s`).
  2.  Phone opens App -> Calls `GetLastViewed(User:123)`.
  3.  **Race Condition:** What if TV writes 510s *while* Phone reads 500s?
  4.  **Resolution:** **Last Write Wins (LWW)** is acceptable here. It's okay if we are off by 10 seconds. We don't need strict ACID transactions for bookmarking.

### Scenario B: Global Release (Thundering Herd)
**Interviewer:** *"We release a new season at midnight. 50 Million users hit 'Play' instantly. How does the system survive?"*

* ✅ **Senior Solution:**
  1.  **Pre-warming (Cache):** Load the metadata into Redis/Memcached 1 hour before.
  2.  **Pre-warming (CDN):** Push the video files to the Edge Servers (ISPs) during off-peak hours (3 AM previous day).
  3.  **Jitter:** If the API is overwhelmed, the client retries with `Exponential Backoff + Jitter` to spread the load.

### Scenario C: The "Death Star" Architecture
**Interviewer:** *"We have 1,000 microservices. Debugging a slow request is impossible."*

* ✅ **Senior Solution:** "**Distributed Tracing (Zipkin/Jaeger).**"
  * Every request gets a `TraceID` at the API Gateway.
  * This ID is passed in HTTP headers (`X-B3-TraceId`) to every downstream service.
  * We can visualize the waterfall graph to see exactly which microservice took 200ms.

### Scenario D: Cost Optimization
**Interviewer:** *"S3 storage costs are eating our profits. We store too many formats."*

* ✅ **Senior Solution:** "**Dynamic Storage Policies.**"
  * **Hot Data:** "Stranger Things" (New) -> Store on SSD, Replicate to 100% of CDNs.
  * **Warm Data:** "Friends" (Old but popular) -> Standard S3, Replicate to 50% of CDNs.
  * **Cold Data:** "Obscure 1950s Docu" -> Glacier (Deep Archive). Don't replicate to CDN. Serve from Origin (Higher latency accepted).

---

### **Final Check**
We have covered:
* [x] **Requirements & BotE** (Storage/Bandwidth).
* [x] **Architecture** (Client, Backend, CDN).
* [x] **Protocols** (DASH, HLS).
* [x] **DB Design** (Cassandra for history, Postgres for Billing).
* [x] **Patterns** (Circuit Breaker, Bulkhead, BFF).
* [x] **Scenarios** (Concurrency, Caching).

