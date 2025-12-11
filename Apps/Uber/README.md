# 🚗 The Senior System Design Playbook: Design Uber

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Design a real-time marketplace connecting Riders and Drivers with **low latency**, **perfect matching**, and **fault tolerance**.

In a senior interview, "Design Uber" is not about a map UI. It is about **Spatial Indexing (QuadTrees/Geohashing)**, **Bidirectional Matching**, and handling **Massive Write Throughput** (Location updates).

---

## 📖 Table of Contents
1. [Part 1: Requirements & The Write-Heavy Beast](#-part-1-requirements--the-write-heavy-beast)
2. [Part 2: High-Level Architecture (WebSockets & Services)](#-part-2-high-level-architecture)
3. [Part 3: The Secret Sauce (Geospatial Indexing)](#-part-3-the-secret-sauce-geospatial-indexing)
4. [Part 4: The Dispatch Service (Matching Logic)](#-part-4-the-dispatch-service-matching-logic)
5. [Part 5: Trip State Management (Consistency)](#-part-5-trip-state-management-consistency)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The Write-Heavy Beast

### 1. Requirements
* **Functional:** Rider requests ride, Driver accepts ride, Live Tracking, Payment, Trip History.
* **Non-Functional:**
    * **Low Latency:** Matching must happen in < 2 seconds.
    * **High Consistency:** A driver cannot be assigned to two riders.
    * **High Availability:** If the map breaks, the ride must go on.

### 2. Back-of-the-Envelope Math (The "Heartbeat" Problem)
* **Users:** 100M Riders, 1M Active Drivers.
* **The Constraint:** Drivers send GPS updates every **3 seconds**.
* **Write Load:** $1M \text{ Drivers} / 3 \text{ sec} \approx 333,000$ writes/second.
* **Insight:** A standard SQL database (Postgres/MySQL) will melt under 333k writes/sec of geospatial data. We need a specialized strategy.

---

## 🏗️ Part 2: High-Level Architecture

We need to separate the **Location Updates** (Ephemeral) from the **Trip Data** (Durable).

### 1. The Gateway (Persistent Connections)
* **Riders:** Use HTTPS (REST) to request rides.
* **Drivers:** Use **WebSockets** (Persistent TCP).
    * *Why?* The server needs to *push* "New Ride Request" to the driver instantly. Polling is too slow and battery-draining.

### 2. The Microservices
* **Driver Location Service (DLS):** Ingests the massive stream of GPS points.
* **Map Service:** Calculates ETAs and Routes (Google Maps Wrapper + Custom Logic).
* **Dispatch Service:** The brain. Matches Rider A to Driver B.
* **Trip Service:** Manages the state machine (Requested -> Started -> Ended).

## System Architecture

<img src="https://drive.google.com/file/d/1VeayuMlv29u8ntMTmCNlEWq2Se3vE6Z_/view?usp=drive_link" width="550">

**Key Components:**
* **API Gateway:** NGINX/Envoy handling load balancing and routing.
* **Matching Service:** Uses geospatial indexing (Google S2/H3) to match riders with drivers.
* **Trip Service:** Manages the state machine of the ride.

---

## 📍 Part 3: The Secret Sauce (Geospatial Indexing)

How do we efficiently answer: *"Find all drivers within 2km of this lat/long?"*

### 1. The Naive Approach (SQL)
* Query: `SELECT * FROM drivers WHERE lat BETWEEN x AND y...`
* **Fail:** This requires a full table scan or massive B-Tree indexing updates every 3 seconds. It doesn't scale.

### 2. Geohashing (The Grid)
* **Concept:** Divide the world into a grid. Convert (Lat, Long) into a string.
    * `Lat: 37.7, Long: -122.4` -> Geohash: `9q8yy`.
    * **Property:** Shared prefix = Closeness. `9q8yy` and `9q8yz` are neighbors.
* **Storage:** Redis `Map<Geohash, List<DriverID>>`.
* **Pros:** Fast prefix lookup.
* **Cons:** Edge cases (drivers at the border of two grids).

### 3. QuadTrees (The Uber Solution) 🏆
* **Concept:** A tree structure where every node has 4 children (NW, NE, SW, SE).
* **Mechanism:**
    * Start with the whole world.
    * If a square has > 500 drivers, split it into 4 smaller squares.
    * Repeat until the square is small enough.
* **Search:** Traverse the tree to the leaf node of the User. Return drivers in that node + neighboring nodes.
* **Implementation:** In-Memory (RAM). Fast to update.

### 4. Google S2 (The Modern Standard)
* Uses **Hilbert Curves** to map 2D spheres onto a 1D line.
* Provides better coverage than Geohash near poles and grid edges.
* Used by Tinder, Pokemon Go, and modern Uber.

---

## 🚕 Part 4: The Dispatch Service (Matching Logic)

The Rider hits "Request Ride". What happens?

1.  **Rider Request:** Sent to Dispatch Service.
2.  **Radius Search:** Dispatcher queries **DLS** (using S2/Geohash): "Get available drivers in cell `9q8yy`".
3.  **Filtering & Ranking:**
    * Filter: Only `Status = AVAILABLE`.
    * Rank: ETA, Driver Score, Vehicle Type.
4.  **The "Soft Lock":**
    * We select Driver A.
    * We send a WebSocket message to Driver A: "Accept Ride?"
    * **Crucial:** We must set a TTL (Time To Live) lock. If Driver A doesn't accept in 10s, unlock and try Driver B.
5.  **Optimistic Locking (Redis):**
    * `SET resource_lock:driver_A_id "rider_X" NX EX 10`
    * (NX = Only set if not exists, EX = Expire in 10s).

---

## 🔄 Part 5: Trip State Management (Consistency)

Once the ride starts, we need reliability.

### 1. The State Machine
* **States:** `REQUESTED` -> `MATCHING` -> `DRIVER_ASSIGNED` -> `EN_ROUTE_TO_PICKUP` -> `IN_TRIP` -> `COMPLETED`.
* **Storage:** **PostgreSQL** (ACID is mandatory here).
* **Archive:** Once the trip is `COMPLETED`, move it to **Cassandra/S3** for history (Cold Storage).

### 2. Handling Failure (The "Ghost Ride")
* **Problem:** Driver A accepts, but their phone dies immediately. Rider X is waiting forever.
* **Solution:** **The Heartbeat Monitor**.
    * The server expects a heartbeat from the Driver every 5 seconds.
    * If 3 heartbeats are missed -> Trigger "Driver Disconnect" workflow.
    * Notify Rider -> Re-queue the Trip in Dispatch Service -> Match with Driver B.

---
# System Design: The Uber Heartbeat (Mobile Presence)

## 1. The Constraint
Mobile devices are unreliable. A "missing heartbeat" might mean the driver is in a tunnel, not that the app crashed. Therefore, our system must tolerate *short* failures but act on *long* ones.

## 2. Core Architecture
We piggyback "Liveness" on top of "Location Updates".

### Components
1.  **Driver App:** Uses **Bi-directional streams (gRPC/WebSocket)** over QUIC/TCP.
2.  **Edge Gateway:** Terminates SSL. Tracks connection health.
3.  **Kafka:** Buffers high-velocity streams (Backpressure handling).
4.  **In-Memory Store (Redis/Dynomite):** Stores ephemeral state with **TTL**.

### The Flow
1.  **Update:** Driver sends `{lat: 12.34, long: 56.78}`.
2.  **State Update:** System updates Redis Key `driver_123`.
3.  **TTL Reset:** System sets `EXPIRE driver_123 15`.
4.  **Failure:** If no packet arrives in 15s, Redis key vanishes. Driver is considered "Offline" by the Dispatch system.

---

## 3. Scaling Techniques

### A. Consistent Hashing (Ringpop)
Uber developed **Ringpop**, a library that uses Consistent Hashing on the application layer.
* Services form a ring.
* Requests for `driver_123` are always routed to the same Node X.
* If Node X dies, `driver_123` is handled by Node Y.
* This allows stateful processing in memory without hitting the DB for every heartbeat.

### B. Geo-Sharding
Data is partitioned by **S2 Cells**.
* Drivers in a specific S2 cell (e.g., Downtown SF) communicate with a specific Kafka partition and Redis Shard.
* **Query:** "Find drivers near me" becomes "Query Redis Shard associated with my S2 Cell."

---

# Senior Backend Developer Interview Q&A: Mobile Heartbeats

### Architecture & Protocols
**Q1: Why use WebSockets/gRPC instead of REST for heartbeats?**
**A:** REST requires a new TCP 3-way handshake and SSL negotiation for every request. Doing this every 4 seconds for millions of users consumes massive CPU and increases latency. WebSockets/gRPC keep a single persistent connection open.

**Q2: What happens when a driver enters a tunnel (Loss of Signal)?**
**A:** The persistent connection breaks.
1.  **Server side:** The TTL in Redis expires. The driver effectively "disappears" from the dispatch map.
2.  **Client side:** The app queues the location points locally.
3.  **Reconnection:** When exiting the tunnel, the app batch-uploads the queued points so the trip history remains accurate, and re-establishes the heartbeat.

**Q3: How do you handle the "Thundering Herd" when a cell tower goes down and comes back up?**
**A:** If 50,000 drivers reconnect simultaneously, they can crash the Auth Service.
* **Solution:** Implement **Exponential Backoff with Jitter** on the client.
    * Retry 1: Wait 1s + random(0.1s)
    * Retry 2: Wait 2s + random(0.5s)
    * Retry 3: Wait 4s + random(1s)

### Data & State
**Q4: Redis is fast, but what if the Redis node crashes? Do we lose all driver availability?**
**A:** Yes, transiently. However, because heartbeats arrive every 4 seconds, the data (which drivers are online) will "self-heal" within seconds on the replica/new node as soon as the next heartbeat packets arrive. We prioritize **Availability (AP)** over **Consistency (CP)** here.

**Q5: Why use UDP (or QUIC) for location heartbeats?**
**A:** TCP guarantees delivery and ordering, which causes "Head-of-Line Blocking." If packet 1 is lost, packet 2 waits. For real-time location, we don't care about old data. If packet 1 is lost but packet 2 arrives, we want packet 2 immediately. QUIC solves this.

**Q6: How does "Ringpop" (Uber's sharding library) handle heartbeats?**
**A:** Ringpop acts as a gossip-based membership protocol *between servers*. It allows any server to receive a request and forward it to the specific server that "owns" that driver ID (stateless frontend, stateful backend).

### Logic & Scenarios
**Q7: How do you distinguish between "App Crash" and "Network Loss"?**
**A:** From the server's perspective, they look the same (silence).
* **Refinement:** If the TCP connection is closed cleanly (FIN packet), we know it's a crash or intentional close. If it just times out, it's likely network. We might set a "Ghost" state (likely to return) vs "Offline" (gone).

**Q8: Explain "Geospatial Indexing" in the context of heartbeats.**
**A:** We don't just store "ID: Alive". We store "S2_Cell_ID: [List of Driver IDs]". When a heartbeat comes in, we must update the index. If a driver moves from Cell A to Cell B, we must atomically remove them from A and add to B.

**Q9: What is the impact of GPS drift on heartbeats?**
**A:** Stationary drivers might appear to "jump" around due to GPS inaccuracy.
* **Fix:** Kalman Filters are applied on the client (or server) to smooth out the noise before the heartbeat data is considered valid for dispatching.

**Q10: How do you handle "Zombie Drivers"? (App backgrounded but sending pings)**
**A:** The OS might allow the app to ping in the background even if the driver isn't actually working.
* **Fix:** The heartbeat payload includes `app_state` (Foreground/Background). Dispatch logic prioritizes Foreground drivers.


## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: Handling 300k GPS Writes/Sec
**Interviewer:** *"We can't write to the DB 300,000 times a second. How do we handle this?"*

* ✅ **Senior Answer:** "**Ephemeral vs. Durable Data Strategy.**"
    * **Ephemeral (Live Tracking):** Use **Redis** (Geospatial Index). This data is overwritten every 3 seconds. We don't care about history here.
    * **Durable (Trip History):** Use a **Data Buffer (Kafka)**.
        * The Driver sends GPS.
        * We update Redis (Fast).
        * We *also* push to Kafka.
        * A separate consumer reads Kafka, filters the points (e.g., keep 1 point every 30 seconds), and writes to **Cassandra** for the "Past Trips" feature.

### Scenario B: Ringpop (Datacenter Failure)
**Interviewer:** *"Uber had a famous problem where they needed shared state across nodes without a central DB bottleneck."*

* ✅ **Senior Answer:** "**Consistent Hashing + Gossip Protocol (Ringpop).**"
    * Uber open-sourced **Ringpop**.
    * It allows application servers to form a ring.
    * If I need to find "Driver A", I hash `DriverID` to find which server owns that driver.
    * If that server is down, the Gossip protocol detects it, and the ring rebalances instantly.
    * **Benefit:** Linearly scalable state without a Redis bottleneck.

### Scenario C: Surge Pricing (Dynamic Supply/Demand)
**Interviewer:** *"How do we calculate Surge Pricing in real-time?"*

* ✅ **Senior Answer:** "**Geohash Aggregation.**"
    * We don't calculate surge per user. We calculate it per **Grid Cell**.
    * **Stream Processing (Flink/Spark Streaming):**
        * Input: Stream of "Ride Requests" (Demand) and "Available Drivers" (Supply).
        * Window: Every 5 minutes.
        * Calculation: `Ratio = Demand / Supply`.
        * If `Ratio > 1.5` -> Set `SurgeMultiplier = 1.2x` for that Geohash.
    * This multiplier is cached in Redis and applied to all quotes in that area.

### Scenario D: "I'm at the Airport" (The Queue Problem)
**Interviewer:** *"Airports are different. Drivers queue up. It's FIFO, not 'closest driver'. How to handle?"*

* ✅ **Senior Answer:** "**Geofencing + FIFO Queue.**"
    * Define a Polygon (Geofence) around the Airport waiting area.
    * If a Driver enters the Polygon -> Add them to a **Redis List** (Queue).
    * When a Rider requests from the Airport Terminal -> Pop the head of the Redis List.
    * Ignore GPS distance matching inside the Geofence.

---

### **Final Summary for the Interview**
To win the Uber interview, focus on:
1.  **QuadTrees/S2:** You must know spatial indexing.
2.  **WebSockets:** Essential for driver communication.
3.  **Write Buffer:** Don't write raw GPS to SQL.
4.  **State Machine:** Handle the "Driver Phone Died" edge case.
