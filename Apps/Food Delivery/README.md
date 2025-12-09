# 🍔 The Senior System Design Playbook: Design Food Delivery (UberEats/DoorDash)

> **Target Audience:** Senior Engineers & Product Architects  
> **Goal:** Design a **3-Sided Marketplace** handling **High-Burst Traffic** (Lunch/Dinner), **Complex State Machines**, and **Logistics Optimization**.

In a senior interview, "Design Food Delivery" is not just about moving a car on a map. It is about **Menu Data Modeling** (JSON vs SQL), **Order Batching** (Traveling Salesman Problem), and **Just-In-Time Logistics**.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The "Hangry" User](#-part-1-requirements--the-hangry-user)
2. [Part 2: High-Level Architecture (The 3 Apps)](#-part-2-high-level-architecture-the-3-apps)
3. [Part 3: Menu Data Modeling (The Hidden Trap)](#-part-3-menu-data-modeling-the-hidden-trap)
4. [Part 4: The Order State Machine (ACID)](#-part-4-the-order-state-machine-acid)
5. [Part 5: Logistics & Batching (The Efficiency Engine)](#-part-5-logistics--batching-the-efficiency-engine)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The "Hangry" User

### 1. Requirements
* **Customer App:** Search/Filter food, View Menus, Place Orders, Track Delivery.
* **Restaurant App (Merchant):** Edit Menu, Accept/Reject Orders, Update "Prep Time".
* **Courier App (Dasher):** Accept Delivery, Navigation, Earnings.
* **Non-Functional:**
    * **Low Latency:** Menu updates must be fast.
    * **High Consistency:** No over-selling (Ordering an item that is out of stock).
    * **Resiliency:** If the Restaurant tablet dies, the order shouldn't hang forever.

### 2. Traffic Patterns (The "Lunch Rush")
Unlike Facebook (steady traffic), Food Delivery has massive spikes.
* **11:30 AM - 1:00 PM:** 100x Load.
* **7:00 PM - 9:00 PM:** 100x Load.
* **Implication:** Database auto-scaling is too slow. You need **Scheduled Scaling** (Pre-warm servers at 11:00 AM) and aggressive caching.

---

## 🏗️ Part 2: High-Level Architecture (The 3 Apps)

We are building three distinct systems that share a backend.

[Image of Food Delivery High Level Architecture Diagram]

### 1. The Core Services
* **Merchant Service:** Manages menus, prices, and restaurant hours.
* **Order Service:** The Orchestrator. Handles the Transaction lifecycle.
* **Search Service:** Elasticsearch. Handles "Italian food near me."
* **Dispatch Service:** Matches Orders -> Couriers.

### 2. Communication Strategy
* **Customer -> Server:** HTTP/REST (Browsing). WebSockets (Tracking).
* **Restaurant -> Server:** **Long Polling / WebSockets**.
    * *Why?* Restaurants act like servers. They need to receive orders instantly. A "Push" model is required.
* **Courier -> Server:** WebSockets (GPS Stream).

---

## 📜 Part 3: Menu Data Modeling (The Hidden Trap)

**Interviewer:** *"Design the database schema for the Menu."*
* **The Trap:** Trying to do this in a rigid SQL schema (Rows/Columns).
* **The Reality:** Menus are deeply nested and unstructured.
    * *Pizza -> Sizes (S/M/L) -> Crusts (Thin/Thick) -> Toppings (Pepperoni/Mushrooms) -> Extras.*
    * Every restaurant has a different structure.

### The Solution: Document Store (NoSQL)
Use **MongoDB** or **DynamoDB**. Store the Menu as a nested **JSON Document**.

```json
{
  "restaurant_id": "123",
  "menu_sections": [
    {
      "name": "Pizza",
      "items": [
        {
          "name": "Margherita",
          "price": 12.00,
          "modifiers": [
             {"group": "Size", "options": ["S", "M", "L"]},
             {"group": "Toppings", "multi_select": true}
          ]
        }
      ]
    }
  ]
}
```
# 🍕 Food Delivery System Design Playbook
> **Level:** Senior / Architect
> **Focus:** High Scalability, Consistency, and Optimization

This repository documents the architectural patterns, state machines, and logistics logic required to build a scalable Food Delivery platform (e.g., UberEats, DoorDash, Swiggy).

---

## ⚡ High-Level Optimization: The Menu
**Constraint:** The system is extremely **Read-Heavy**.
* **Ratio:** 1,000,000 Reads : 1 Update.

### Strategy
1.  **Storage:** Store the hierarchical menu structure in **NoSQL** (Document Store like MongoDB).
2.  **Caching:** Cache the *entire* JSON blob in **Redis**.
3.  **Access:** Serve reads directly from Redis (RAM) to avoid DB hits.
4.  **Consistency:** **Invalidate** the cache immediately upon any update (Price change, Out of Stock).

---

## 🔄 Part 4: The Order State Machine (ACID)
An order is not just a static entry; it is a workflow involving three parties: **User, Restaurant, Courier**.

### 1. The States
| State | Description |
| :--- | :--- |
| `CREATED` | User pays. Money held in **Escrow**. |
| `SENT_TO_MERCHANT` | Restaurant tablet rings/notifies. |
| `CONFIRMED` | Restaurant clicks "Accept". *(If Rejected -> Trigger Refund)*. |
| `PREPARING` | Kitchen is actively cooking. |
| `READY_FOR_PICKUP` | Food is bagged. |
| `COURIER_ASSIGNED` | Driver is on the way. |
| `PICKED_UP` | Driver has the food. |
| `DELIVERED` | Success. Funds released. |

### 2. Handling Failures (The "Tablet Died" Scenario)
**Problem:** Order is sent, but the restaurant's WiFi is down. The order sits in `SENT_TO_MERCHANT` for 10+ mins.
**Solution:** Timeouts + Fallbacks (Twilio).

1.  **Timer:** Start a Priority Queue timer when order enters `SENT_TO_MERCHANT`.
2.  **Check:** If state is not `CONFIRMED` in **3 minutes**:
    * Trigger automated **Phone Call (IVR)** to restaurant landline.
3.  **Fallback:** If still no answer:
    * Auto-cancel order.
    * Refund User.
    * Mark Restaurant as **"Offline"**.

---

## 🛵 Part 5: Logistics & Batching
*The Efficiency Engine - Optimizing Unit Economics.*

### 1. The "Just-In-Time" Assignment
**Problem:** Assigning a driver immediately causes them to wait 20 mins at the restaurant (Food Prep). Driver loses money.

**✅ Solution: Machine Learning Prediction**
* **Inputs:** `DayOfWeek`, `ItemCount`, `RestaurantHistoricalAvg`.
* **Model Output:** "This Pizza takes 18 mins."
* **Dispatch Logic:**
    ```text
    DispatchTime = OrderPlacedTime + PredictedPrepTime - DriverTravelTime
    ```

### 2. Order Batching (The TSP Variant)
**Scenario:** Two users in the same building order from the same place.

* **Naive Approach:** Send 2 drivers. (High Cost).
* **Senior Approach:** **Batching**.
    * Look for orders within a **Time Window** (e.g., 5 mins) and **Geo-Window** (e.g., 1 mile).
    * **Route:** `Restaurant` -> `User A` -> `User B`.
    * **Constraint:** Enforce "Max Delay" (e.g., User B cannot be delayed > 7 mins).

---

## 🧠 Part 6: Senior Q&A Scenarios

### Scenario A: Geo + Text Search
> **Interviewer:** "I want to search for 'Burgers' that are 'Open Now' and 'Within 5 miles'."

**✅ Answer: Elasticsearch**
Do not use SQL `LIKE`. Use a Composite Query:
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "tags": "burger" }},
        { "filter": { "term": { "status": "open" }}},
        { "filter": {
            "geo_distance": {
              "distance": "5km",
              "location": { "lat": 40.73, "lon": -74.1 }
            }
        }}
      ]
    }
  }
}
```
### Scenario B: Avoiding "The Double Order"
> **Interviewer:** "User clicks 'Place Order'. The app hangs/lags. They click again in frustration. We charge them twice."

**✅ Answer: Idempotency Key (UUID)**
We use a unique identifier to ensure an operation is processed exactly once, regardless of how many times the request is received.

1.  **Generation:** The Client generates a unique UUID (`Order_X`) when the checkout screen loads.
2.  **Click 1:** Server processes `Order_X` $\rightarrow$ Returns `Success`.
3.  **Click 2:** Server checks the DB/Cache, sees `Order_X` already exists.
    * **Action:** Immediately returns the result of the *first* click (replay the response).
    * **Result:** No new order is created, and the user is only charged once.

---

### Scenario C: Driver Location Updates
> **Interviewer:** "We have 500k active drivers sending GPS updates every 3 seconds. How do we show this on the User's map without crashing the DB?"

**✅ Answer: Redis Pub/Sub (Ephemeral Data)**
Writing high-frequency, ephemeral data to a disk-based database (like Postgres) will cause IOPS saturation.



* **Strategy:** RAM-to-RAM data flow.
* **The Flow:**
    1.  **Driver App** sends coordinates via WebSocket.
    2.  **Server** publishes payload to a **Redis Pub/Sub Channel** (e.g., `channel_order_123`).
    3.  **Customer App** (subscriber) listens to `channel_order_123` via WebSocket.
* **Archiving:** A separate background worker asynchronously dumps data to **Cassandra** (wide-column store) for "Trip History" and analytics later.

---

### Scenario D: The "Sold Out" Item
> **Interviewer:** "A restaurant runs out of Chicken. 5 users have it in their cart and hit checkout simultaneously."

**✅ Answer: Optimistic Locking & Late Validation**
Real-time inventory sync across thousands of distributed devices is difficult. We favor "Late Validation."

1.  **Trigger:** Restaurant tablet updates status to "Sold Out".
2.  **Cache:** System invalidates the **Redis** menu cache.
3.  **Checkout Validation:**
    * Just before the payment gateway transaction, the server performs a **final read** against the Master Database.
    * **Logic:**
        ```python
        if item.status == 'sold_out':
            raise Error("Item no longer available")
            return RejectOrder()
        ```

---

## 🛠 Tech Stack Summary

| Component | Technology | Reasoning |
| :--- | :--- | :--- |
| **Menu** | **NoSQL (JSON) + Redis** | Handles complex, deep-nested JSON structures with high read throughput. |
| **Order** | **SQL (Postgres)** | Strict **ACID** compliance is required for financial state transitions and payments. |
| **Dispatch** | **Python / ML** | Best ecosystem for complex math, prep-time prediction models, and batching algorithms. |
| **Search** | **Elasticsearch** | Specialized optimized index for Geo-spatial (lat/long) + Full-text (fuzzy match) search. |
| **Live Tracking** | **Redis Pub/Sub** | Extremely low latency for handling high-throughput, ephemeral GPS data. |