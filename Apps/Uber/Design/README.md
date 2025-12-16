# System Design: Ride-Hailing Service (Uber/Lyft)

## 1. Requirements

### Functional
* **Riders:** Request a ride, View ETA, Real-time tracking, Trip History.
* **Drivers:** Accept/Reject rides, Navigation, Earnings.
* **System:** Match Rider to Driver (Dispatch), Calculate Price (Surge).

### Non-Functional
* **Low Latency:** Matching must happen in < 1 minute.
* **Strong Consistency:** A driver cannot be assigned to two riders simultaneously (CP).
* **High Availability:** Location tracking must never go down (AP).

## 2. Estimations (The Scale)

* **Users:** 100 Million MAU / 1 Million Active Drivers.
* **Write Heavy (The Challenge):**
    * Drivers update location every 4 seconds.
    * 1M drivers × 1 update/4s = **250,000 writes per second (QPS)**.
    * This is too high for a standard database to index in real-time.
* **Storage:** Trip history grows indefinitely (Cold storage needed).

---

## 3. Database Architecture

Uber requires a mix of **Real-time Geospatial** storage and **Transactional** storage.

| Component | Database Choice | Reason |
| :--- | :--- | :--- |
| **Trips & Users** | **Sharded MySQL / PostgreSQL** | ACID required. Uber built "Schemaless" on top of MySQL for this. |
| **Driver Location (Live)** | **Redis (Geospatial)** | In-memory storage for super-fast "Find Nearest" queries. |
| **Location History** | **Cassandra** | Write-heavy log of GPS points for analytics/safety. |
| **Dispatch Queue** | **Kafka** | To handle the stream of ride requests and matches asynchronously. |

---

## 4. Schema Design

### A. Trips Service (MySQL - Sharded)
We shard based on `trip_id` or `city_id`. Strong consistency is critical here (State Machine: REQUESTED -> MATCHED -> IN_PROGRESS).

```sql
-- Trips Table
CREATE TABLE trips (
    trip_id BIGSERIAL PRIMARY KEY,
    rider_id BIGINT NOT NULL,
    driver_id BIGINT, -- Nullable until matched
    status VARCHAR(20) CHECK (status IN ('REQUESTED', 'MATCHED', 'STARTED', 'COMPLETED')),
    pickup_lat DOUBLE PRECISION,
    pickup_long DOUBLE PRECISION,
    drop_lat DOUBLE PRECISION,
    drop_long DOUBLE PRECISION,
    fare_amount DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index for finding active trips for a user
CREATE INDEX idx_rider_trips ON trips(rider_id, created_at DESC);
```

### B. Driver Service (MySQL)
Stores static driver data and current status (Online/Offline).

```sql
CREATE TABLE drivers (
    driver_id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    vehicle_info JSONB, -- {'model': 'Toyota Prius', 'plate': 'ABC-123'}
    current_status VARCHAR(20) -- 'AVAILABLE', 'BUSY', 'OFFLINE'
);
```

### C. Active Location Store (Redis)
We do **not** store live locations in MySQL. We use Redis Geospatial commands or an ephemeral store.

**Key:** `driver_locations`
**Value:** `(longitude, latitude, driver_id)`

---

## 5. Senior Interview Queries & Logic

### Q1: "How do we efficiently find the nearest 5 drivers?"
**Naive SQL Approach (Don't do this):**
```sql
-- Calculating distance for 1M drivers is O(N) = System Crash
SELECT * FROM drivers 
WHERE SQRT(POW(lat - user_lat, 2) + POW(long - user_long, 2)) < 5;
```

**Senior Solution (Geospatial Indexing):**
Use **Redis** with Geohashing or Google S2 Geometry.

**Redis Command:**
```redis
GEORADIUS driver_locations -122.4194 37.7749 2 km WITHDIST COUNT 5 ASC
```
**Explanation:** This finds drivers within 2km of the coordinates (San Francisco), sorts by distance, and returns the top 5. It is O(log N) + M, which is extremely fast.

### Q2: "How do we prevent two riders from booking the same driver?"
**Scenario:** Driver A is the closest match for Rider X and Rider Y simultaneously.
**Solution:** Database Transaction with Locking (Optimistic or Pessimistic).

```sql
-- The "Lock" Query
BEGIN;

-- 1. Check if driver is still available
SELECT status FROM drivers WHERE driver_id = 101 FOR UPDATE;

-- 2. If status == 'AVAILABLE', proceed to assign
UPDATE trips SET driver_id = 101, status = 'MATCHED' WHERE trip_id = 500;
UPDATE drivers SET status = 'BUSY' WHERE driver_id = 101;

COMMIT;
```

### Q3: "Calculate Dynamic Pricing (Surge) for a specific area."
**Logic:** Surge is calculated per "S2 Cell" (a hexagonal map zone).
We compare **Demand** (Open Apps/Requests) vs. **Supply** (Available Drivers).

```sql
-- Analytical Query (likely run on a Read Replica or Data Warehouse)
SELECT 
    s2_cell_id, 
    COUNT(CASE WHEN status = 'REQUESTED' THEN 1 END) as demand,
    COUNT(CASE WHEN status = 'AVAILABLE' THEN 1 END) as supply
FROM active_session_snapshot
GROUP BY s2_cell_id;

-- If Demand > Supply * 1.5 THEN Surge_Multiplier = 1.2x
```

---

## 6. API Design (Real-Time Communication)

REST is too slow for driver location updates. We use **WebSockets** or **gRPC** (bidirectional streaming).

### Critical Endpoints

**1. Update Location (Driver -> Server)**
* **Protocol:** WebSocket / TCP (Custom protocol often used to save battery).
* **Payload:**
    ```json
    {
      "driver_id": 101,
      "lat": 37.7749,
      "long": -122.4194,
      "timestamp": 167888888
    }
    ```

**2. Request Ride (Rider -> Server)**
* **Protocol:** REST (POST)
* **Endpoint:** `/v1/trips/request`
* **Response:** Returns `trip_id`. The client then subscribes to a WebSocket channel `trip_updates_{trip_id}` to get the match notification.

---

## 7. The Secret Sauce: QuadTrees & Google S2

In a senior interview, you must explain **how** the map is indexed.

1.  **The Problem:** Lat/Long are continuous numbers. Databases index discrete values.
2.  **The Solution:** Divide the world into a grid (cells).
    * **Geohash:** Encodes a rectangle into a string (e.g., `9q8yy`).
    * **Google S2:** Divides the sphere into **Cells** (active drivers are mapped to a Cell ID).
3.  **The Algorithm:**
    * Convert Rider's Location -> Cell ID.
    * Query: "Give me all drivers in Cell ID X and its 8 neighbors."

---

## 8. Summary Checklist

* [ ] **Database:** MySQL (Sharded) for Trips; Redis for Live Locations.
* [ ] **Consistency:** Strong Consistency (CP) for Booking; Eventual (AP) for Location Tracking.
* [ ] **Indexing:** S2 Geometry / Geohashes for spatial search.
* [ ] **Communication:** WebSockets for the "moving car" animation on the user's phone.