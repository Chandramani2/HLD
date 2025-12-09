# ⚖️ The Senior ELB (Elastic Load Balancer) Playbook

> **Target Audience:** Cloud Architects, DevOps, & SREs  
> **Goal:** Master **Layer 7 Routing**, **Extreme Throughput Scaling**, and **Zero-Downtime Deployments**.

In a senior interview, you must explain why you chose an **NLB over an ALB** for high throughput, how **Connection Draining** prevents user errors during deployments, and the hidden costs of **Cross-Zone Load Balancing**.

---

## 📖 Table of Contents
1. [Part 1: The Taxonomy (ALB vs NLB vs GWLB)](#-part-1-the-taxonomy-alb-vs-nlb-vs-gwlb)
2. [Part 2: Routing Logic (Path-Based & Host-Based)](#-part-2-routing-logic-path-based--host-based)
3. [Part 3: Advanced Traffic Management (Stickiness & Draining)](#-part-3-advanced-traffic-management-stickiness--draining)
4. [Part 4: Scaling & Performance (Pre-Warming)](#-part-4-scaling--performance-pre-warming)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: The Taxonomy (ALB vs NLB vs GWLB)

AWS offers 3 main types. Choosing the wrong one is a fatal architecture flaw.

### 1. Application Load Balancer (ALB) - Layer 7
* **Traffic:** HTTP / HTTPS / gRPC.
* **Features:** SSL Termination, WAF integration, Path-based routing (`/api` -> Service A).
* **Speed:** Slower than NLB (inspects packets).
* **IP Address:** **Dynamic**. You must use DNS (CNAME). You *cannot* get a static IP.
* **Use Case:** Microservices, Web Apps, Containers (ECS/EKS).

### 2. Network Load Balancer (NLB) - Layer 4
* **Traffic:** TCP / UDP / TLS.
* **Features:** "Dumb" pass-through. No header inspection.
* **Speed:** **Ultra-low latency**. Handles millions of requests/sec.
* **IP Address:** **Static IP**. Essential for corporate firewalls that whitelist specific IPs.
* **Use Case:** Databases, Gaming (UDP), IoT, or when you need a Static IP.

### 3. Gateway Load Balancer (GWLB) - Layer 3
* **Traffic:** IP Packets.
* **Use Case:** Inspecting traffic via 3rd party virtual appliances (Firewalls, Intrusion Detection Systems).
* **Architecture:** Uses **GENEVE** encapsulation to route traffic transparently through a fleet of security appliances.

[Image of AWS Load Balancer Types Diagram]

---

## 🛣️ Part 2: Routing Logic (Path-Based & Host-Based)

ALB is a "Smart Router". It replaces the need for a separate Nginx fleet in many cases.

### 1. Listener Rules
Instead of one LB per service, use one LB for *all* services.
* **Host-Based:** `api.myapp.com` -> API Target Group. `admin.myapp.com` -> Admin Target Group.
* **Path-Based:** `myapp.com/users` -> Users Service. `myapp.com/cart` -> Cart Service.
* **Query String:** `?version=beta` -> Beta Service (Canary testing).

### 2. Target Groups (The Destinations)
An ALB doesn't point to a "Server." It points to a **Target Group**.
* **Instance Type:** Points to EC2 Instance IDs.
* **IP Type:** Points to Private IPs (Critical for connecting to on-prem via VPN).
* **Lambda Type:** Points to a Serverless Function (L7 translation to JSON event).

---

## 🚦 Part 3: Advanced Traffic Management

### 1. Connection Draining (Deregistration Delay)
**Scenario:** You want to update an instance. You detach it from the ELB.
* **Without Draining:** The ELB cuts the connection immediately. User sees `502 Bad Gateway`.
* **With Draining (Default 300s):** The ELB stops sending *new* requests, but keeps the connection open for 300s to allow *in-flight* requests to finish.
* **Senior Tip:** Set this to `30s` for fast REST APIs, but `300s` for file upload servers.

### 2. Sticky Sessions (Session Affinity)
**Scenario:** A legacy app stores user sessions in local RAM (Java Heap).
* **Mechanism:** ALB generates a cookie (`AWSALB`).
* **Behavior:** The client sends this cookie, and ALB routes them to the *exact same EC2 instance* every time.
* **Risk:** Creates "Hot Nodes." If one node has all the active users and crashes, those users are logged out. **Prefer Stateless (Redis) sessions.**

### 3. Cross-Zone Load Balancing
* **Scenario:** Zone A has 2 targets. Zone B has 8 targets.
* **Disabled:** Traffic is split 50/50 between Zones.
  * Zone A targets get 25% load each.
  * Zone B targets get 6.25% load each. (Imbalanced).
* **Enabled (Default on ALB):** Traffic is split evenly across *all 10 targets* regardless of zone (10% each).
  * **Cost Warning:** Data transfer between zones costs money.

---

## 🚀 Part 4: Scaling & Performance (Pre-Warming)

ELBs are not infinite magic. They are software running on EC2 instances managed by AWS.

### 1. The Scaling Mechanism
* When traffic spikes, AWS scales the ELB (adds more nodes/IPs in the background).
* **The Lag:** This takes minutes.

### 2. The "Pre-Warming" Request
* **Scenario:** You are launching a Super Bowl ad. Traffic will go from 0 to 1 Million in 1 second.
* **Problem:** The ELB will crash (503 Service Unavailable) because it can't scale that fast.
* **Solution:** Open a support ticket with AWS for **ELB Pre-Warming**. They will provision capacity ahead of time.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Static IP" Requirement
**Interviewer:** *"A corporate client needs to whitelist our API IP in their firewall. We are using an ALB."*

* **The Problem:** ALBs change IPs dynamically.
* ✅ **Senior Answer:** "**Put an NLB in front of the ALB.**"
  * **Architecture:** Client -> NLB (Static Elastic IP) -> ALB (Smart Routing) -> EC2.
  * Use the NLB's static IP for the firewall whitelist.
  * Configure the ALB as the Target Group for the NLB (Using ALB-type Target Group).

### Scenario B: 502 Bad Gateway Debugging
**Interviewer:** *"Users are seeing random 502 errors on the ALB."*

* ✅ **Senior Answer:** Check the **Keep-Alive Timeout**.
  * **Rule:** The target (Node.js/Nginx) keep-alive timeout must be **longer** than the ALB idle timeout.
  * **Why:** If Node.js closes the TCP connection silently while the ALB thinks it's open, the ALB sends a request to a closed socket -> 502.

### Scenario C: Handling Spikes (Thundering Herd)
**Interviewer:** *"10,000 users disconnect and reconnect instantly (e.g., after a soccer match ends)."*

* ✅ **Senior Answer:** "**Add Jitter.**"
  * The ALB might queue requests (Surge Queue), but if that fills, it drops packets (Spillover).
  * We cannot scale the backend instantly.
  * **Fix:** Implementing exponential backoff with **Jitter** on the client-side retry logic is the only way to save the infrastructure.

### Scenario D: Cost Optimization
**Interviewer:** *"Our Data Transfer bill is huge. We use Cross-Zone Load Balancing."*

* ✅ **Senior Answer:** "**Disable Cross-Zone (if safe).**"
  * If you have a perfectly even fleet (e.g., 5 servers in Zone A, 5 in Zone B), disabling Cross-Zone LB ensures traffic stays within the Zone.
  * **Result:** Intra-region data transfer is free (in many cases) or cheaper than Cross-Zone.
  * **Trade-off:** If Zone A fails, Zone B takes 100% load instantly, risking overload.

---

### **Final Checklist**
1.  **Selection:** NLB for Speed/Static IP. ALB for Routing/WAF.
2.  **Operations:** Connection Draining is mandatory for zero-downtime.
3.  **Scale:** Pre-warm ELBs before marketing events.
4.  **Security:** Offload SSL at the ALB to save CPU on web servers.

**This concludes the ELB Playbook.**