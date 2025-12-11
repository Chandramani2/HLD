# Apache Cassandra: Architecture, Internals, and Configuration

## 1. Internal Working & Architecture
Cassandra operates on a **Peer-to-Peer** (Ring) architecture. All nodes are equal; there is no master-slave relationship, ensuring no single point of failure.

### A. The Write Path (Optimized for Speed)
Cassandra treats writes as "append-only" operations, minimizing disk seek time.
1. **Commit Log (Disk):** Every write is immediately appended to the Commit Log for durability (disaster recovery).
2. **MemTable (RAM):** Simultaneously, data is written to a sorted in-memory structure called the MemTable.
3. **Flush (SSTable):** When the MemTable is full, it is flushed to disk as an immutable **SSTable** (Sorted String Table).
4. **Compaction:** Background processes merge multiple SSTables to remove obsolete data and tombstones.

### B. The Read Path (Reconciliation)
Reads are more expensive than writes as they may require merging data from multiple sources.
1. **MemTable:** Checked first for the freshest data.
2. **Row Cache (Optional):** Checked for frequently accessed rows.
3. **Bloom Filter:** A probabilistic structure checks if an SSTable *might* contain the data (avoids unnecessary disk seeks).
4. **Partition Key Cache & Index:** Locates the data offset on the disk.
5. **SSTable Scan:** Reads data from relevant SSTables.
6. **Merge:** The latest version of the data (based on timestamps) is returned to the client.

---

## 2. Core Data Structures (LSM Tree)

| Structure | Location | Mutability | Function |
| :--- | :--- | :--- | :--- |
| **Commit Log** | Disk | Append-only | Crash recovery. |
| **MemTable** | RAM | Mutable | Write-back cache; sorts data before flushing. |
| **SSTable** | Disk | Immutable | Persistent storage. Once written, it is never changed, only deleted by compaction. |
| **Bloom Filter**| RAM | N/A | Determines if a key exists in a file to save I/O. |
| **Tombstone** | Disk | Immutable | A marker indicating a delete operation. |

---

## 3. Key Components
* **Node:** A single Cassandra instance.
* **Rack:** Logical grouping of nodes (failure zone).
* **Gossip:** Protocol for nodes to discover cluster state (up/down status) every second.
* **Snitch:** Maps IP addresses to physical racks and data centers.
* **Partitioner:** Determines how data is distributed across the ring (Consistent Hashing).

---

## 4. Configuration (`cassandra.yaml`)
Key settings found in `/etc/cassandra/cassandra.yaml`:

* `cluster_name`: Must match on all nodes.
* `seeds`: IP list of stable nodes used for bootstrapping new nodes.
* `listen_address`: IP for internode communication (Gossip/Replication).
* `rpc_address`: IP for client driver connections.
* `endpoint_snitch`: `GossipingPropertyFileSnitch` is recommended for production (rack-aware).
* `num_tokens`: Default `256`. Defines the number of vnodes (virtual nodes) per physical machine.
* `auto_bootstrap`: Set to `false` only when initializing a fresh cluster; `true` when adding nodes.

---

# Senior Backend Developer Interview Q&A: Cassandra

### Architecture & Internals
**Q1: How does Cassandra achieve high availability compared to a Master-Slave system?**
**A:** Cassandra uses a peer-to-peer leaderless architecture. Data is replicated across multiple nodes (Replication Factor). If one node goes down, requests are handled by other replicas. There is no downtime for master failover.

**Q2: Explain the concept of Tunable Consistency.**
**A:** Developers can trade off consistency for availability per query.
* **Write `ANY`:** Fast, low consistency.
* **Read/Write `QUORUM`:** Strong consistency (R + W > Replication Factor).
* **Read `ONE`:** Fast, eventual consistency.

**Q3: What are Tombstones and why are they dangerous?**
**A:** A Tombstone is a marker for a deleted record. Since SSTables are immutable, data isn't immediately removed. If a query scans a partition with thousands of tombstones, it causes high latency and potential `OutOfMemory` errors (the "Doomstone" problem).

**Q4: What is the difference between Partition Key and Clustering Key?**
**A:**
* **Partition Key:** Determines *which node* stores the data (distribution).
* **Clustering Key:** Determines *how data is sorted* within that partition on the disk.

**Q5: How does Cassandra handle "Hinted Handoff"?**
**A:** If a replica node is down during a write, the coordinator stores a "hint" locally. When the target node comes back online, the coordinator replays the write to ensure data consistency.

### Data Modeling & Performance
**Q6: Why are "Wide Partitions" bad in Cassandra?**
**A:** A single partition (defined by the Partition Key) should ideally stay under 100MB. Wide partitions (e.g., storing all user logs in one partition) cause heap pressure, inefficient compaction, and hotspots on specific nodes.

**Q7: When would you use Leveled Compaction Strategy (LCS) over Size-Tiered Compaction Strategy (STCS)?**
**A:**
* **STCS (Default):** Best for heavy write workloads (IoT logs).
* **LCS:** Best for read-heavy workloads where read latency matters, as it guarantees a row exists in fewer SSTables.

**Q8: Why are Secondary Indexes generally discouraged in Cassandra?**
**A:** They require a "scatter-gather" read operation. The coordinator must query *every* node in the cluster to find the data, which scales poorly. `Materialized Views` or separate tables are preferred.

**Q9: What is the CAP theorem status of Cassandra?**
**A:** Cassandra is an **AP** system (Available and Partition Tolerant). It sacrifices Strong Consistency for Availability, though it can mimic CP behavior using `QUORUM` or Lightweight Transactions (LWT).

**Q10: Explain the purpose of the Bloom Filter in the read path.**
**A:** It is a memory-efficient probabilistic structure that tells the database if a key *definitely does not exist* in an SSTable. This prevents expensive disk seeks for data that isn't there.

### Operations & Troubleshooting
**Q11: What happens during a "Nodetool Repair"?**
**A:** It is an anti-entropy process that compares Merkle Trees (hashes of data) across replicas. If mismatches are found, it streams the missing/correct data to sync the nodes.

**Q12: How do Virtual Nodes (vnodes) help when adding a new server?**
**A:** Without vnodes, adding a node requires rebalancing huge chunks of contiguous token ranges. With vnodes (e.g., 256 tokens), the new node takes small, random slices of data from *all* existing nodes, speeding up bootstrapping and balancing load evenly.

**Q13: What is the cost of Lightweight Transactions (LWT / `IF EXISTS`)?**
**A:** LWT uses the Paxos consensus protocol. It requires 4 round-trips between nodes (Prepare, Promise, Propose, Accept), making it significantly slower than standard writes. Use sparingly.

**Q14: How does Cassandra handle timestamps during a write conflict?**
**A:** Cassandra uses "Last Write Wins" (LWW) based on the client-provided timestamp. If clocks are not synchronized (NTP drift), data loss can occur where older data overwrites newer data.

**Q15: What is the impact of setting `durable_writes=false`?**
**A:** It skips writing to the Commit Log. Writes are faster, but if the node crashes before flushing the MemTable to the SSTable, that data is permanently lost.

### Advanced
**Q16: How does the Snitch affect replication?**
**A:** The `NetworkTopologyStrategy` uses the Snitch to place replicas on different racks and data centers. This ensures that a rack failure doesn't result in data loss.

**Q17: What is Read Repair?**
**A:** When a client reads data at `QUORUM`, if one replica has obsolete data, Cassandra returns the latest data to the client *and* triggers a background update to fix the stale replica.

**Q18: Why is `COUNT(*)` slow in Cassandra?**
**A:** Cassandra has no metadata counter for table size. A `COUNT(*)` forces a full table scan, reading every SSTable and filtering tombstones, which usually times out on large datasets.

**Q19: How do you handle Time Series data in Cassandra?**
**A:** Use the "Bucket" pattern. Combine a base ID and a time bucket (e.g., `sensor_id : 2023-10-25`) as the Partition Key to keep partitions small, and use the specific Timestamp as the Clustering key for sorting.

**Q20: What is the role of the Coordinator Node?**
**A:** Any node can act as a coordinator for a client request. It is responsible for hashing the key, determining which replicas own the data, sending requests to them, and gathering responses to satisfy the Consistency Level.