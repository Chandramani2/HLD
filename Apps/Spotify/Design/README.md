# System Design: Audio Streaming (Spotify)

## 1. Requirements

### Functional
* **Music Playback:** Stream songs with no buffering.
* **Music Discovery:** "Discover Weekly", Home Page recommendations.
* **Search:** Find Artists, Albums, Songs, Podcasts.
* **Playlists:** Create, Edit, and Share playlists (Collaborative).
* **Offline Mode:** Download encrypted content.

### Non-Functional
* **Low Latency:** Playback must start < 200ms.
* **High Availability:** 99.99% (Music stops = User churn).
* **Scalability:** 100 Million tracks, 500 Million users.

## 2. Estimations (Scale)

* **Users:** 500 Million MAU.
* **Songs:** 100 Million tracks.
* **Storage:**
    * Avg song size: 5MB.
    * Transcoding: 3 qualities (96, 160, 320 kbps) × 2 formats (Ogg, AAC).
    * Total Storage: 100M × 5MB × 6 versions ≈ **3 PB** (Much smaller than Netflix).
    * **The Real Challenge:** Metadata. Storing the graph of Artists, Writers, Producers, and Rights Holders is complex.

---

## 3. High-Level Architecture

Spotify moved from a P2P model (early days) to a standard **Client-Server-CDN** model.



| Component | Database/Tech Choice | Reason |
| :--- | :--- | :--- |
| **User/Billing** | **PostgreSQL** | ACID for subscriptions. |
| **Playlist Service** | **Cassandra / Bigtable** | Massive write throughput. Playlists are modified constantly. |
| **Metadata** | **PostgreSQL + Memcached** | Complex relations (Artist -> Album -> Song). |
| **Search** | **Elasticsearch** | Text search for titles/artists. |
| **Audio Files** | **GCP Cloud Storage / S3 + CDN** | Static asset delivery. |

---

## 4. Schema Design

### A. Metadata (PostgreSQL - The Graph)
Relational is best here because data is highly structured and relational (Artist has many Albums, Albums have many Songs).

```sql
CREATE TABLE songs (
    song_id UUID PRIMARY KEY,
    title VARCHAR(255),
    artist_id UUID,
    album_id UUID,
    duration_sec INT,
    file_path_map JSONB -- {'320kbps': 'cdn/path/high.ogg'}
);

CREATE TABLE artists (
    artist_id UUID PRIMARY KEY,
    name VARCHAR(255),
    monthly_listeners BIGINT
);
```

### B. Playlist Service (Cassandra)
This is the hardest part. A playlist is an ordered list of songs.
Standard SQL struggles when you have 5 billion playlists and users reordering tracks #500 to #2.

```cql
-- Simplified approach
CREATE TABLE playlists (
    playlist_id UUID,
    revision_id UUID, -- For version control/collaborative editing
    owner_id UUID,
    song_list LIST<UUID>, -- Or a separate table for tracks if list is huge
    is_public BOOLEAN,
    PRIMARY KEY (playlist_id)
);
```
**Optimization:** Spotify actually uses a system allowing "snapshots" of playlists to handle collaborative editing (like Git for music).

---

## 5. Senior Interview Topics & Logic

### Q1: "How does 'Discover Weekly' work?" (The AI part)
It's not real-time. It's a **Batch Job** (Hadoop/Spark/Dataflow).

1.  **Collaborative Filtering:** "Users who listened to Song A also listened to Song B."
2.  **Matrix Factorization:** Create a huge matrix of Users vs. Songs.
3.  **NLP:** Analyze text in playlists. If a song is in a playlist called "Sad Breakup," the AI tags the song with "Sad" sentiment.
4.  **Audio Analysis:** Raw audio models (CNNs) analyze tempo/key to find similar sounding obscure tracks.
5.  **Execution:** The pipeline runs once a week, pre-calculates the list, and stores it in **Cassandra** for fast retrieval on Monday morning.

### Q2: "Design the 'Top 50 Global' Leaderboard."
**Naive:** `SELECT count(*) FROM plays GROUP BY song_id` (Too slow).
**Senior:** **Stream Aggregation (Kafka + Flink).**

1.  **Event:** Every time a user listens > 30s, the client emits a `SongPlayedEvent`.
2.  **Ingest:** Kafka receives millions of events/sec.
3.  **Process:** Apache Flink aggregates counts in time windows (e.g., 10-second sliding window).
4.  **Store:** Write the aggregated counts to **Redis** (Sorted Set).
5.  **Read:** Client queries Redis for the top 50 keys.

### Q3: "How do we handle Offline Mode and Piracy?"
**DRM (Digital Rights Management).**

* Spotify does not download an MP3. It downloads an **Encrypted Blob** (AES-128/256).
* **The Key:** The decryption key is stored in the app's secure storage (Keystore/Keychain).
* **Expiry:** The key has a TTL (Time To Live). If the user goes online after 30 days, the app checks subscription status. If `Active`, it renews the key. If `Cancelled`, it deletes the key, rendering the blob useless.

---

## 6. API & Protocol Design (The "Hermes" Protocol)

Spotify originally used a custom protocol over TCP called **Hermes** (based on ZeroMQ pattern) for low latency, but now uses standard APIs.

### A. Playback Request
* **Endpoint:** `GET /v1/tracks/{id}/stream`
* **Response:**
    * Does NOT return the file.
    * Returns a **CDN URL** with a short-lived access token.
    * Returns **Decryption Key** (if authorized).

### B. Search (Typeahead)
* **Endpoint:** `GET /v1/search?q=taylor&type=artist`
* **Backend:** Hits Elasticsearch.
* **Optimization:** "Fuzzy Matching" and "Popularity Boosting". (Typing "Tailer Swift" should still find "Taylor Swift").

---

## 7. Caching Strategy (The "Senior" Detail)

Spotify has a very unique caching mechanism on the Client side.

1.  **L1 Cache (Memory):** The next 10 seconds of the song are in RAM (Ring Buffer).
2.  **L2 Cache (Disk):** Recently played songs are stored on disk (up to 1GB or user limit).
3.  **Predictive Caching:** If you are playing Song #3 in an Album, the app proactively downloads the first 30 seconds of Song #4. This ensures **Zero Latency** when the track changes.

---

## 8. Summary Checklist

* [ ] **Database:** Postgres (Metadata), Cassandra (Playlists), Redis (Leaderboards).
* [ ] **Storage:** Google Cloud Storage / S3.
* [ ] **Protocol:** HTTP/2 + Encrypted Audio Blobs.
* [ ] **Algorithm:** Collaborative Filtering (Spark) for Recommendations.
* [ ] **CDN:** Essential for global audio delivery.