# 🔗 The Senior System Design Playbook: Design TinyURL

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Design a URL Shortener handling **Billions of clicks**, **Low Latency Redirection**, and **Granular Analytics**.

In a senior interview, "Design TinyURL" is not just about `Base62(ID)`. It is about **Write Concurrency**, **301 vs 302 Redirect trade-offs**, and **Offline Key Generation**.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The Math (Base62)](#-part-1-requirements--the-math-base62)
2. [Part 2: High-Level Architecture (301 vs 302)](#-part-2-high-level-architecture-301-vs-302)
3. [Part 3: The Core Algorithm (Key Generation Service)](#-part-3-the-core-algorithm-key-generation-service)
4. [Part 4: Data Layer (Scale & Cleanup)](#-part-4-data-layer-scale--cleanup)
5. [Part 5: Analytics (The Hidden Complexity)](#-part-5-analytics-the-hidden-complexity)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The Math (Base62)

### 1. Requirements
* **Functional:** Shorten URL, Redirect, Custom Aliases, Expiry.
* **Non-Functional:**
  * **Read-Heavy:** 100 reads for every 1 write.
  * **Speed:** Redirection must happen in < 10ms.
  * **Unpredictability:** IDs should not be sequential (prevent scraping).

### 2. Back-of-the-Envelope Math (Capacity)
* **Scale:** 100 Million new links/month.
* **Lifetime:** System runs for 10 years.
* **Total Objects:** $100M \times 12 \times 10 = 12 \text{ Billion URLs}$.
* **Encoding Math (Base62):**
  * Characters: `[A-Z, a-z, 0-9]` = 62 characters.
  * Length 6: $62^6 \approx 56 \text{ Billion}$.
  * **Conclusion:** A **7-character string** ($62^7 \approx 3.5 \text{ Trillion}$) is more than enough for 100 years.

---

## 🏗️ Part 2: High-Level Architecture (301 vs 302)

The most critical HTTP decision you will make.

### 1. The Redirection Flow
1.  User clicks `tiny.url/AbCd123`.
2.  LB hits **Service**.
3.  Service looks up `AbCd123` in **Redis**.
4.  If Miss -> Look up in **DB** (Cassandra/DynamoDB).
5.  Service returns HTTP Redirect to original URL.

### 2. The Interview Trap: 301 vs 302
**Interviewer:** *"Which HTTP Status code do you use?"*

| Code | Meaning | Implication | Senior Choice |
| :--- | :--- | :--- | :--- |
| **301** | Permanent Redirect | Browser caches the mapping. Subsequent clicks **bypass your server**. | **Bad for Analytics.** Use only if server load is critical. |
| **302** | Temporary Redirect | Browser hits your server **every time**. | **Good for Analytics.** Allows tracking click source, time, and location. |



---

## 🔑 Part 3: The Core Algorithm (Key Generation Service)

How do we generate unique 7-char strings without collisions?

### Approach A: Hashing (MD5/SHA) - ❌ The Naive Way
* `Hash(LongURL)`.
* *Problem 1:* Hash is too long (128-bit). Truncating it causes collisions.
* *Problem 2:* Same LongURL = Same Hash. If User A and User B both shorten `google.com`, they get the same short link. We can't track them separately.

### Approach B: Auto-Increment Database ID - ⚠️ The Mid Way
* Use SQL Auto-Increment ID -> Convert to Base62.
* `ID 1000` -> `QiS`.
* *Problem:* Sequential IDs are guessable (Security risk). Hard to scale across multiple DB shards.

### Approach C: Key Generation Service (KGS) - 🏆 The Senior Way
**Pre-generate** keys offline.
1.  **KGS Worker:** Generates random 7-char strings, checks DB for uniqueness, and stores them in a **"Key DB"**.
2.  **Application Server:**
  * Doesn't compute anything.
  * Just pops a key from the **Key DB**.
  * Assigns `Key: "AbCd123"` to `Value: "google.com"`.
  * Marks key as "Used".
3.  **Concurrency:**
  * App Server caches 1,000 keys in memory (Buffer).
  * Even if App Server crashes, we only lose 1,000 unused keys (Acceptable).



---

## 💾 Part 4: Data Layer (Scale & Cleanup)

### 1. Database Choice
* **Schema:** `{ ShortKey (PK), LongURL, CreatedAt, Expiry, UserID }`.
* **Choice:** **NoSQL (DynamoDB / Cassandra / Riak)**.
  * *Why?* We only need Key-Value lookup. No complex joins. Read speed is paramount.
  * *Scale:* Easy to shard based on the `ShortKey`.

### 2. Handling Expiry (Purging)
We have billions of old links. How do we delete them?
* **Active:** Cron job runs at night (Too slow).
* **Passive (Lazy):** When a user tries to access a link, check `CurrentTime > Expiry`. If yes, delete and return Error.
* **TTL (DynamoDB/Redis):** Native TTL features automatically delete rows without custom code.

---

## 📊 Part 5: Analytics (The Hidden Complexity)

If you use 302 Redirects, you are writing analytics data for every click.

### The Write Bottleneck
* **Traffic:** 100k Clicks/sec.
* **Trap:** Do NOT write to the DB synchronously during the redirect. It adds latency.

### The Architecture
1.  **Redirect Service:** Returns 302 to user immediately.
2.  **Async Log:** Pushes event `{ShortID, Timestamp, IP, UserAgent}` to **Kafka**.
3.  **Analytics Service:** Consumes Kafka.
4.  **Aggregation:** Uses **Apache Flink** or **ClickHouse** to aggregate:
  * Clicks per Country.
  * Clicks per Browser.
  * Clicks per Hour.

---

## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: Custom Aliases
**Interviewer:** *"User wants `tiny.url/SuperBowl`. How do we handle concurrency if two users request it at once?"*

* ✅ **Senior Answer:** "**Conditional Write (CAS) / Unique Constraint.**"
  * We cannot use the KGS (random strings) here.
  * We must insert into the DB with a condition: `INSERT IF NOT EXISTS`.
  * First write succeeds. Second write receives `409 Conflict`.
  * Return error to the second user: "Alias taken."

### Scenario B: Short Key Exhaustion
**Interviewer:** *"What if we run out of 7-char combinations?"*

* ✅ **Senior Answer:** "**Append Strategy.**"
  * Simply move to 8 characters.
  * $62^8$ is 218 Trillion combinations. Enough for another millennium.

### Scenario C: Separating Read/Write
**Interviewer:** *"The Analytics writes are slowing down the Redirection reads."*

* ✅ **Senior Answer:** "**CQRS (Command Query Responsibility Segregation).**"
  * **Service A (Redirect):** Reads form Redis/DynamoDB. Optimized for speed.
  * **Service B (Analytics):** Writes to Kafka/ClickHouse. Optimized for ingestion.
  * The two flows never touch the same database tables.

### Scenario D: Malicious Use (Phishing)
**Interviewer:** *"Spammers are using our service to hide phishing URLs."*

* ✅ **Senior Answer:** "**Async Abuse Detection.**"
  * When a URL is shortened, put it in a "Pending Scan" queue.
  * Use Google Safe Browsing API to check the Long URL.
  * If flagged: Ban the User ID and disable the Short Link.
  * *Rate Limiting:* Limit anonymous users to 5 links/minute to prevent massive spam runs.

---

### **Final Checklist**
1.  **ID Gen:** Offline Key Generation (KGS) to avoid collisions.
2.  **DB:** NoSQL (Key-Value) for speed.
3.  **Redirect:** 302 for Analytics.
4.  **Analytics:** Async via Kafka (don't block the user).

**This concludes the TinyURL Playbook.**