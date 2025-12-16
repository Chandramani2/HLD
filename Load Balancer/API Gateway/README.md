# API Gateway: Architecture & System Design

An API Gateway is a server that acts as the single entry point for all clients (mobile apps, web browsers, external 3rd parties) trying to access your backend services.

> **🏢 The Office Building Analogy**
>
> If the microservices are the various departments (Billing, HR, Sales) working inside a large office building, the **API Gateway is the Reception Desk**.
>
> It stops visitors at the door, checks their ID, gives them a badge, and gives them directions to the exact room they need, so they don't wander the halls aimlessly.

---

## 1. Core Functions of an API Gateway

The API Gateway abstracts the complexity of the backend from the client. Instead of a mobile app needing to know the IP addresses of 50 different microservices, it only needs to know the URL of the Gateway.

### A. Request Routing (The Traffic Director)
This is the most basic function. The Gateway inspects the incoming request URL and routes it to the correct microservice.

**Example:**
* Client calls: `api.myapp.com/users/123` → **Gateway** routes to **User Service**.
* Client calls: `api.myapp.com/orders/create` → **Gateway** routes to **Order Service**.

### B. Authentication & Authorization (The Security Guard)
Instead of implementing login logic in every single microservice (which is redundant and risky), you offload it to the Gateway.

* **Function:** The Gateway checks the HTTP headers for a valid token (e.g., JWT).
* **Action:**
    * *If valid:* It allows the request to pass.
    * *If invalid:* It returns `401 Unauthorized` immediately, protecting the backend services from even seeing the bad request.

### C. Rate Limiting & Throttling (The Bouncer)
To prevent your system from being overwhelmed by too many requests (DDoS attacks or just a buggy client), the Gateway limits how often a user can call the API.

* **Example:** "User X can only make 100 requests per minute."
* **Result:** If they exceed this, the Gateway returns `429 Too Many Requests`.

### D. Protocol Translation
Old systems might use XML/SOAP, while modern frontends use JSON/REST or gRPC. The Gateway can translate between these.

* **Scenario:** A modern React app sends a JSON request.
* **Process:** The Gateway converts it to XML for a legacy banking system, gets the XML response, converts it back to JSON, and sends it to the app.

### E. Response Aggregation (The Bundler)
This is critical for performance. A single screen on a mobile app might need data from the User Service, the Cart Service, and the Product Service.

* **Without Gateway:** The mobile app makes 3 separate network calls over the internet (slow).
* **With Gateway:** The mobile app makes **1 call** to the Gateway. The Gateway (which is on the same high-speed network as the services) calls all 3 services in parallel, combines the results into one JSON object, and sends it back.

---

## 2. How it is Used in System Design

When designing complex systems, you place the API Gateway at the "Edge" of your network.

### Pattern 1: The Standard Microservices Facade
**Setup:** `Client` → `Load Balancer` → `API Gateway` → `Microservices`

* **Goal:** Decouple the client from the backend.
* **Benefit:** You can refactor your backend (e.g., split one service into two) without breaking the client app, because the Gateway masks the change.



### Pattern 2: Backend for Frontend (BFF)
Sometimes, a Mobile App needs different data than a Desktop Web App (e.g., Mobile needs less data to save battery/bandwidth).

**Setup:**
* **Mobile API Gateway** → Optimized for small payloads.
* **Web API Gateway** → Optimized for rich data.

**Goal:** Tailored experiences for different devices.

### Pattern 3: Offloading "Cross-Cutting Concerns"
In System Design interviews, you should always look for opportunities to simplify individual services. The API Gateway allows you to say:

> "My microservices will purely focus on business logic. I will move SSL termination, Logging, and Auth to the Gateway."

---

## 3. Popular API Gateway Tools

* **Nginx / HAProxy:** Often used as simple, high-performance gateways (requires manual config).
* **AWS API Gateway:** Fully managed, serverless, scales automatically. Great for AWS-heavy stacks.
* **Kong:** Very popular open-source gateway based on Nginx. Highly extensible with plugins.
* **Zuul (Netflix):** An older Java-based gateway, famous for popularizing the pattern in the Spring Boot ecosystem.

---

## 4. Summary: When to use it?

| Use an API Gateway If... | Do NOT use an API Gateway If... |
| :--- | :--- |
| You have a **Microservices** architecture. | You have a **Monolithic** application (it adds unnecessary complexity). |
| You need to centralize security and monitoring. | You need **ultra-low latency** (the gateway adds a tiny hop). |
| You have multiple different clients (Web, Mobile, IoT). | You are building a small internal tool with few users. |

# System Design: Load Balancer vs. API Gateway

## 1. High-Level Difference
To visualize the difference: A **Load Balancer** is the "traffic cop" keeping cars moving, while an **API Gateway** is the "grand concierge" checking IDs, translating languages, and enforcing building rules.

| Feature | Load Balancer (LB) | API Gateway (AG) |
| :--- | :--- | :--- |
| **Primary Goal** | High Availability & Scalability (Traffic Distribution). | API Management, Security & Composition. |
| **OSI Layer** | Mostly **Layer 4** (Transport) & Layer 7. | Exclusively **Layer 7** (Application). |
| **Key Function** | Distributes traffic across healthy servers. | Routes requests to specific microservices based on logic. |
| **Security** | SSL Termination, DDoS protection. | Authentication (JWT/OAuth), Rate Limiting, Whitelisting. |
| **Intelligence** | Low (Routes based on algorithms like Round Robin). | High (Routes based on path, headers, or user identity). |

---

## 2. Component Breakdown

### What is a Load Balancer?
Its main job is **horizontal scaling** and **redundancy**.
* **Traffic Distribution:** Uses algorithms like Round Robin, Least Connections, or IP Hash.
* **Health Checks:** Constantly pings servers; if one fails, it stops sending traffic to it.
* **SSL Offloading:** Decrypts incoming HTTPS traffic to relieve backend servers of the computational burden.

### What is an API Gateway?
It sits between the client and backend services, acting as a single entry point.
* **Request Routing:** Maps endpoints (e.g., `/users`) to specific microservices (e.g., `UserService`).
* **Protocol Translation:** Converts web-friendly REST/HTTP requests into internal protocols like gRPC.
* **Cross-Cutting Concerns:** Handles shared logic like **Auth**, **Rate Limiting**, **Caching**, and **Logging**.
* **Response Aggregation:** Calls multiple microservices and combines data into a single JSON response.

---

## 3. The Architecture: How They Work Together
In a robust system, they are chained together. The Load Balancer handles the raw volume, and the Gateway handles the logic.

**The Flow:**
1.  **The Entry Point (LB):** Client hits DNS -> Resolves to Public Load Balancer (L4).
2.  **The Management Layer (AG):** LB forwards traffic to the API Gateway Cluster. The Gateway authenticates the user and inspects the request.
3.  **The Service Layer (Microservices):** Gateway routes the request to the specific backend service.

---

## 4. Implementation Strategies

### Option A: The Cloud-Native Approach (e.g., AWS)
*Best for: Teams wanting minimal maintenance.*
* **LB:** AWS Application Load Balancer (ALB).
* **AG:** Amazon API Gateway.
* **Setup:** ALB (Public Subnet) -> Private Link -> API Gateway (VPC) -> EC2/Lambda.

### Option B: The Kubernetes (K8s) Approach
*Best for: Microservices needing fine-grained control.*
* **LB:** Cloud Provider LB (AWS NLB / GCP LB).
* **AG:** Ingress Controller (Nginx, Kong, or Istio).
* **Configuration Example (Ingress):**
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: stock-app-ingress
      annotations:
        nginx.ingress.kubernetes.io/limit-rps: "5" # Gateway Logic
    spec:
      rules:
      - host: api.stockbroker.com
        http:
          paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
    ```

### Option C: Open Source (Self-Hosted)
*Best for: On-premise or avoiding vendor lock-in.*
* **LB:** NGINX (Layer 4 config).
* **AG:** Kong or Tyk.
* **Setup:** NGINX terminates SSL -> forwards unencrypted traffic to Kong nodes -> Kong routes to services.

---

## 5. Edge Case: The "Single Gateway" Scenario
*Question: If I only have 1 API Gateway, how does the Load Balancer divide traffic?*

**Answer:** It doesn't "divide" traffic. It acts as a **Passthrough**.
1.  **100% Traffic Flow:** The LB sends every request to the single AG instance.
2.  **Why do this?**
    * **SSL Termination:** The LB handles encryption/decryption, saving the Gateway's CPU.
    * **Security:** The Gateway stays on a private IP; the LB exposes the public IP.
    * **Future Proofing:** When you add a second Gateway, the LB automatically starts balancing without needing DNS changes.
3.  **Risk:** This is a Single Point of Failure (SPOF). If the Gateway crashes, the LB returns a 503 error.

---

## 6. High Availability Architecture (Multiple Gateways)
To remove the single point of failure, you deploy multiple API Gateway instances behind the Load Balancer.

**The Logic Flow:**
1.  **Client Request:** Comes in via HTTPS.
2.  **Public Load Balancer:**
    * Checks Health: "Are Gateway A and Gateway B healthy?"
    * Algorithm: Round Robin (or Least Connections).
    * Action: Forwards request to **Gateway A**.
3.  **Gateway A:**
    * Authenticates User.
    * Routes to **Order Service**.
4.  **Next Client Request:**
    * Public Load Balancer forwards to **Gateway B**.

> **Interview Summary:**
> "A **Load Balancer** ensures system reliability by spreading traffic and preventing server crashes. An **API Gateway** simplifies development by unifying microservices into a single interface and handling cross-cutting concerns like security. In a production system, we typically place a Load Balancer *in front of* a cluster of API Gateways."