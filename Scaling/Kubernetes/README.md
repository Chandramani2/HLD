# ☸️ The Senior Kubernetes (K8s) Playbook

> **Target Audience:** Senior DevOps, SREs, & Platform Engineers  
> **Goal:** Move beyond "deploying pods" to mastering **Architecture**, **Networking Internals**, and **Day-2 Operations**.

In a senior interview, you must view Kubernetes not as a tool, but as a **Distributed Operating System** for the cloud. It is a declarative system based on the **Reconciliation Loop** (Desired State vs. Actual State).

---

## 📖 Table of Contents
1. [Part 1: The Architecture (Brain vs. Muscle)](#-part-1-the-architecture-brain-vs-muscle)
2. [Part 2: Core Primitives (Beyond Pods)](#-part-2-core-primitives-beyond-pods)
3. [Part 3: Networking ( The Hardest Part)](#-part-3-networking-the-hardest-part)
4. [Part 4: Storage & State (PVCs & StatefulSets)](#-part-4-storage--state-pvcs--statefulsets)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🧠 Part 1: The Architecture (Brain vs. Muscle)

K8s is a master-slave architecture. Knowing which component fails is critical for debugging.

### 1. The Control Plane (Master Nodes) - The Brain
* **API Server:** The frontend. The *only* component that talks to Etcd. All generic tools (`kubectl`) talk to this.
* **Etcd:** The generic **Database**. Key-Value store. Holds the entire state of the cluster (Secrets, Configs, Pod locations). **Critical:** If Etcd dies, the cluster is brain-dead.
* **Scheduler:** The **Matchmaker**. Decides *where* a pod should run (based on RAM/CPU, Taints/Tolerations). It does not start the pod; it just assigns a NodeName.
* **Controller Manager:** The **Reconciler**. A loop that constantly checks "Is the current state == desired state?" If 2 pods die, it starts 2 new ones.

### 2. The Worker Nodes - The Muscle
* **Kubelet:** The **Agent**. Reads the specs from the API Server and tells the Container Runtime (Docker/Containerd) to start the container.
* **Kube-Proxy:** The **Networker**. Maintains network rules (IP tables/IPVS) on the node to allow network communication between Pods and Services.



---

## 📦 Part 2: Core Primitives (Beyond Pods)

### 1. Pod (The Atomic Unit)
* Containers share the same **Network Namespace** (Same IP, `localhost` access) and **Volume mounts**.
* **Senior Note:** Pods are ephemeral (Cattle, not Pets). Never treat a Pod IP as static.

### 2. Deployment (Stateless)
* Manages **ReplicaSets**.
* Handles **Rolling Updates**. (Spins up new version, waits for health check, drains old version).
* **Use Case:** Web Servers, APIs.

### 3. StatefulSet (Stateful)
* **Unique Identity:** Pods get sticky names (`db-0`, `db-1`, `db-2`) instead of random hashes.
* **Ordered Deployment:** `db-1` won't start until `db-0` is healthy.
* **Stable Storage:** If `db-0` restarts on a different node, it re-attaches to the *same* disk (PVC).
* **Use Case:** Databases (Postgres, Cassandra), Kafka.

### 4. DaemonSet (System)
* Ensures **one copy** of the pod runs on **every single node**.
* **Use Case:** Log Collectors (Fluentd), Monitoring Agents (Prometheus Node Exporter).

---

## 🌐 Part 3: Networking (The Hardest Part)

How do pods talk to each other?

### 1. The Service (Stable IP)
Pods die; IPs change. Services provides a static VIP (Virtual IP).

* **ClusterIP (Default):** Internal only. Only reachable from inside the cluster.
* **NodePort:** Opens a specific port (30000-32767) on **every Worker Node**. External traffic hits `NodeIP:Port`.
* **LoadBalancer:** Asks the Cloud Provider (AWS/GCP) to provision a physical Load Balancer that points to the NodePorts.



[Image of Kubernetes Service Types Diagram]


### 2. Ingress (Layer 7 Routing)
* **Problem:** Using `LoadBalancer` for every service is expensive ($$$) and wastes IPs.
* **Solution:** **Ingress Controller** (Nginx/Traefik).
    * One Load Balancer sits at the edge.
    * Routes traffic based on Host/Path (`api.com/cart` -> Cart Service, `api.com/user` -> User Service).
    * Handles SSL Termination.

---

## 💾 Part 4: Storage & State (PVCs & StatefulSets)

Docker Volumes die with the container. K8s decouples storage from compute.

### 1. PV (Persistent Volume)
* The actual piece of storage (AWS EBS, Google Disk, NFS). It is a cluster resource.

### 2. PVC (Persistent Volume Claim)
* The "Ticket" a developer creates to ask for storage.
* **Process:**
    1.  Dev creates PVC ("I need 10GB").
    2.  K8s looks for a matching PV or provisions one dynamically (StorageClass).
    3.  Pod mounts the PVC.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: "My Pod is Pending"
**Interviewer:** *"I deployed a pod, but it stays in `Pending` state. Debug this."*

* ✅ **Senior Answer:** `Pending` means the **Scheduler** cannot find a suitable node.
    1.  **Resource Constraints:** Are requests (CPU/RAM) too high? Do we have enough capacity?
    2.  **Taints & Tolerations:** Did someone Taint the nodes (e.g., `NoSchedule` on GPU nodes) and the pod lacks the Toleration?
    3.  **Affinity Rules:** Is the pod asking to run on a specific node that doesn't exist?
    * *Action:* `kubectl describe pod <name>` and look at "Events".

### Scenario B: Zero Downtime Deployments
**Interviewer:** *"We deploy 10 times a day. Users see 502 errors during deployment."*

* ✅ **Senior Answer:** "**Configure Readiness Probes & RollingUpdate Strategy.**"
    * **RollingUpdate:** Default K8s strategy. But if the app takes 10s to boot, K8s might kill the old pod too early.
    * **Readiness Probe:** Tells K8s "I am running, but I am NOT ready for traffic yet."
    * K8s will not send traffic to the new pod until the Probe passes (e.g., HTTP 200 on `/health`).
    * K8s will not kill the old pod until the new pod is Ready.

### Scenario C: "CrashLoopBackOff"
**Interviewer:** *"A pod starts, crashes, restarts, crashes... Why?"*

* ✅ **Senior Answer:**
    1.  **Application Panic:** Code exception on startup. Check `kubectl logs`.
    2.  **Misconfiguration:** Missing Environment Variable or Secret.
    3.  **Liveness Probe Fail:** The app starts, but the Liveness Probe (health check) fails repeatedly, causing K8s to kill and restart it.

### Scenario D: Databases in K8s (Do or Don't?)
**Interviewer:** *"Should we run our production PostgreSQL in Kubernetes?"*

* ✅ **Senior Answer:** "**It depends (The Maturity Model).**"
    * **Day 1:** No. Use managed RDS/CloudSQL. The operational overhead of managing backups, failover, and replication in K8s is high.
    * **Day 2 (Advanced):** Yes, if using an **Operator** (e.g., Zalando Postgres Operator or Strimzi for Kafka). Operators automate the complex logic of "Promote Slave to Master" that a simple StatefulSet cannot handle.

### Scenario E: Security (Namespace Isolation)
**Interviewer:** *"Dev Team A and Dev Team B share a cluster. How do we stop Team A from deleting Team B's database?"*

* ✅ **Senior Answer:** "**RBAC (Role Based Access Control) + Network Policies.**"
    1.  **RBAC:** Create separate **Namespaces**. Give Team A `RoleBinding` only for `Namespace-A`. They literally cannot see `Namespace-B`.
    2.  **Network Policies:** By default, all pods can talk to all pods. Apply a Network Policy to `Namespace-B` that says "Deny Ingress from Namespace-A".

---

### **Final Checklist**
1.  **Architecture:** Control Plane vs. Data Plane.
2.  **Networking:** Service Types & Ingress.
3.  **Scaling:** HPA (Horizontal) vs. VPA (Vertical).
4.  **Debugging:** Describe Pod, Logs, Probes.
