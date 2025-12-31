# System Design: E-Commerce Platform (Amazon Scale)

## 1. Requirements & Scope

### Functional Requirements
* **Product Discovery:** Users can search and find products based on titles or names.
* **Product Details:** Users can view descriptions, prices, and images.
* **Cart Management:** Users can select quantities and manage items in a persistent cart.
* **Checkout & Payment:** Secure transaction processing and order placement.
* **Order Tracking:** Users can check the real-time status of their orders.
* **Inventory Management:** Manage purchase of items with limited stock to prevent overselling.

### Non-Functional Requirements (NFRs)
* **Scale:** Support 10M Monthly Active Users (MAU).
* **Availability:** High availability for searching and browsing.
* **Consistency:** High consistency for Payment and Inventory management (ACID compliance).
* **Performance:** Low latency (~200ms) for core user flows.

---

## 2. Back-of-the-Envelope Calculations

### Traffic Estimates
* **Daily Active Users (DAU):** ~500k to 1M (assuming 5-10% of MAU).
* **Average Order Rate:** 10 orders/sec (as specified in requirements).
* **Peak Traffic:** During sales, traffic can spike 10x-50x (500 orders/sec).
* **Read/Write Ratio:** Browsing to Ordering ratio is roughly 100:1.
    * **Search/Product Views:** ~1,000 to 5,000 QPS.

### Storage Estimates (5-Year Plan)
* **Product Catalog:** 1M products × 10KB metadata = 10GB.
* **Media (S3):** 1M products × 5 images × 200KB = 1TB.
* **Orders:** 10 orders/sec × 86,400s × 365 days × 5 years ≈ 1.5 Billion orders.
    * 1.5B orders × 2KB/row = **3TB of Order History Data**.

### Bandwidth Estimates
* **Egress:** 1,000 QPS × (10KB metadata + 100KB image) ≈ **110 MB/s**.
    * This justifies the use of a **CDN** to offload traffic from the origin servers.

---

## 3. High-Level System Architecture



<img width="2098" height="1576" alt="Image" src="https://github.com/user-attachments/assets/c384e0e8-b1ff-4586-8bfa-c0136bc12a4c" />


### Core Services
* **API Gateway:** Handles routing, authentication, and rate limiting.
* **Search Service:** Uses **ElasticSearch** for high-performance text searching.
* **Product Service:** Manages product metadata using **MongoDB** (for schema flexibility).
* **Inventory Service:** Acts as the source of truth for stock using **Postgres** and **Redis**.
* **Payment Service:** Interfaces with external Payment Gateways and ensures idempotency.
* **Notification Service:** Asynchronous service using **Kafka** to send emails/SMS.

---

## 4. Database Schema & Data Modeling

| Entity | DB Type | Key Attributes |
| :--- | :--- | :--- |
| **User** | MySQL | `userId`, `name`, `email`, `password_hash`, `address_list[]` |
| **Product** | MongoDB | `productId`, `name`, `description`, `price`, `qty`, `blob_urls[]` |
| **Cart** | Postgres | `cartId`, `userId`, `items: [{productId, qty}]` |
| **Inventory**| Postgres | `productId (indexed)`, `qty (source of truth)` |
| **Order** | Postgres | `orderId`, `userId`, `items[]`, `total`, `paymentId`, `status` |

---

## 5. Redis Key-Value Design
Redis is used for caching and distributed locking to maintain consistency in high-concurrency scenarios.

### Example 1: Inventory Write-Through Cache
* **Key:** `stock:{productId}`
* **Value:** `integer` (Current available quantity)
* **Example:** `SET stock:p99 45`

### Example 2: Distributed Lock (Flash Sale)
* **Key:** `lock:product:{productId}`
* **Value:** `request_uuid` (Expiry: 2 seconds)
* **Example:** `SET lock:product:p99 uuid_123 NX PX 2000`
* **Purpose:** Ensures only one checkout process decrements the stock for a specific item at a time.

---

## 6. System Design Interview Q&A (15-20 Questions)

1. **Q: How do you handle the "Double Spending" or "Overselling" problem?**
    * **A:** We use a **Redis Distributed Lock** combined with a database transaction in the Inventory Service. We check stock in Redis first; if available, we lock the item, decrement stock in the DB, and update Redis.

2. **Q: Why use ElasticSearch instead of just querying the Product DB?**
    * **A:** Standard databases are slow at "Fuzzy" or text-based searching. ElasticSearch uses an inverted index to provide sub-second search results for 1M+ products.

3. **Q: How do you sync data between MongoDB (Product DB) and ElasticSearch?**
    * **A:** We use **CDC (Change Data Capture)**. Any write to MongoDB triggers an event that is pushed to ElasticSearch to keep the search index eventually consistent.

4. **Q: What is the benefit of using Kafka in the Payment flow?**
    * **A:** It decouples the payment from downstream tasks. Once payment is successful, we produce a "PaymentConfirmed" event. The Order, Inventory, and Notification services consume this event independently.

5. **Q: How do you achieve 200ms latency for a global user base?**
    * **A:** By using a **CDN** to cache static images and metadata at edge locations and **Redis** to cache hot product details.

6. **Q: How do you handle a failure in the Payment Gateway?**
    * **A:** We implement a retry mechanism with exponential backoff and provide a "Pending" status to the user to avoid blocking the UI.

7. **Q: Why use Postgres for Orders instead of NoSQL?**
    * **A:** Orders require **ACID** properties. We cannot afford to lose an order record or have an inconsistent state between payment and order status.

8. **Q: How do you handle a scenario where the Redis cache goes down?**
    * **A:** The Inventory Service falls back to the Postgres DB (Source of Truth). We use DB-level pessimistic locking (`SELECT FOR UPDATE`) to maintain consistency until Redis is back.

9. **Q: What happens if a user adds an item to the cart and the price changes?**
    * **A:** As shown in the diagram, the Cart service fetches the latest price. During Checkout, a final price validation is performed before the payment request is created.

10. **Q: How do you ensure the system is "Highly Available"?**
    * **A:** Every service is deployed in multiple availability zones (AZs) behind a Load Balancer. Databases use Primary-Replica setups with auto-failover.

11. **Q: What is "Rate Limiting" in the API Gateway?**
    * **A:** It prevents DDoS attacks and API abuse by limiting the number of requests a single User ID or IP can make per second.

12. **Q: How do you manage limited stock during a Flash Sale?**
    * **A:** We use **Redis atomic decrements** (`DECR`). Since Redis is single-threaded, it handles high-concurrency stock deductions safely and much faster than a relational DB.

13. **Q: Why is there a "Notification Service" separate from the Order Service?**
    * **A:** Sending emails or SMS can be slow or fail. By separating it, we ensure the Order process isn't delayed by third-party mail server latency.

14. **Q: How do you handle "Idempotency" in payments?**
    * **A:** The client sends a unique `idempotency_key`. The Payment service stores this key; if a duplicate request arrives with the same key, it returns the previous result instead of charging again.

15. **Q: How do you optimize database performance for millions of orders?**
    * **A:** We use **Database Sharding** based on `userId` and implement an archival strategy to move orders older than 2 years to a "Cold Storage" data warehouse.

16. **Q: Why is the Inventory DB marked as a potential SPOF (Single Point of Failure)?**
    * **A:** Because every checkout depends on it. To mitigate this, we use the Redis "Write-through" strategy and DB replication as shown in the LLD.

17. **Q: What is the role of the "Order Consumer" in the diagram?**
    * **A:** It listens to the Kafka "Payment" topic. When a payment success event appears, it updates the `OrderDB` status from `PENDING` to `PAID`.

18. **Q: How do you handle global product availability (Multi-region)?**
    * **A:** We use Geo-sharding. Users in Asia are directed to Asian data centers via the API Gateway to minimize network hop latency.

19. **Q: Why use S3 for images instead of storing them in the DB?**
    * **A:** Databases are not optimized for large binary objects (Blobs). S3 is cheaper, scales infinitely, and integrates perfectly with CDNs.

20. **Q: How would you track "Trending Products"?**
    * **A:** We can stream "Product View" events to a stream processing engine like Apache Flink to calculate the most viewed items in the last hour and update a "Trending" Redis sorted set.