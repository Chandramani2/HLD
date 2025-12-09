# 🛑 Rate Limiting Architectures: The Implementation Guide

> **Goal:** Implement Rate Limiting at three distinct layers:
> 1.  **Application Side:** Custom Middleware with Redis (Node.js).
> 2.  **API Gateway:** Edge protection using Nginx (Leaky Bucket).
> 3.  **Sidecar Mesh:** Distributed proxying using Envoy.

---

## 🏗️ Part 1: Application Side Implementation
**Strategy:** Distributed Middleware using Redis.
**Best For:** Complex business logic (e.g., "Premium users get 1000 req/s, Free users get 10 req/s").

### 1. Project Structure
```text
app-limiter/
├── docker-compose.yml      # Spins up Redis
├── src/
│   ├── middleware/
│   │   └── rateLimiter.js  # The Core Logic
│   ├── server.js           # Entry point
│   └── utils/
│       └── redisClient.js  # Redis connection
└── package.json
```

### 2. The Implementation Code

**`src/utils/redisClient.js`**
```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });
module.exports = redis;
```

**`src/middleware/rateLimiter.js` (The Fixed Window Counter)**
```javascript
const redis = require('../utils/redisClient');

const rateLimiter = (limit, windowSeconds) => {
  return async (req, res, next) => {
    const ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress;
    // Senior Tip: Use a composite key for granularity
    const key = `rate_limit:${ip}`;

    try {
      // Atomic Increment
      const requests = await redis.incr(key);

      // Set Expiry on the first request only
      if (requests === 1) {
        await redis.expire(key, windowSeconds);
      }

      // Check Limit
      if (requests > limit) {
        return res.status(429).json({
          error: "Too Many Requests",
          retryAfter: await redis.ttl(key)
        });
      }

      next();
    } catch (err) {
      console.error("Redis Error:", err);
      // Fail Open: If Redis is down, allow traffic (Resiliency)
      next();
    }
  };
};

module.exports = rateLimiter;
```

**`src/server.js`**
```javascript
const express = require('express');
const rateLimiter = require('./middleware/rateLimiter');
const app = express();

// Apply Limit: 10 requests per 60 seconds
app.use(rateLimiter(10, 60));

app.get('/', (req, res) => {
  res.send('Request Allowed!');
});

app.listen(3000, () => console.log('App running on port 3000'));
```

---

## 🚪 Part 2: API Gateway Implementation
**Strategy:** Nginx Zone Configuration.
**Best For:** DDoS protection, protecting legacy backends, IP-based blocking.

### 1. Project Structure
```text
gateway-limiter/
├── docker-compose.yml
├── nginx.conf              # The Limiter Logic
└── backend/                # Dummy service
    └── server.js
```

### 2. The Implementation Code (`nginx.conf`)



```nginx
events {}

http {
    # 1. Define the Limit Zone
    # $binary_remote_addr: Use 4 bytes per IP (Memory efficient)
    # zone=api_limit:10m: Allocate 10MB memory for keys
    # rate=5r/s: The steady leak rate (Leaky Bucket)
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

    upstream backend_servers {
        server backend:3000;
    }

    server {
        listen 80;

        location / {
            # 2. Apply the Limit
            # burst=10: Allow a temporary spike of 10 requests queueing up
            # nodelay: Process the burst instantly. If burst is exceeded -> 503.
            limit_req zone=api_limit burst=10 nodelay;

            # 3. Custom Error Handling
            limit_req_status 429;
            error_page 429 = @rate_limited;

            proxy_pass http://backend_servers;
        }

        location @rate_limited {
            default_type application/json;
            return 429 '{"error": "Too Many Requests - Gateway Rejection"}';
        }
    }
}
```

### 3. Docker Compose (`docker-compose.yml`)
```yaml
version: '3'
services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
    depends_on:
      - backend
  backend:
    image: node:18-alpine
    # Simple inline server for testing
    command: node -e 'require("http").createServer((req,res)=>res.end("OK")).listen(3000)'
```

---

## 🏎️ Part 3: Sidecar / Service Mesh Implementation
**Strategy:** Envoy Proxy (Local Rate Limit).
**Best For:** Microservices in Kubernetes. Decouples logic from the app language. The App thinks it's talking to the world, but it talks to `localhost` (Envoy).

### 1. Project Structure
```text
sidecar-limiter/
├── envoy.yaml              # The Sidecar Config
├── docker-compose.yml
└── service.js              # The Application
```

### 2. The Architecture
* **Traffic Flow:** `User` -> `Envoy (Container A)` -> `Service (Container B)`.
* They share the same network namespace in a Pod (simulated here via Docker Network).

### 3. The Envoy Configuration (`envoy.yaml`)
Envoy configuration is verbose. Here is a simplified **Token Bucket** setup.

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 } # Envoy listens here
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": [type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager](https://type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager)
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: service_cluster }
          http_filters:
          # --- RATE LIMIT FILTER START ---
          - name: envoy.filters.http.local_ratelimit
            typed_config:
              "@type": [type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit](https://type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit)
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 10
                tokens_per_fill: 2
                fill_interval: 1s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
          # --- RATE LIMIT FILTER END ---
          - name: envoy.filters.http.router
            typed_config:
              "@type": [type.googleapis.com/envoy.extensions.filters.http.router.v3.Router](https://type.googleapis.com/envoy.extensions.filters.http.router.v3.Router)

  clusters:
  - name: service_cluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: service_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: app_service
                port_value: 3000
```

---

## 🧠 Senior Summary: Which one to choose?

| Feature | Application Side | API Gateway (Nginx) | Sidecar (Envoy) |
| :--- | :--- | :--- | :--- |
| **Awareness** | **High.** Knows User ID, Subscription Plan, Data payload. | **Medium.** Knows IP, Headers, Path. | **Low/Medium.** Infrastructure level traffic shaping. |
| **Performance** | Slower (Node.js single thread + Redis Hop). | **Fastest** (C/C++ optimized). | **Fast** (C++ Proxy). |
| **Complexity** | Low. Standard code. | Medium. Config files. | **High.** YAML hell & K8s. |
| **Use Case** | Business Logic ("Gold Users get more"). | DDoS Protection ("Block IP"). | Service-to-Service ("Service A can't spam Service B"). |

# JAVA

# ☕ Spring Boot Rate Limiting: The Complete Guide

> **Goal:** Implement Rate Limiting in a Spring Boot application using various strategies ranging from "Do it yourself" to "Enterprise Libraries".
> **Stack:** Java 17+, Spring Boot 3.x, Redis.

---

## 📖 Strategies Overview

1.  **Manual Interceptor:** Best for learning internals. Uses raw Redis `INCR`.
2.  **AOP (Aspect Oriented Programming):** Best for clean code. Custom `@RateLimit` annotation.
3.  **Bucket4j:** Best for production algorithms (Token Bucket) and distributed clusters.
4.  **Resilience4j:** Best for microservices (Circuit Breakers + Rate Limiters combined).



---

## 🛠️ Method 1: The Manual Way (HandlerInterceptor + Redis)
*Using raw Redis atomic operations. Good for simple Fixed Window requirements.*

### 1. Project Setup
**`pom.xml`**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 2. The Interceptor Logic
**`src/main/java/com/app/interceptor/SimpleRateLimitInterceptor.java`**
```java
package com.app.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import java.util.concurrent.TimeUnit;

@Component
public class SimpleRateLimitInterceptor implements HandlerInterceptor {

    private final StringRedisTemplate redisTemplate;
    private static final int MAX_REQ = 10;
    private static final int WINDOW_SEC = 60;

    public SimpleRateLimitInterceptor(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String ip = request.getRemoteAddr();
        String key = "limit:" + ip;
        
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        
        // 1. Increment (Atomic)
        Long count = ops.increment(key);

        // 2. Set Expiry on first request
        if (count != null && count == 1) {
            redisTemplate.expire(key, WINDOW_SEC, TimeUnit.SECONDS);
        }

        // 3. Block if limit exceeded
        if (count != null && count > MAX_REQ) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            return false;
        }
        return true;
    }
}
```

### 3. Registration
**`src/main/java/com/app/config/WebConfig.java`**
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired SimpleRateLimitInterceptor interceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor).addPathPatterns("/api/**");
    }
}
```

---

## 🎨 Method 2: The AOP Way (Custom Annotation)
*Best for granular control. Allows you to annotate specific controllers with `@RateLimit(limit=5)`.*

### 1. The Annotation
**`src/main/java/com/app/annotation/RateLimit.java`**
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int limit() default 10;
    int time() default 60; // seconds
}
```

### 2. The Aspect
**`src/main/java/com/app/aspect/RateLimitAspect.java`**
```java
@Aspect
@Component
public class RateLimitAspect {

    @Autowired StringRedisTemplate redisTemplate;

    @Around("@annotation(rateLimit)") // Intercepts methods with the annotation
    public Object enforceLimit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        // Get IP from RequestContextHolder
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String ip = request.getRemoteAddr();
        String key = "limit:" + ip + ":" + joinPoint.getSignature().getName();

        Long count = redisTemplate.opsForValue().increment(key);
        
        if (count != null && count == 1) {
            redisTemplate.expire(key, rateLimit.time(), TimeUnit.SECONDS);
        }

        if (count != null && count > rateLimit.limit()) {
            throw new ResponseStatusException(HttpStatus.TOO_MANY_REQUESTS, "Slow down!");
        }

        return joinPoint.proceed(); // Continue to Controller
    }
}
```

### 3. Usage
```java
@RestController
public class ApiController {

    @RateLimit(limit = 5, time = 60)
    @GetMapping("/heavy-task")
    public String heavy() {
        return "Task Done";
    }
}
```

---

## 🪣 Method 3: The Production Way (Bucket4j)
*Uses the **Token Bucket** algorithm. Handles bursts better than fixed counters. Highly recommended for production.*

### 1. Dependencies
```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.1.0</version>
</dependency>
```

### 2. Implementation
**`src/main/java/com/app/service/RateLimitService.java`**
```java
@Service
public class RateLimitService {
    
    // Cache buckets in memory (ConcurrentHashMap)
    // For distributed systems, Bucket4j supports Redis/Hazelcast extensions
    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    public Bucket resolveBucket(String apiKey) {
        return cache.computeIfAbsent(apiKey, this::newBucket);
    }

    private Bucket newBucket(String apiKey) {
        // Capacity: 10 tokens. Refill: 10 tokens per 1 minute.
        Bandwidth limit = Bandwidth.classic(10, Refill.greedy(10, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }
}
```

### 3. Controller Usage
```java
@GetMapping("/bucket")
public ResponseEntity<String> bucketApi(@RequestHeader("X-API-KEY") String apiKey) {
    Bucket bucket = rateLimitService.resolveBucket(apiKey);

    if (bucket.tryConsume(1)) {
        return ResponseEntity.ok("Success");
    } else {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).body("Quota Exceeded");
    }
}
```

---

## 🛡️ Method 4: The Cloud-Native Way (Resilience4j)
*Best if you are already using Circuit Breakers. Config-driven.*

### 1. Dependencies
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 2. Configuration (`application.yml`)
No code needed for logic, just config!

```yaml
resilience4j:
  ratelimiter:
    instances:
      backendA:
        limitForPeriod: 5
        limitRefreshPeriod: 60s
        timeoutDuration: 0s # Fail immediately if limit reached
```

### 3. Usage
```java
@RestController
public class ResilienceController {

    @RateLimiter(name = "backendA", fallbackMethod = "fallback")
    @GetMapping("/resilience")
    public String api() {
        return "Success";
    }

    // Optional Fallback
    public String fallback(RequestNotPermitted exception) {
        return "Too many requests - Resilience4j blocked you.";
    }
}
```

---

## 📊 Comparison Summary

| Method | Algorithm | Complexity | Best Use Case |
| :--- | :--- | :--- | :--- |
| **Manual (Interceptor)** | Fixed Window | Low | Learning; Simple IP blocking. |
| **AOP (Annotation)** | Fixed Window | Medium | Clean code; Per-endpoint limits. |
| **Bucket4j** | **Token Bucket** | Medium/High | **Production Standard**; Smooth traffic shaping. |
| **Resilience4j** | Token/Leaky | Medium | **Microservices**; if using Circuit Breakers already. |

**Next Step:** If you choose **Bucket4j** for a distributed system (multiple Spring instances), you **must** use the `bucket4j-redis` extension, otherwise, the limits will be local to each server (e.g., 5 servers = 5x limit).