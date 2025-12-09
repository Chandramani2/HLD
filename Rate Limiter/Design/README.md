# 🛡️ High-Level Design: Distributed Rate Limiter

## 1. Overview

A Rate Limiter is a critical component in distributed systems designed to control the amount of incoming traffic to a service. It prevents abuse, ensures fair usage (preventing "noisy neighbors"), and protects backend services from being overwhelmed by unpredictable spikes in traffic.

**Core Architecture Decisions:**
* **Centralized State:** Uses Redis to hold counters, ensuring consistency across multiple rate limiter instances.
* **Low Latency:** Uses an in-memory store and efficient algorithms (Token Bucket/Sliding Window) to minimize the impact on the "Hot Path."
* **Fail-Open Strategy:** If the Rate Limiter service fails, traffic is allowed through to avoid a total system outage.

---

## 2. Use Case Diagram

### Explanation
This diagram describes the functional scope of the system.
* **Actors:** The Client (requesting resources) and the Admin (configuring rules).
* **Boundary:** The Rate Limiter system encapsulates the logic for checking, policy enforcement, and logging.

### Diagram
```mermaid
graph LR
    %% Actors
    Client((Client))
    Admin((Sys Admin))

    %% System Boundary
    subgraph "Rate Limiter System"
        direction TB
        UC1(Request API Resource)
        UC2(Apply Rate Limit Policies)
        UC3(Configure Limit Rules)
        UC4(View Metrics & Logs)
    end

    %% Relationships
    Client --> UC1
    UC1 -.->|include| UC2
    Admin --> UC3
    Admin --> UC4

    %% Styling
    classDef actorStyle fill:#f9f,stroke:#333,stroke-width:2px;
    class Client,Admin actorStyle;
```

---

## 3. Component Diagram

### Explanation
This diagram illustrates the logical building blocks.
* **API Gateway:** Offloads the decision-making.
* **Rate Limiter Service:** Stateless decision engine.
* **Redis:** Stores the ephemeral counters.
* **Rule Config:** Stores the long-term policies (SQL/Cache).

### Diagram
```mermaid
graph TD
    Client[Client Device]

    subgraph "Edge Layer"
        GW[API Gateway]
    end

    subgraph "Rate Limiting Subsystem"
        RL[Rate Limiter Service]
        Config[Rule Config Service]
        Redis[("Distributed State Store\n(Redis)")]
    end

    subgraph "Backend System"
        Microservice[Target Microservice]
    end
    
    Metrics[Metrics Service]

    %% Connections
    Client -- "HTTPS" --> GW
    GW -- "gRPC (Check)" --> RL
    
    RL -- "TCP (Incr/Get)" --> Redis
    RL -- "HTTPS (Rules)" --> Config
    RL -.->|Async Events| Metrics
    
    GW -- "HTTPS (Allowed)" --> Microservice

    %% Styling
    style Redis fill:#f9f,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5
```

---

## 4. Sequence Diagram (Request Flow)

### Explanation
This details the critical "Hot Path".
1.  **Check:** Gateway calls Rate Limiter.
2.  **Atomic Op:** Rate Limiter runs a Lua script in Redis.
3.  **Decision:** Gateway either forwards traffic or returns HTTP 429 immediately.

### Diagram
```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant GW as API Gateway
    participant RL as Rate Limiter Service
    participant R as Redis (State Store)
    participant TS as Target Service

    C->>GW: API Request (e.g., GET /checkout)
    
    activate GW
    GW->>RL: CheckPermission(UserID, IP, URI)
    
    activate RL
    note right of RL: 1. Identify Policy (e.g., 10 req/min)\n2. Construct Redis Key
    RL->>R: EVAL(Lua Script: Check & Increment)
    activate R
    R-->>RL: Returns: { CurrentCount, TimeToReset }
    deactivate R
    
    alt Count <= Limit
        RL-->>GW: Decision: ALLOW
        GW->>TS: Forward API Request
        activate TS
        TS-->>GW: API Response (200 OK)
        deactivate TS
        GW-->>C: API Response (200 OK)
    else Count > Limit
        RL-->>GW: Decision: DENY (Wait 5s)
        deactivate RL
        GW-->>C: HTTP 429 Too Many Requests\nHeader: Retry-After: 5
    end
    deactivate GW
```

---

## 5. Deployment Diagram

### Explanation
This maps the software to the physical cloud infrastructure (AWS/GCP context).
* **Public Subnet:** Contains the Load Balancer.
* **Private Subnet:** Contains the application logic (Gateway, Rate Limiter) and Data tiers.
* **Redis Cluster:** Shows the primary/replica setup for high availability.

### Diagram
```mermaid
graph TD
    %% Node: Client
    subgraph "Client Tier"
        Browser[Mobile App / Browser]
    end

    %% Cloud Boundary
    subgraph "Cloud Infrastructure (AWS/GCP)"
        
        %% Node: Load Balancer
        subgraph "Public Subnet"
            ELB[External Load Balancer]
        end

        %% Node: App Cluster
        subgraph "Kubernetes Cluster (App Zone)"
            GW_Pod[API Gateway Pods]
            RL_Pod[Rate Limiter Pods]
        end

        %% Node: Data Tier
        subgraph "Data Tier (Private Subnet)"
            Redis_Master[("Redis Master")]
            Redis_Replica[("Redis Replica")]
        end

        %% Node: Backend
        subgraph "Backend Services"
            Target_Svc[Microservices Cluster]
        end
    end

    %% Network Connections
    Browser -- "Internet" --> ELB
    ELB -- "Traffic Dist." --> GW_Pod
    GW_Pod -- "Internal gRPC" --> RL_Pod
    RL_Pod -- "Read/Write" --> Redis_Master
    Redis_Master -.->|Replication| Redis_Replica
    GW_Pod -- "Forward Traffic" --> Target_Svc

    %% Styling for "Nodes"
    style ELB fill:#e1f5fe
    style GW_Pod fill:#e1f5fe
    style RL_Pod fill:#e1f5fe
    style Redis_Master fill:#fff9c4
    style Redis_Replica fill:#fff9c4
```