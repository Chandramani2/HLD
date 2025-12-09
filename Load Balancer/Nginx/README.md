# 🟩 The Senior Nginx Playbook

> **Target Audience:** DevOps, SREs, & System Architects  
> **Goal:** Master **Reverse Proxying**, **Load Balancing**, **Caching Strategies**, and **Performance Tuning**.

In a senior interview, you must explain why Nginx uses **Event-Driven Architecture** instead of Threads (Apache), how to debug **Regex Priority** in location blocks, and how to handle **The C10k Problem**.

Nginx is not just a web server. It is the Edge Layer. You must demonstrate knowledge of Event Loops (epoll), Location Block Priorities, Reverse Proxying, and Traffic Shaping

---

## 📖 Table of Contents
1. [Part 1: Architecture (Process Model & Epoll)](#-part-1-architecture-process-model--epoll)
2. [Part 2: Configuration Mastery (Location Priority)](#-part-2-configuration-mastery-location-priority)
3. [Part 3: Performance Tuning (Buffers & Keepalive)](#-part-3-performance-tuning-buffers--keepalive)
4. [Part 4: Nginx as a Load Balancer (L4 vs L7)](#-part-4-nginx-as-a-load-balancer-l4-vs-l7)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## ⚙️ Part 1: Architecture (Process Model & Epoll)

Why can Nginx handle 10,000 connections with 5MB of RAM, while Apache crashes?

### 1. The Master-Worker Model
* **Master Process:** Reads config, binds to ports, manages workers. (Runs as `root`).
* **Worker Processes:** Do the actual work. (Run as `www-data`).
* **Senior Config:** `worker_processes auto;` (Spawns 1 worker per CPU core).

### 2. Event-Driven vs. Threaded
* **Apache (Prefork/Worker):** 1 Connection = 1 Thread/Process.
  * *Blocking:* If the DB is slow, the Thread waits. RAM usage explodes.
* **Nginx (Event Loop):** 1 Worker = Thousands of Connections.
  * *Non-Blocking:* Uses **Epoll** (Linux) or **Kqueue** (BSD).
  * "I have a new request. Send it to PHP-FPM. While waiting, I will handle the next request."
  * **Result:** Constant RAM usage regardless of load.



---

## 📝 Part 2: Configuration Mastery (Location Priority)

The #1 source of Nginx bugs is overlapping `location` blocks.

### 1. The Priority Rule
Nginx does **NOT** match top-to-bottom. It follows a strict logic:

1.  **Exact Match (`=`)**: Stops searching.
2.  **Longest Prefix Match**: Remembers the best match, but keeps searching Regex.
3.  **Regex Match (`~` or `~*`)**: First defined Regex wins. Overrides Prefix match.
4.  **Best Prefix**: If no Regex matches, use the remembered Prefix.

### 2. The "Slash" Trap (`root` vs `alias`)
* **`root`:** Appends the location to the path.
  * `location /static/ { root /var/www; }`
  * Request `/static/img.png` -> File `/var/www/static/img.png`.
* **`alias`:** Replaces the location with the path.
  * `location /static/ { alias /var/www/images/; }`
  * Request `/static/img.png` -> File `/var/www/images/img.png`.

---

## 🚀 Part 3: Performance Tuning (Buffers & Keepalive)

Default Nginx configs are meant for compatibility, not speed.

### 1. Upstream Keepalive (Critical for Microservices)
By default, Nginx opens a **new TCP connection** to the backend (Node/Go/Python) for every request. This exhausts ephemeral ports.
* **Fix:** Enable `keepalive` in the upstream block.

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1; # Required for keepalive
    proxy_set_header Connection "";
}
```

### 2. Buffering (The "Slowloris" Defense)
* **Client Body Buffer:** If a user uploads a 1GB file, Nginx buffers it to disk *before* sending it to the backend. This protects your Node.js app from slow clients holding connections open.
* **Sendfile:** `sendfile on;`. Allows the Kernel to copy data from Disk -> Network Card directly (Zero Copy), bypassing the CPU.

### 3. File Descriptors (The Limit)
* **Error:** `24: Too many open files`.
* **Fix:**
  * OS Level: `ulimit -n 65535`
  * Nginx Level: `worker_rlimit_nofile 65535;`

---

## ⚖️ Part 4: Nginx as a Load Balancer (L4 vs L7)

Nginx is a robust Load Balancer.

### 1. Layer 7 (HTTP) - The `http` Context
Can route based on Headers, Cookies, URL.

```nginx
upstream my_app {
    least_conn; # Algorithm: Least Connections
    server app1:3000 weight=3;
    server app2:3000;
}
```

### 2. Layer 4 (TCP/UDP) - The `stream` Context
Can load balance MySQL, Redis, or DNS traffic.

```nginx
stream {
    upstream db_cluster {
        server db1:3306;
        server db2:3306;
    }
    server {
        listen 3306;
        proxy_pass db_cluster;
    }
}
```

### 3. Passive Health Checks
Nginx (Open Source) doesn't have an `/health` probe. It watches traffic.
* `server app1:3000 max_fails=3 fail_timeout=30s;`
* If app1 fails 3 times, Nginx marks it dead for 30s.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Thundering Herd" (Caching)
**Interviewer:** *"We use Nginx for caching. The cache expires. 1,000 users hit the URL instantly. All 1,000 requests hit the backend DB. The DB dies."*

* ✅ **Senior Answer:** "**Use `proxy_cache_lock`.**"
  * Configuration: `proxy_cache_lock on;`
  * **Mechanism:** When a cache miss occurs, Nginx allows **only one** request to pass to the backend to populate the cache.
  * The other 999 requests wait (hang) until the first one returns and populates the cache.
  * Then, the 999 are served from the cache.

### Scenario B: 502 Bad Gateway vs 504 Gateway Timeout
**Interviewer:** *"What is the difference between a 502 and a 504?"*

* ✅ **Senior Answer:**
  * **502 (Bad Gateway):** The upstream (e.g., Node.js) refused the connection. The process is likely crashed or not listening.
  * **504 (Gateway Timeout):** The upstream accepted the connection but took too long to reply (e.g., Slow SQL query).
  * **Why it matters:** 502 = Check Process Manager (PM2). 504 = Check Database/Code Performance.

### Scenario C: Rate Limiting (DDoS Protection)
**Interviewer:** *"How do we limit API calls to 10 requests per second per IP?"*

* ✅ **Senior Answer:** "**The Leaky Bucket Algorithm.**"

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=mylimit burst=20 nodelay;
    }
}
```

* `burst=20`: Allows a spike of 20 requests.
* `nodelay`: Processes the burst instantly (instead of queuing them).
* **$binary_remote_addr**: Uses 4 bytes per IP (IPv4) instead of string storage (efficient RAM usage).

### Scenario D: Zero Downtime Reloads
**Interviewer:** *"How does `nginx -s reload` work without dropping connections?"*

* ✅ **Senior Answer:**
  1.  Master Process checks syntax and forks **new** Worker processes with the new config.
  2.  Master sends a signal to **old** Workers: "Stop accepting new connections."
  3.  Old Workers finish their *current* requests and then exit gracefully.
  4.  New Workers handle all new traffic.

---

### **Final Checklist**
1.  **Architecture:** Master/Worker & Epoll.
2.  **Config:** `keepalive` to upstream & `worker_rlimit_nofile`.
3.  **Routing:** Regex `~` vs Prefix `/` priority.
4.  **Stability:** `proxy_cache_lock` & Rate Limiting.

**This concludes the Nginx Playbook.**