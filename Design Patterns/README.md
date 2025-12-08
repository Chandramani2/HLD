# 🏛️ The Senior HLD Design Patterns Playbook

> **Target Audience:** System Architects & Tech Leads  
> **Goal:** Standardized solutions for **Migration**, **Resiliency**, **Data Consistency**, and **Observability**.

In a senior interview, you aren't just connecting boxes. You are applying established patterns to solve specific distributed system pains like **Latency**, **Partial Failure**, and **Legacy Coupling**.

---

## 📖 Table of Contents
1. [Part 1: Migration & Integration Patterns (The "Brownfield" Reality)](#-part-1-migration--integration)
2. [Part 2: Resiliency Patterns (Handling Failure)](#-part-2-resiliency-patterns)
3. [Part 3: Data Management Patterns (Saga & CQRS)](#-part-3-data-management-patterns)
4. [Part 4: Deployment & Infra Patterns (Cloud Native)](#-part-4-deployment--infra-patterns)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🦖 Part 1: Migration & Integration (The "Brownfield" Reality)

Seniors rarely build from scratch. You usually have to kill a Monolith.

### 1. The Strangler Fig Pattern 🌿
* **Concept:** Named after a vine that grows around a tree and eventually kills it.
* **Mechanism:**
  1.  Put a Proxy (API Gateway) in front of the Legacy Monolith.
  2.  Build a *new* microservice for a specific slice (e.g., "Search").
  3.  Route `/search` traffic to the new service; route everything else to the Monolith.
  4.  Repeat until the Monolith is empty.
* **Why use it?** Allows risk-free, gradual migration without a "Big Bang" rewrite.



### 2. Anti-Corruption Layer (ACL) 🛡️
* **Problem:** Your new Microservice needs to talk to a legacy Mainframe system that uses archaic XML/SOAP formats.
* **Solution:** Do not let the Mainframe's bad design leak into your clean code.
* **Mechanism:** Build a translation layer (Adapter) that sits between them.
  * *New Service* talks JSON to *ACL*.
  * *ACL* talks XML to *Mainframe*.
* **Benefit:** Isolates technical debt.

---

## 🛡️ Part 2: Resiliency Patterns (Handling Failure)

Distributed systems *will* fail. These patterns prevent a single service failure from taking down the whole company.

### 1. Circuit Breaker 🔌
* **Problem:** Service A calls Service B. Service B hangs. Service A runs out of threads waiting. Service A crashes.
* **Solution:** Wrap the call in a Circuit Breaker.
  * **Closed:** Traffic flows normally.
  * **Open:** If error rate > 50%, block requests immediately (Fail Fast) for 10 seconds.
  * **Half-Open:** Let 1 request through. If it works, Close circuit. If it fails, Open again.



### 2. Bulkhead Pattern 🚢
* **Concept:** Ships are divided into watertight compartments. If the hull is breached, only one section floods.
* **Mechanism:** Isolate resources (Thread Pools, Connection Pools) per dependency.
  * *Pool A (User Service):* 10 Threads.
  * *Pool B (Image Service):* 10 Threads.
* **Result:** If Image Service hangs, it uses up Pool B. Pool A is unaffected, so Users can still login.

### 3. Rate Limiting (Throttling) 🛑
* **Problem:** A client sends too many requests, crashing the server.
* **Solution:** Reject requests over a threshold (e.g., 100 req/sec) with `HTTP 429`. (See Rate Limiter Playbook).

---

## 💾 Part 3: Data Management Patterns (Saga & CQRS)

How to handle transactions across microservices without a single database.

### 1. CQRS (Command Query Responsibility Segregation)
* **Concept:** Separation of concerns between **Writes** (Command) and **Reads** (Query).
* **Implementation:**
  * **Write Model:** Optimized for consistency (SQL). Handles `CreateOrder`.
  * **Read Model:** Optimized for speed (NoSQL/Elasticsearch). Handles `GetOrderHistory`.
  * **Sync:** Async events update the Read DB when the Write DB changes.
* **Benefit:** You can scale Reads independently of Writes (e.g., 100 Read servers, 1 Write server).



### 2. Saga Pattern (Distributed Transactions) 📜
* **Problem:** `OrderService` and `PaymentService` are separate. We need an atomic transaction spanning both.
* **Solution:** A sequence of local transactions.
  1.  `OrderService`: Create Order (Pending). -> Event: `OrderCreated`.
  2.  `PaymentService`: Charge Card. -> Event: `PaymentSuccess`.
  3.  `OrderService`: Update Order (Confirmed).
* **Failure (Compensating Transaction):**
  * If `PaymentService` fails -> Trigger `UndoOrder` in `OrderService`.

### 3. Event Sourcing 📼
* **Concept:** Store the *history* of changes (Events), not just the current state.
* **Storage:** `[AccountCreated, Depost(100), Withdraw(50)]`.
* **Current State:** Calculated by replaying the events ($100 - $50 = $50).
* **Benefit:** Perfect audit trail; ability to "time travel" to debug state.

---

## ☁️ Part 4: Deployment & Infra Patterns

### 1. Sidecar Pattern 🏍️
* **Context:** Kubernetes / Containerization.
* **Concept:** Segregate "infrastructure" logic from "business" logic.
* **Mechanism:** Run a helper container (Sidecar) in the same Pod as the App container.
  * **App Container:** Does `CalculateTax()`.
  * **Sidecar (Envoy/Istio):** Handles SSL, Logging, Metrics, Retries, Network Routing.
* **Benefit:** Developers write code; DevOps manages the Sidecar.



### 2. Backends for Frontends (BFF) 📱💻
* **Problem:** Mobile App needs minimal data (save battery/bandwidth). Desktop Web needs massive data. One "General API" serves both poorly.
* **Solution:** Create distinct API Gateways/Services for each UI.
  * `Mobile-BFF`: Calls internal microservices, strips heavy data, returns light JSON.
  * `Web-BFF`: Calls internal microservices, aggregates everything, returns rich JSON.

### 3. Ambassador Pattern 🕴️
* **Concept:** A smart proxy that lives on the *client* side (or same host) to handle connectivity tasks.
* **Use Case:** A legacy app needs to talk to a secure 3rd party API. Instead of modifying the legacy app code to handle OAuth/TLS, send traffic to `localhost:9000` (Ambassador), which handles the auth and proxies to the external API.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Slow 3rd Party" Problem
**Interviewer:** *"Our `Profile Service` calls an external Identity Provider (Auth0) to get user details. Sometimes Auth0 takes 5 seconds, freezing our threads."*

* ❌ **Junior Answer:** "Increase the timeout." (Just delays the failure).
* ✅ **Senior Answer:** "**Circuit Breaker + Bulkhead + Fallback.**"
  1.  **Circuit Breaker:** If Auth0 fails 5 times, stop calling it.
  2.  **Bulkhead:** Limit Auth0 calls to a specific thread pool so it doesn't starve the rest of the app.
  3.  **Fallback (Cache):** If the circuit is open, return the *last known* user profile from Redis instead of an error.

### Scenario B: Mobile vs. Desktop Performance
**Interviewer:** *"Our Mobile team complains the API returns too much data (payload size). The Web team needs that data. We don't want to maintain two different backends."*

* ✅ **Senior Answer:** "**Backends for Frontends (BFF).**"
  * Keep the core microservices generic.
  * Build a lightweight Node.js/Go layer (BFF) specifically for Mobile.
  * The Mobile BFF calls the backend, filters the JSON fields (GraphQL is also a valid alternative here), and sends only what's needed.

### Scenario C: Migrating a Banking Monolith
**Interviewer:** *"We have a 20-year-old Cobol banking core. We want to build a modern React App. We can't rewrite the Cobol."*

* ✅ **Senior Answer:** "**Anti-Corruption Layer (ACL).**"
  * Treat the Cobol system as a "black box."
  * Build an ACL Microservice.
  * The ACL translates modern REST/JSON requests from React into the fixed-width file formats the Cobol system expects.
  * This prevents the "Cobol rot" from infecting the modern codebase.

### Scenario D: High-Read, Low-Write System (News Site)
**Interviewer:** *"Design a News Portal. Editors write articles occasionally. Millions of users read them instantly."*

* ✅ **Senior Answer:** "**CQRS.**"
  * **Write Side:** Normalized Relational DB (Postgres) for Editors. Ensures data integrity.
  * **Read Side:** Denormalized NoSQL (Mongo) or Search Engine (Elastic).
  * **Sync:** When Editor hits "Publish", an event updates the Read DB.
  * The millions of users hit the Read DB, which is optimized purely for retrieval speed.
