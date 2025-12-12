# Internal Data Structures of NoSQL Databases

This document details the internal engineering and core data structures that power NoSQL systems (Key-Value, Document, Column, and Graph).

---

## 1. Log-Structured Merge-Trees (LSM Trees)
**Primary Use Case:** Heavy write workloads.
**Used In:** Apache Cassandra, HBase, LevelDB, RocksDB, ScyllaDB.

LSM Trees are designed to overcome the bottleneck of random disk I/O by turning random write requests into sequential writes.

### Internal Working:
The LSM tree is a collection of components:

1.  **MemTable (Memory Component):**
    * Writes are first logged to a **Commit Log** (for durability) and then inserted into an in-memory structure called the **MemTable** (often a balanced tree or Skip List).
    * Write complexity is fast: $O(\log N)$ or $O(1)$.

2.  **SSTable (Disk Component):**
    * When the MemTable reaches a threshold (e.g., 64MB), it is flushed to disk as an **SSTable (Sorted String Table)**.
    * **Immutable:** Once written, an SSTable is never modified.

3.  **Compaction:**
    * Background processes run **Compaction** to merge multiple SSTables into larger, sorted files, discarding deleted (tombstoned) or overwritten data.

4.  **Bloom Filters:**
    * To speed up reads, **Bloom Filters** are used to determine if a key is "definitely not" in an SSTable, preventing unnecessary disk seeks.

> **[Diagram: Log Structured Merge Tree architecture showing MemTable flush to SSTable]**

---

## 2. B-Trees and B+ Trees
**Primary Use Case:** Read-heavy workloads, Range queries.
**Used In:** MongoDB (WiredTiger), Couchbase, DynamoDB (storage nodes).

B-Trees offer predictable read performance and efficient range scans.

### Internal Working:
A B+ Tree is a self-balancing tree that maintains sorted data.

1.  **High Fan-out:** Nodes have hundreds of children, keeping the tree shallow. A tree with billions of records is often only 3–4 levels deep.
2.  **Disk-Friendly:** Node sizes align with OS memory pages (e.g., 4KB) to minimize disk I/O.
3.  **Leaf Nodes:** In a **B+ Tree**, actual data exists only in leaf nodes. Internal nodes contain only keys for navigation.
4.  **Linked Leaves:** Leaf nodes are linked in a list, making **Range Queries** (e.g., `20 < age < 30`) extremely fast ($O(\log N + k)$ where $k$ is the number of elements in the range).

> **[Diagram: B+ Tree structure highlighting linked leaf nodes]**

---

## 3. Skip Lists
**Primary Use Case:** In-memory sorting, MemTable implementation.
**Used In:** Redis (Sorted Sets), Apache Cassandra (MemTables).

A probabilistic data structure allowing $O(\log N)$ search complexity, similar to a balanced tree but simpler to implement concurrently.

### Internal Working:
1.  **Layers:** Multiple layers of "express lanes" exist above the base linked list.
2.  **Promotion:** A random process (coin flip) determines if an inserted element is promoted to higher layers.
3.  **Traversal:** Search starts at the top layer, skipping large chunks of data, dropping down layers only when overshooting the target.

> **[Diagram: Skip List structure showing multiple pointer layers]**

---

## 4. Inverted Indices
**Primary Use Case:** Full-text search.
**Used In:** Elasticsearch, Apache Solr, MongoDB (Text Search).

Maps content to document IDs, rather than IDs to content.

### Internal Working:
1.  **Tokenization:** Text is broken into tokens (e.g., "Blue sky" $\rightarrow$ "blue", "sky").
2.  **Index Creation:** A sorted list of unique terms maps to the documents containing them.
    * `blue` $\rightarrow$ `[Doc1, Doc5]`
    * `sky`  $\rightarrow$ `[Doc1, Doc3]`
3.  **Intersection:** A query for "Blue AND Sky" finds the intersection of the two lists (`Doc1`).

---

## 5. Hash Maps / Consistent Hashing
**Primary Use Case:** Key-Value lookups, Distributed Partitioning.
**Used In:** Redis, DynamoDB, Riak.

Provides $O(1)$ access for exact key matches.

### Internal Working:
1.  **Hashing:** Keys are passed through a hash function to generate a numerical value/bucket.
2.  **Consistent Hashing (Distributed):**
    * Data is mapped to a circular **Hash Ring**.
    * Nodes (Servers) are placed on the ring.
    * A key is stored on the first node found moving clockwise from the key’s hash.
    * Minimizes data movement when nodes are added/removed.

> **[Diagram: Consistent Hashing Ring with nodes and keys]**

---

## 6. Index-Free Adjacency
**Primary Use Case:** Graph traversals.
**Used In:** Neo4j, TigerGraph.

### Internal Working:
1.  **Physical Pointers:** Every node (vertex) stores a list of direct memory pointers to connected nodes (edges).
2.  **Traversal:** The engine "hops" from node to node using these pointers.
3.  **Performance:** Traversal cost is proportional to the subgraph visited, not the total dataset size ($O(k)$ neighbors), avoiding expensive global index lookups (Joins).

---

## Summary Comparison

| Data Structure | Best For... | Avg Complexity | Database Examples |
| :--- | :--- | :--- | :--- |
| **LSM Tree** | High Write Throughput | Write: $O(1)$ | Cassandra, RocksDB |
| **B+ Tree** | Reads & Range Scans | Read: $O(\log N)$ | MongoDB |
| **Hash Map** | Exact Key Match | Read/Write: $O(1)$ | Redis, DynamoDB |
| **Skip List** | In-Memory Ordering | Search: $O(\log N)$ | Redis (Sorted Sets) |
| **Inverted Index** | Text Search | Search: $O(1)$/term | Elasticsearch |