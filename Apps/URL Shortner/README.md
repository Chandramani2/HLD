# System Design: Scalable URL Shortener (Distributed Approach)

## 1. Requirements & Scale
### Functional Requirements
* **Shorten URL:** Generate a unique, short alias for a long URL.
* **Redirection:** Redirect users from a short code to the original long URL via 301 (permanent) or 302 (temporary/analytics) redirects.
* **Custom Aliases:** Allow users to specify a unique custom string.
* **Expiration:** Support optional expiration times for links.

### Non-Functional Requirements (NFRs)
* **Scale:** Support 100M Daily Active Users (DAU) and 1 Billion total URLs.
* **Low Latency:** Redirection should be fast (~200ms).
* **High Availability:** Availability is prioritized over strict consistency.
* **Uniqueness:** Ensure no collisions for short codes.

### Back-of-the-Envelope Calculations
* **Write Traffic:** 100M / 10^5 seconds/day ≈ 1,000 requests per second (RPS).
* **Read Traffic:** Assuming a 10:1 Read/Write ratio = 10,000 RPS. Peak traffic can reach 100,000 RPS.
* **Storage:** Using Base62 (0-9, a-z, A-Z), a 6-character code provides $62^6 \approx 56$ Billion unique combinations.
* **Server Throughput:** One standard EC2 instance handles ~1,000 RPS; Read services must scale horizontally to handle the 100k peak.

---

## 2. Redis Key-Value Store Design
Redis is utilized as a **Read-through LRU (Least Recently Used) cache** and a **Global Counter**.

| Use Case | Key Pattern | Value Type | Example Key:Value |
| :--- | :--- | :--- | :--- |
| **URL Mapping** | `shortCode:{code}` | String | `shortCode:8B2x` : `https://original-site.com/very-long-path` |
| **Global Counter** | `global_counter` | Integer | `global_counter` : `145600` |
| **Custom Alias Check** | `alias:{custom_name}` | Boolean/ID | `alias:my-birthday` : `1` |

* **Redirection Flow:** The Read Service first checks Redis for the `shortCode`. If found (Cache Hit), it redirects immediately; if not (Cache Miss), it queries the Database and populates Redis.
* **Counter Flow:** The Write Service uses `INCR` on the `global_counter` in Redis to get a unique number, which is then converted to a Base62 string.

---

## 3. High-Level Architecture (Zookeeper Integration)



### Distributed Coordination (Zookeeper)
To avoid a single point of failure in the counter, Zookeeper coordinates unique IDs for distributed workers.
* **Worker Registration:** Each Write Service instance registers as an ephemeral node in Zookeeper to receive a unique `workerId`.
* **Snowflake ID Generation:** * **41-bit Timestamp:** Supports ~69.7 years of unique IDs.
    * **10-bit Worker ID:** Supports up to 1,024 active servers.
    * **12-bit Sequence:** Allows 4,096 IDs per millisecond per worker.

---

<img width="3287" height="3592" alt="Image" src="https://github.com/user-attachments/assets/023cd9be-4b01-438d-99ba-8e07a4f6778c" />

## 4. Database Schema
| Entity | Storage Type | Fields |
| :--- | :--- | :--- |
| **Url Table** | SQL (Postgres/MySQL) | `shortUrl (PK)`, `longUrl`, `userId (FK)`, `createdAt`, `expirationTime` |
| **User Table** | SQL | `userId (PK)`, `name`, `email` |

---

## 5. System Design Interview Q&A

1. **Q: Why is "Availability >> Consistency" appropriate here?**
    * **A:** If a short URL takes a few seconds to "propagate" to all read replicas, it is acceptable. However, the redirection service must never be down.

2. **Q: How do you handle collisions if using a Random Number Generator?**
    * **A:** This is known as the **Birthday Paradox**. Even with 1B URLs, there is a chance of collision. The service must check the Database/Redis before finalizing a random code.

3. **Q: How does Zookeeper improve on a central Redis counter?**
    * **A:** A central Redis counter is a single point of failure (SPOF). Zookeeper allows for a decentralized Snowflake ID approach where each server generates IDs independently without constant network calls to a central counter.

4. **Q: What is the benefit of 302 redirects for business intelligence?**
    * **A:** 302 redirects prevent the browser from caching the result, meaning every click hits our servers, allowing us to track user IP, location, and click-through rates.

5. **Q: How do you handle "Predictability" for security?**
    * **A:** Instead of a raw counter (which is guessable), we use a bijective function or shuffle the Base62 character set to make the generated short codes look random to users.

6. **Q: Why use 41 bits for the timestamp in Snowflake IDs?**
    * **A:** $2^{41}$ milliseconds covers roughly 69 years, which is sufficient for the lifecycle of most modern software systems.