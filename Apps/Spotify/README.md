# 🎧 The Senior System Design Playbook: Design Spotify

> **Target Audience:** Senior Engineers & Architects  
> **Goal:** Design an audio streaming platform with **500M+ Users**, **Instant Playback**, and **Offline Capabilities**.

In a senior interview, "Design Spotify" differs from Netflix. The files are smaller, the catalog is larger (100M+ tracks vs 5k movies), and the recommendation engine (Discover Weekly) is the core product differentiator.

---

## 📖 Table of Contents
1. [Part 1: Requirements & The "Gapless" Constraint](#-part-1-requirements--the-gapless-constraint)
2. [Part 2: High-Level Architecture (Control vs. Data)](#-part-2-high-level-architecture-control-vs-data)
3. [Part 3: The Data Plane (CDN vs. P2P)](#-part-3-the-data-plane-cdn-vs-p2p)
4. [Part 4: Metadata & Search (The Massive Catalog)](#-part-4-metadata--search-the-massive-catalog)
5. [Part 5: Recommendations (Vectors & AI)](#-part-5-recommendations-vectors--ai)
6. [Part 6: Senior Level Q&A Scenarios](#-part-6-senior-level-qa-scenarios)

---

## 🧮 Part 1: Requirements & The "Gapless" Constraint

### 1. Requirements
* **Functional:** Play Music, Search, Create Playlists, "Discover Weekly", Offline Mode.
* **Non-Functional:**
    * **Low Latency:** < 200ms time to first byte (TTFB).
    * **High Availability:** The music never stops.
    * **Consistency:** Playlist updates must eventually sync across devices.

### 2. Back-of-the-Envelope Math (The "Small File" Problem)
* **Catalog:** 100 Million Songs.
* **Avg Size:** 5 MB per song (High Quality).
* **Total Storage:** $100M \times 5MB = 500TB$.
* **Insight:** Unlike Netflix (Petabytes), Spotify's entire catalog is only **500TB**. This is small enough to fit entirely in the RAM/SSD cache of a moderate-sized CDN network. The challenge isn't storage; it's **Retrieval Latency**.

---

## 🏗️ Part 2: High-Level Architecture (Control vs. Data)

We separate the **Player** (Data) from the **App** (Control).



### 1. The Control Plane (Microservices)
* **Playlist Service:** Manages user collections.
* **Metadata Service:** Artist, Album, Song info.
* **User Profile:** Subscription status, Following.
* **Search Service:** Elasticsearch cluster.

### 2. The Data Plane (Audio Delivery)
* **Origin Storage:** Google Cloud Storage (GCS) / S3. Contains the "Master" files.
* **Transcoding:** When a label uploads a `.wav`, workers transcode it into `AAC`, `Ogg Vorbis`, and `HE-AACv2` (for low bandwidth).
* **Encrypted Chunks:** Files are split into chunks and encrypted (DRM) to prevent piracy.

---

## ☁️ Part 3: The Data Plane (CDN vs. P2P)

**Interviewer:** *"Does Spotify use Peer-to-Peer (P2P) like BitTorrent?"*

### The Historical Context (Senior Knowledge)
* **Old Spotify (2010):** Used P2P. Desktop clients shared cache with other users to save bandwidth costs.
* **Modern Spotify:** **Pure CDN (Centralized).**
    * *Why?* Mobile phones have limited battery/upload bandwidth. Cloud storage became cheap. P2P is unpredictable.

### The Caching Strategy (The "Spotify Special")
Spotify relies heavily on **Local Caching**.
1.  **Memory Cache:** The next 2 songs in the queue are pre-fetched into RAM.
2.  **Disk Cache:** Played songs are stored on the user's disk (encrypted).
    * *Result:* 50% of plays come from the local disk, never hitting the server.
3.  **CDN (Edge):** If not local, fetch from nearest Edge location.

---

## 🗃️ Part 4: Metadata & Search (The Massive Catalog)

How do we organize 100M songs with complex relationships (Artist -> Album -> Track -> Featuring X)?

### 1. The Database Choice
* **PostgreSQL:** For User data, Billing, and relational mapping.
* **Cassandra / ScyllaDB:** For **Playlists**.
    * *Why?* A playlist is an ordered list of Track IDs. It is write-heavy (users constantly add/remove). Cassandra handles high write throughput and replication well.

### 2. Search (Fuzzy & Weighted)
* **Elasticsearch / Solr:**
    * Store metadata Documents: `{ "title": "Hello", "artist": "Adele", "popularity": 99 }`.
* **Typo Tolerance:** Implementation of Levenshtein distance (Fuzzy matching).
* **Ranking:** Boost results based on user's region and global popularity.
    * *Query:* "Hello" -> Adele (99 pop) ranks higher than Lionel Richie (80 pop).

---

## 🤖 Part 5: Recommendations (Vectors & AI)

How does "Discover Weekly" work? This is the most technical part.

### 1. Collaborative Filtering (Matrix Factorization)
* **Concept:** "Users who listened to X also listened to Y."
* **The Matrix:** Rows = Users, Columns = Songs.
* **Problem:** The matrix is massive and sparse.
* **Solution:** **Implicit Feedback**. We don't wait for "Likes." We treat "Played > 30 seconds" as a Like.

### 2. Vector Embeddings (The AI Approach)
* Convert every song into a vector (array of numbers) based on audio features (Tempo, Key, Loudness) and User Logs.
* **Annoy (Approximate Nearest Neighbors Oh Yeah):**
    * Spotify open-sourced library.
    * It places vectors in a high-dimensional space.
    * **Recommendation:** "Find the 10 nearest neighbors to the vector of the user's taste."



[Image of Vector Embedding Space Recommendation Diagram]


---

## 🧠 Part 6: Senior Level Q&A Scenarios

### Scenario A: Instant Playback (Latency)
**Interviewer:** *"When I press 'Next', the song starts instantly. How? Network latency is 100ms."*

* ✅ **Senior Answer:** "**Predictive Pre-fetching.**"
    * The client doesn't just download the current song.
    * It looks at the **Play Queue**.
    * It downloads the *first 10 seconds* of the next 3 songs in the background.
    * When you press Next, the data is already in RAM.

### Scenario B: The "Justin Bieber" Problem (Thundering Herd)
**Interviewer:** *"New album drops. 50M users hit 'Play' at midnight."*

* ✅ **Senior Answer:** "**CDN Warming & Randomized Jitter.**"
    * **Warming:** Push the album files to all Edge Nodes (ISPs) 24 hours prior.
    * **Jitter:** We cannot serve 50M requests at 12:00:00.
        * The App UI says "Available", but the actual API call has a random backoff (0-2000ms).
        * Users won't notice a 1s delay, but it saves the backend.

### Scenario C: Offline Mode Security
**Interviewer:** *"If I download songs and go offline, how do you prevent me from extracting the MP3s?"*

* ✅ **Senior Answer:** "**AES Encryption + Local Key Management.**"
    * Files on disk are AES-128 encrypted.
    * The App has the Decryption Key stored in the secure enclave (Keychain/Keystore).
    * The App is a "Black Box" player. It decrypts chunks in memory only during playback.

### Scenario D: Collaborative Playlists (Concurrency)
**Interviewer:** *"User A and User B are editing the same playlist simultaneously. How do we prevent conflicts?"*

* ✅ **Senior Answer:** "**Operational Transformation (OT) or CRDTs.**"
    * A playlist is a list of operations: `[Add(SongA, Index1), Remove(SongB)]`.
    * We use a **Conflict-Free Replicated Data Type (CRDT)** structure in the backend.
    * It merges operations automatically so both users eventually see the same list, regardless of network timing.