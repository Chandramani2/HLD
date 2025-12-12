# Internal Working of B-Trees and B+ Trees in NoSQL Databases

This document details the internal engineering of **B-Trees** and **B+ Trees**, the fundamental data structures powering the storage engines of many NoSQL databases (like MongoDB and Couchbase) designed for read-heavy and range-based workloads.

---

## 1. The Problem: Why not Binary Search Trees?
To understand B-Trees, one must first understand the limitations of standard Binary Search Trees (BST) in a database context.

* **Deep Trees:** In a binary tree, every node has only 2 children. With 1 million records, the tree height is roughly 20 ($2^{20} \approx 10^6$).
* **Disk I/O Bottleneck:** Nodes are stored on disk. Traversing 20 levels requires 20 separate disk seeks. Since disk seeks are slow (milliseconds), 20 seeks per query makes the database unusable.

**The Solution:** Make the tree "fatter" and "shallower" by allowing nodes to have hundreds of children. This is the **B-Tree**.

---

## 2. The B-Tree
A B-Tree is a self-balancing search tree designed specifically for systems that read and write large blocks of data.

### A. Internal Structure & Rules
A B-Tree of order $m$ satisfies these properties:
1.  **Nodes:** Every node contains at most $m-1$ keys and $m$ children.
2.  **Occupancy:** Every node (except root) must be at least half full ($m/2$ children).
3.  **Sorted:** Keys inside a node are sorted in ascending order.
4.  **Data Location (Crucial):** In a standard B-Tree, **data (payloads) can be stored in any node**, including the root or internal nodes. If you find the key, you stop searching immediately.

### B. Internal Algorithms

#### 1. Search ($O(\log N)$)
1.  Start at the **Root**.
2.  Perform a binary search (or linear scan) within the node's keys to find the correct child pointer.
3.  Follow the pointer to the next disk block.
4.  Repeat until the key is found or a leaf is reached.

#### 2. Insertion (The "Split" Mechanism)
1.  Find the leaf node where the key belongs.
2.  If the node is not full, insert the key in sorted order.
3.  **If the node is full:**
    * The node is **split** into two nodes.
    * The **median key** (middle element) is "promoted" (pushed up) to the parent node.
    * **Chain Reaction:** If the parent is full, it splits too. If the root splits, the tree height grows by 1.

#### 3. Deletion (The "Merge/Borrow" Mechanism)
1.  If deleting a key causes a node to be less than half full (underflow):
    * **Borrow:** Try to take a key from an immediate sibling ("rich neighbor").
    * **Merge:** If siblings are also poor, merge the node with a sibling and pull a key down from the parent.

---

## 3. The B+ Tree (The NoSQL Standard)
While B-Trees are efficient, **B+ Trees** are the industry standard for production databases. They are optimized specifically for range queries and modern file system page caching.

### A. Key Differences from B-Tree
1.  **Data only at Leaves:** Internal nodes **do not** store actual data (records/JSON). They store *only* keys to act as a roadmap. All actual data resides strictly in Leaf Nodes.
2.  **Redundant Keys:** A key might appear twice: once in an internal node (for navigation) and once in the leaf (holding the data).
3.  **Linked Leaves:** All leaf nodes are connected in a **Doubly Linked List**. This enables high-speed range scans.

### B. Internal Workings

#### 1. Search
* Traversal must continue all the way to the leaf level, even if the key matches an entry in an internal node (since internal nodes are just routers).

#### 2. Range Scan (The Superpower)
* **Query:** `Find users with Age > 25`
* **Process:**
    1.  The engine searches for `25`.
    2.  Once it hits the leaf node containing `25`, it ignores the tree structure above.
    3.  It follows the **"Next" pointer** in the linked list to retrieve `26, 27, 28...` sequentially.
* **Efficiency:** This creates a sequential read pattern on the disk, which is significantly faster than random seeking.

---

## 4. B-Tree vs. B+ Tree Comparison

| Feature | B-Tree | B+ Tree |
| :--- | :--- | :--- |
| **Data Storage** | Internal nodes & Leaves | Leaves Only |
| **Search Speed** | Fast for frequent keys (near root) | Consistent (always hit leaf) |
| **Range Queries** | Slow (must jump up/down tree) | **Very Fast** (follow linked list) |
| **Tree Height** | Higher (Nodes store bulky data) | **Lower** (Nodes store only small keys) |
| **Fan-out** | Lower | **Massive** |

### The "Fan-out" Advantage Explained
* **Scenario:** Disk block size is 4KB.
* **B-Tree:** If you store a 1KB JSON doc in internal nodes, a 4KB block fits only 4 items. Fan-out = 4. The tree becomes very deep.
* **B+ Tree:** Internal nodes only store the ID (e.g., 8 bytes) and a pointer (8 bytes). 16 bytes total.
    * $4096 / 16 = 256$ children per node.
    * With just 3 levels ($256 \times 256 \times 256$), you can index **16 million** documents.
    * **Result:** Any record is found in roughly **3 disk hops**.

---

## 5. Implementation in Real NoSQL Systems

### MongoDB (WiredTiger Engine)
* Uses **B+ Trees** for indexes.
* The `.wt` files are essentially pages of B+ Trees.
* Uses **Prefix Compression** in internal nodes to fit more keys, further increasing fan-out.

### Couchbase
* Uses an **HB+ Trie** (a hybrid of B+ Trees and Tries) for its Index Service to handle high concurrency and large key spaces efficiently.

### Amazon DynamoDB
* Uses B-Tree variants for local storage nodes (partitions) to manage data sorted by Partition Key + Sort Key.