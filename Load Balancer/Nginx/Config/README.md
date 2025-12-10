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