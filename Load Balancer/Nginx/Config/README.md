# ⚖️ Nginx Load Balancer Configuration

> **Goal:** Distribute incoming traffic across multiple backend servers to improve performance, reliability, and scalability.

To implement a load balancer in Nginx, you need to define an `upstream` group (your backend servers) and then configure a `server` block to proxy traffic to that group.

---

## 1. The Basic Configuration (Round Robin) 🔄

By default, Nginx uses **Round Robin**. It distributes requests sequentially (Server 1 -> Server 2 -> Server 3 -> Server 1...).

Open your Nginx config file (usually `/etc/nginx/nginx.conf` or `/etc/nginx/conf.d/default.conf`) and add this:

```nginx
http {
    # 1. Define the group of backend servers
    upstream backend_servers {
        server 10.0.0.1:8080;
        server 10.0.0.2:8080;
        server 10.0.0.3:8080;
    }

    server {
        listen 80;
        server_name yourdomain.com;

        location / {
            # 2. Forward traffic to the upstream group
            proxy_pass http://backend_servers;
            
            # 3. Essential Headers (Pass real client IP to backend)
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

---

## 2. Advanced Load Balancing Algorithms 🧠

If your servers have different specs or your app requires user sessions to stay on one server, use these directives inside the `upstream` block.

### A. Least Connections (`least_conn`)
Best for requests that take varying amounts of time. Nginx sends the request to the server with the fewest active connections.

```nginx
upstream backend_servers {
    least_conn;  # <--- Logic changer
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### B. IP Hash (Sticky Sessions) (`ip_hash`)
Best if your app stores session data locally on the server (stateful). It ensures a user with a specific IP always goes to the same server.

```nginx
upstream backend_servers {
    ip_hash;     # <--- Logic changer
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### C. Weighted Load Balancing
Best if Server 1 is powerful (e.g., 32GB RAM) and Server 2 is weak (e.g., 8GB RAM).

```nginx
upstream backend_servers {
    server 10.0.0.1:8080 weight=3; # Receives 3x more traffic
    server 10.0.0.2:8080 weight=1;
}
```

---

## 3. Handling Server Failures (Passive Health Checks) 🚑

Nginx Open Source performs "passive" health checks. If a request to a server fails, Nginx marks it as "down" for a specific time and tries the next server.

```nginx
upstream backend_servers {
    # If a server fails 3 times within 30 seconds, mark it down for 30 seconds.
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
    
    # Backup server: Only used if ALL other servers are down
    server 10.0.0.4:8080 backup; 
}
```

---

## 4. How to Apply the Changes 🚀

1.  **Check syntax:** Ensure there are no errors in your file.
    ```bash
    sudo nginx -t
    ```
2.  **Reload Nginx:** Apply changes without dropping connections.
    ```bash
    sudo systemctl reload nginx
    ```

---

## 📝 Summary of Directives

| Directive | Description | Use Case |
| :--- | :--- | :--- |
| `upstream` | Defines the pool of backend servers. | Mandatory for load balancing. |
| `proxy_pass` | Tells Nginx where to send the request. | Mandatory. |
| `least_conn` | Sends traffic to the idlest server. | Long-running tasks (video encoding, reports). |
| `ip_hash` | Maps User IP -> Specific Server. | Stateful apps (shopping carts saved on disk). |
| `max_fails` | Handling crashes. | Preventing traffic to dead servers. |

# 🔀 Nginx Load Balancing Algorithms: The Complete Guide

> **Overview:** Nginx offers several methods to distribute traffic. Choosing the right one depends on your application type (Stateful vs. Stateless) and your infrastructure (Equal vs. Unequal servers).

Below is the complete list of algorithms available in Nginx Open Source and Nginx Plus.

---

## 1. Standard Algorithms (Open Source) 🆓
These are available in the free version of Nginx and cover 95% of use cases.

### A. Round Robin (The Default) 🔄
**How it works:** Requests are distributed sequentially. Request 1 goes to Server A, Request 2 to Server B, Request 3 to Server C, then back to A.

**Best for:** Servers with identical specifications and stateless applications.

```nginx
upstream backend {
    # No directive needed; this is default
    server 10.0.0.1;
    server 10.0.0.2;
}
```

### B. Least Connections (`least_conn`) 📉
**How it works:** Sends the new request to the server with the **fewest active connections**.

**Best for:** Requests that take varying times to complete (e.g., file uploads, video encoding). It prevents a single server from getting bogged down by heavy tasks.

```nginx
upstream backend {
    least_conn;
    server 10.0.0.1;
    server 10.0.0.2;
}
```

### C. IP Hash (`ip_hash`) 📌
**How it works:** Calculates a hash based on the client's IP address. A specific user IP will **always** reach the same backend server (unless that server goes down).

**Best for:** "Sticky Sessions" where session data is stored locally on the server RAM.

```nginx
upstream backend {
    ip_hash;
    server 10.0.0.1;
    server 10.0.0.2;
}
```

### D. Generic Hash (`hash`) 🔑
**How it works:** A more flexible version of IP Hash. You can hash **any** variable (User-Agent, Cookie, Request URI).

**Best for:** Cache sharding. For example, hashing the `request_uri` ensures that the same image URL always goes to the same server (maximizing cache hits).

```nginx
upstream backend {
    # Hash based on the URL
    hash $request_uri consistent;
    server 10.0.0.1;
    server 10.0.0.2;
}
```
*Note: The `consistent` flag uses Consistent Hashing (Ring Hashing) to minimize reshuffling if a server is added/removed.*

### E. Random (`random`) 🎲
**How it works:** Picks a server at random.

**Best for:** Distributed systems where you have a very large number of upstream servers and the load is generally uniform.

```nginx
upstream backend {
    # Pick 2 random servers, then choose the one with least connections
    random two least_conn; 
    server 10.0.0.1;
    server 10.0.0.2;
    server 10.0.0.3;
    server 10.0.0.4;
}
```

---

## 2. Premium Algorithms (Nginx Plus Only) 💰
These require a paid subscription.

### A. Least Time (`least_time`) ⏱️
**How it works:** Calculates the average response time of each server and sends traffic to the fastest one.

**Best for:** Latency-sensitive applications where milliseconds matter.

```nginx
upstream backend {
    # header = Time to first byte
    # last_byte = Time to full response
    least_time header inflight;
    server 10.0.0.1;
    server 10.0.0.2;
}
```

---

## 3. The Power Modifiers (Can apply to ANY algorithm) ⚡

These are not algorithms themselves, but settings that tweak how the algorithms work.

### A. Weights (`weight`)
**Concept:** Server A is a supercomputer; Server B is a laptop. Give A more work.

```nginx
upstream backend {
    server 10.0.0.1 weight=3; # Takes 75% of traffic
    server 10.0.0.2 weight=1; # Takes 25% of traffic
}
```

### B. Slow Start (`slow_start`) (Plus Only)
**Concept:** When a server recovers from a crash, don't flood it instantly. Gradually ramp up traffic over time.

```nginx
upstream backend {
    server 10.0.0.1 slow_start=30s;
}
```

---

## 📝 Quick Comparison Table

| Algorithm | Type | Use Case |
| :--- | :--- | :--- |
| **Round Robin** | Static | Equal servers, simple apps. |
| **Least Conn** | Dynamic | Long-running tasks, unequal load. |
| **IP Hash** | Static | Session Persistence (Sticky). |
| **Generic Hash** | Static | Caching (URL), Header routing. |
| **Least Time** | Dynamic | High-performance (Paid only). |