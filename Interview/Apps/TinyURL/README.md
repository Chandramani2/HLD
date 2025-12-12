# System Design Case Study: URL Shortener (TinyURL)

## The Challenge
**Goal:** Design a system that converts long URLs into short aliases (e.g., `tinyurl.com/xyz123`) and redirects users when clicked.
* **Scale:** 100 Million new URLs per day. Billions of redirects.
* **Critical Requirement:** The short ID (e.g., `xyz123`) must be **globally unique** and as short as possible (usually 6-7 characters).

---

## 1. The Core Database Decision: SQL vs. NoSQL
*This is the most common trap in this interview.*

### Option A: Relational (PostgreSQL/MySQL)
* **The "Easy" Way:** SQL databases have a built-in feature called `AUTO_INCREMENT`.
* **How it works:** You insert the long URL, the DB gives you ID `1001`. You convert `1001` to Base62 (`abc`). Done.
* **The Problem:** Writing 100M rows/day will crush a single MySQL instance. Sharding a relational DB based on an auto-incrementing ID is extremely painful (you have to manage offsets across servers to avoid collisions).

### Option B: NoSQL (Cassandra/DynamoDB)
* **The "Scalable" Way:** These databases can handle infinite writes and reads. You can easily shard by `ShortID`.
* **The Problem (The "Trick" Requirement):** NoSQL databases **do not** have auto-incrementing IDs. They rely on UUIDs (which are too long: `123e4567-e89b...`).
* **The Fix:** If you choose NoSQL for scale, you must design a separate mechanism to generate unique, short IDs.

---

## 2. The "Specific Trick" for Unique IDs
Since we chose **NoSQL (e.g., Cassandra)** for its read speed and scale, we cannot rely on the database to generate IDs. We have two main "tricks" to solve this:

### Approach 1: The "Counter" Trick (Redis)
Instead of using a database counter, use a **Redis Cluster**.
1.  **Redis INCR:** Redis is single-threaded and atomic. It can increment a number from 1,000,000 to 1,000,001 instantly, ensuring no two servers get the same number.
2.  **Batching:** To reduce load, your web servers don't ask Redis for *every* ID. They ask for a "range."
    * Server A asks Redis: *"Give me 100,000 IDs."*
    * Redis says: *"Okay, you own range 1,000,000 to 1,100,000."*
    * Server A uses those IDs in memory. If Server A crashes and loses the unused IDs, it's fine; we just skip them.

### Approach 2: The Key Generation Service (KGS)
*This is the "advanced" answer.*
1.  **Offline Generation:** We build a standalone service that pre-generates random 6-character strings (`abc001`, `abc002`...) and stores them in a separate database (can be SQL or a specialized KV store).
2.  **Two States:** These keys have a flag: `Used` or `Not Used`.
3.  **Loading:** The web servers load 1,000 unused keys into memory. When a user sends a URL, the server instantly grabs a pre-made key from memory, marks it as used, and saves the mapping to the main NoSQL DB.
4.  **Benefit:** 0% collision chance, zero calculation time on the write path.

---

## 3. Final Architecture

### Write Path (Shortening)
1.  User sends `create(long_url)`.
2.  App Server requests a unique ID range from **Redis** (or fetches a key from **KGS**).
3.  App Server converts ID to Base62 (e.g., ID 10005 $\rightarrow$ `s5d`).
4.  App Server writes key: `s5d`, value: `long_url` to **Cassandra** (Wide-Column Store).
    * *Why Cassandra?* Massive write throughput and easy replication.

### Read Path (Redirecting)
1.  User requests `GET /s5d`.
2.  Load Balancer checks **Redis Cache** first.
    * **Hit:** Redirect immediately (10ms).
    * **Miss:** Fetch from Cassandra, update Cache, then redirect.
---

## 1. The Database Decision
The choice between SQL and NoSQL dictates your ID generation strategy.

### **Option A: Relational Database (SQL)**
* **Database:** PostgreSQL / MySQL.
* **Strategy:** Use the built-in `AUTO_INCREMENT` feature.
* **Workflow:** Insert Record $\rightarrow$ Get ID (1001) $\rightarrow$ Convert to Base62 (e.g., "a9").
* **Pros:** Simple, guarantees uniqueness easily.
* **Cons:** Hard to scale writes. Multi-master replication with auto-increment is complex (requires offsetting).

### **Option B: NoSQL (Wide-Column or Key-Value)**
* **Database:** Cassandra, DynamoDB, or MongoDB.
* **Strategy:** Optimized for massive reads/writes.
* **Pros:** Infinite horizontal scaling. High availability.
* **Cons:** **No Auto-Increment.** You cannot ask the DB for the "next number."
* **Verdict:** Preferred for modern, high-scale systems, but requires an external ID generator (The "Trick").

---

## 2. The "Trick": Unique ID Generation Strategies
Since NoSQL lacks auto-increment, we must generate IDs *before* inserting into the database.

### **Strategy 1: Distributed Counter (Redis)**
Use Redis as a centralized counter because it supports atomic increments (`INCR`).
* **Batching:** To avoid hitting Redis for every single URL, application servers fetch a **Range** of IDs (e.g., 10,000 at a time).
* **Failure Case:** If a server crashes, the unused IDs in its range are lost. This is acceptable (we have 3.5 trillion combinations).

### **Strategy 2: Key Generation Service (KGS)**
A standalone service that pre-generates keys offline.
* **Workflow:**
    1. KGS generates random 6-char strings (`abc001`, `xyz999`) and stores them in a "Key DB".
    2. Web Servers request a batch of keys.
    3. KGS moves keys from `Unused` to `Used` table.
* **Benefit:** Zero collision checks during the user request. Lowest possible latency.

---

## 3. Data Flow & Architecture

| Component | Choice | Why? |
| :--- | :--- | :--- |
| **Frontend DB** | **Cassandra / DynamoDB** | Holds the `ShortID -> LongURL` mapping. Needs massive read throughput and linear write scaling. |
| **Cache** | **Redis** | Stores the most popular 20% of URLs. Handles ~90% of read traffic. |
| **ID Gen** | **ZooKeeper / Redis** | Manages the atomic integer ranges for servers to ensure no two servers generate the same ID. |
| **Analytics** | **Kafka + Hadoop** | Async stream to track click counts (don't write click counts to the main DB synchronously!). |

### **Step-by-Step Flow**
1.  **Write:** User sends URL $\rightarrow$ Server gets ID from **Redis Range** $\rightarrow$ Converts to Base62 $\rightarrow$ Writes to **Cassandra**.
2.  **Read:** User clicks link $\rightarrow$ Server checks **Redis Cache** $\rightarrow$ If miss, read **Cassandra** $\rightarrow$ Return 301 Redirect.

# Appendix: Base62 Conversion Implementation (Java)

This utility class handles the bidirectional conversion between the integer ID (generated by Redis/KGS) and the short string (displayed to the user).

### The Math
* **Base62 Alphabet:** `0-9`, `a-z`, `A-Z` (Total 62 characters).
* **Encoding:** Works exactly like converting a decimal number to binary, but using modulo 62 instead of 2.
* **Decoding:** Reverses the process to retrieve the database ID from the short string.

```java
import java.util.ArrayList;
import java.util.List;

public class Base62Encoder {
    
    // The set of characters used for encoding
    // Order matters! 0-9, a-z, A-Z
    private static final String ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final int BASE = ALPHABET.length(); // 62

    /**
     * Encodes a database ID into a Short URL string.
     * Example: 10005 -> "s5d"
     */
    public static String encode(long id) {
        if (id == 0) return String.valueOf(ALPHABET.charAt(0));

        StringBuilder sb = new StringBuilder();
        
        while (id > 0) {
            // 1. Get the remainder (modulo)
            int remainder = (int) (id % BASE);
            
            // 2. Map remainder to a character
            sb.append(ALPHABET.charAt(remainder));
            
            // 3. Divide by base to move to the next digit
            id /= BASE;
        }
        
        // Reverse string because we processed least significant digit first
        return sb.reverse().toString();
    }

    /**
     * Decodes a Short URL string back to a database ID.
     * Example: "s5d" -> 10005
     */
    public static long decode(String str) {
        long id = 0;
        
        for (char c : str.toCharArray()) {
            // 1. Shift the current total (like moving decimal place)
            id = id * BASE;
            
            // 2. Add the value of the current character
            int value = ALPHABET.indexOf(c);
            
            if (value == -1) {
                throw new IllegalArgumentException("Invalid character in Short URL: " + c);
            }
            
            id += value;
        }
        
        return id;
    }

    // Simple test to demonstrate usage
    public static void main(String[] args) {
        long originalId = 123456789L;
        
        // 1. Encode
        String shortUrl = encode(originalId);
        System.out.println("ID: " + originalId + " -> ShortURL: " + shortUrl); 
        // Output: 8M0kX
        
        // 2. Decode
        long decodedId = decode(shortUrl);
        System.out.println("ShortURL: " + shortUrl + " -> ID: " + decodedId);
        // Output: 123456789
    }
}
```