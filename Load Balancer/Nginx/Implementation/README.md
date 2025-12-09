# 🔌 The Senior Nginx Implementation Playbook

> **Target Audience:** DevOps Engineers & System Architects  
> **Goal:** Master **Architectural Placement**, **Container Networking**, and **Reverse Proxy Configuration**.

In a senior interview, the question "How does Nginx connect to my app?" is about **Service Discovery** and **Protocol Translation**. You must explain how Nginx finds your service in a dynamic environment (Docker/Kubernetes).

---

## 📖 Table of Contents
1. [Part 1: Where to Place Nginx? (The 3 Patterns)](#-part-1-where-to-place-nginx-the-3-patterns)
2. [Part 2: The Connection Logic (Upstreams & DNS)](#-part-2-the-connection-logic-upstreams--dns)
3. [Part 3: Implementation Code (Docker Compose & Config)](#-part-3-implementation-code-docker-compose--config)
4. [Part 4: Connecting to Python/PHP (uWSGI vs FastCGI)](#-part-4-connecting-to-pythonphp-uwsgi-vs-fastcgi)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🗺️ Part 1: Where to Place Nginx? (The 3 Patterns)

Where does the binary actually run?

### 1. The Edge Gateway (Standard VM/Docker)
* **Location:** A dedicated server (or container) at the edge of your Public Subnet.
* **Flow:** `User` -> `Internet` -> `Nginx (Public IP)` -> `Private Network` -> `App Servers`.
* **Role:** SSL Termination, DDoS protection, Static File serving.

### 2. The Ingress Controller (Kubernetes)
* **Location:** Runs as a **Pod** inside the cluster, exposed via a Cloud Load Balancer.
* **Flow:** `User` -> `AWS NLB` -> `Nginx Ingress Pod` -> `App Service` -> `App Pod`.
* **Role:** Routing traffic based on Hostname (`api.site.com` vs `web.site.com`) to different internal services.

### 3. The Sidecar (Legacy / Specific)
* **Location:** Installed on the *same* machine (localhost) as the application.
* **Flow:** `User` -> `Nginx (Port 80)` -> `App (Port 3000)`.
* **Role:** Translating FastCGI (PHP) to HTTP or handling local buffering.



---

## 🔗 Part 2: The Connection Logic (Upstreams & DNS)

How does Nginx know where your app is?

### 1. The "Upstream" Block
Nginx acts as an **HTTP Client**. It opens a TCP connection to your backend.
* You define a pool of servers called an `upstream`.
* Nginx load balances between them.

### 2. Docker DNS (The Magic)
In Docker Compose or Kubernetes, you don't use IPs (e.g., `172.18.0.4`). You use **Service Names**.
* If your Node.js container is named `my-backend`.
* Docker's internal DNS resolves `ping my-backend` to the correct internal IP.
* Nginx config uses `http://my-backend:3000`.

---

## 💻 Part 3: Implementation Code (Docker Compose & Config)

Let's build a production-ready setup where Nginx sits in front of a Node.js API.

### Step 1: The Infrastructure (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  # 1. The Backend Service
  api_service:
    image: my-node-app
    container_name: backend_container
    # Senior Tip: Do NOT expose ports (3000:3000) to the host.
    # Keep it private inside the Docker network.
    expose:
      - "3000"
    networks:
      - app_network

  # 2. The Nginx Gateway
  nginx_gateway:
    image: nginx:alpine
    container_name: web_server
    ports:
      - "80:80"   # Public HTTP
      - "443:443" # Public HTTPS
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api_service
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

### Step 2: The Configuration (`nginx.conf`)

```nginx
# 1. Define the Upstream
# 'api_service' matches the service name in docker-compose.
upstream backend_cluster {
    least_conn;  # Load Balancing Algorithm
    server api_service:3000; 
    keepalive 64; # Keep connection open to backend for speed
}

server {
    listen 80;
    server_name api.myapp.com;

    # 2. Security Headers (Senior Best Practice)
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    # 3. Routing Logic
    location / {
        # The Proxy Pass
        proxy_pass http://backend_cluster;

        # 4. Header Forwarding (Crucial!)
        # Without these, the App thinks every request comes from Nginx (Internal IP)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 5. Performance Tuning
        proxy_http_version 1.1; # Required for keepalive / WebSockets
        proxy_set_header Connection "";
    }
}
```

---

## 🐍 Part 4: Connecting to Python/PHP (uWSGI vs FastCGI)

Not everything speaks HTTP.

### 1. Python (Django/Flask)
* Python apps run behind a WSGI server (Gunicorn/uWSGI).
* **Option A (HTTP):** Gunicorn runs on port 8000. Nginx `proxy_pass http://python_app:8000`. (Easiest).
* **Option B (Socket):** Gunicorn creates a file `app.sock`. Nginx writes directly to the file. (Faster, requires shared volume).

**Nginx Config for Socket:**
```nginx
location / {
    include uwsgi_params;
    uwsgi_pass unix:/shared_volume/app.sock;
}
```

### 2. PHP (Laravel/Wordpress)
* PHP-FPM does **not** speak HTTP. It speaks the binary **FastCGI** protocol.
* You **cannot** use `proxy_pass`. You must use `fastcgi_pass`.

**Nginx Config for PHP:**
```nginx
location ~ \.php$ {
    # 'php_app' is the Docker service name, 9000 is default FPM port
    fastcgi_pass php_app:9000; 
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Localhost" Trap
**Interviewer:** *"I configured Nginx inside Docker to point to `proxy_pass http://localhost:3000`, but I get a 502."*

* ✅ **Senior Answer:**
    * "Inside a container, `localhost` refers to **the container itself**.
    * Nginx is looking for a Node app running *inside* the Nginx container, which doesn't exist.
    * **Fix:** Use the Docker Service Name (`http://api_service:3000`) or `host.docker.internal` (for Mac/Windows dev)."

### Scenario B: Client IP is Wrong
**Interviewer:** *"My backend logs show every request coming from `172.18.0.5` (The Nginx IP). I need the User's real IP."*

* ✅ **Senior Answer:**
    * Nginx creates a *new* connection to the backend. The source IP is Nginx.
    * **Fix:** Inspect the `X-Forwarded-For` header in the backend code.
    * `const userIP = req.headers['x-forwarded-for'] || req.socket.remoteAddress;`

### Scenario C: Handling WebSockets
**Interviewer:** *"My Socket.io connection fails through Nginx."*

* ✅ **Senior Answer:**
    * WebSockets require the connection to be "Upgraded" from HTTP 1.1 to a TCP stream.
    * Default Nginx drops the `Upgrade` header.
    * **Config Fix:**
    ```nginx
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    ```

### Scenario D: Static Assets (Performance)
**Interviewer:** *"Should requests for `/images/logo.png` go to the Node.js backend?"*

* ❌ **Bad Answer:** "Yes, let the app serve it." (Node is single-threaded; don't waste CPU sending files).
* ✅ **Senior Answer:** "**Offload to Nginx.**"
    ```nginx
    location /images/ {
        root /var/www/public;
        expires 30d; # Browser Caching
        access_log off;
    }
    ```
    * Mount the code volume to Nginx. Nginx serves files directly from disk (Zero Copy). The request never touches Node.js.

---

### **Final Checklist**
1.  **Placement:** Edge or Ingress.
2.  **Naming:** Use Docker Service Names, not IPs.
3.  **Headers:** Always pass `Host` and `X-Forwarded-For`.
4.  **Protocols:** `proxy_pass` for HTTP, `fastcgi_pass` for PHP.

**This concludes the Nginx Implementation Playbook.**