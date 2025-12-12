# Internal Working of Skip Lists in Redis (ZSET)

This document details the internal engineering of the **Skip List**, the probabilistic data structure that powers **Redis Sorted Sets (ZSET)**, providing incredible performance for ranking and sorting tasks.

---

## 1. The Core Concept: Why not a Linked List?
To understand Skip Lists, we must first look at the limitations of a standard **Sorted Linked List**.

* **Scenario:** A sorted list: `1 -> 10 -> 20 -> 30 -> 40 -> 50`.
* **The Problem:** To find the value `50`, the engine must visit every single node ($O(N)$). It cannot "jump" to the middle because pointers only reference the immediate *next* node.
* **The Solution:** Add "express lanes" (additional pointers) that skip over multiple nodes to speed up traversal.

---

## 2. Internal Structure of a Skip List
A Skip List is a linked list with **multiple layers** of pointers.

> **[Diagram: Skip List structure showing multiple levels of pointers]**

### A. The Layers (Levels)
* **Level 0 (Bottom Lane):** A standard linked list containing **100%** of the elements. Guarantees reachability for all nodes.
* **Level 1 (Express Lane):** Contains roughly **50%** of the elements (skips 1 node).
* **Level 2 (Super Express):** Contains roughly **25%** of the elements (skips 3 nodes).
* **Level 3+:** Higher levels contain fewer elements, allowing massive jumps.

### B. The Node Structure
In Redis, a node contains an **array of pointers (Forward Pointers)** corresponding to its level height.
* `Node.forward[0]` $\rightarrow$ Next node in Level 0.
* `Node.forward[1]` $\rightarrow$ Next node in Level 1.

---

## 3. Internal Working: The Search Algorithm ($O(\log N)$)
**Query:** Find the value `30`.

1.  **Start at Top:** Begin at the "Head" node at the highest level (e.g., Level 2).
2.  **Scan Forward:** Look at the node `Head.forward[2]` points to.
    * **If Value < 30:** Move to that node (Advance).
    * **If Value > 30:** Don't move. Drop down one level.
3.  **Drop and Repeat:**
    * At Level 1, check the next pointer. Smaller? Move. Larger? Drop.
4.  **Target Reached:** Eventually, the traversal drops to Level 0 and lands exactly on `30`.

**Efficiency:** Instead of checking `1, 2, 3... 30`, the engine might check `1 -> 15 -> 25 -> 30`.

---

## 4. Internal Working: The Insertion (Write Path)
Skip Lists use a **Probabilistic** approach (Coin Flip) rather than strict structural rules (like B-Tree rotations).

**Goal:** Insert value `45`.

### Step 1: Locate Position
Use the search algorithm to find exactly where `45` fits in the bottom list (between `40` and `50`).

### Step 2: Determine Height (The Coin Flip)
How many "express lanes" should this new node have?
1.  Start at Level 1.
2.  Generate a random number (simulate a coin flip).
3.  **Heads:** Promote node to the next level. Flip again.
4.  **Tails:** Stop growing.

* **50%** of nodes stop at Level 0.
* **25%** reach Level 1.
* **12.5%** reach Level 2.

### Step 3: Rewire Pointers
Once the height is decided, Redis updates the pointers of the neighbors.
* *Old:* `40 -> 50`
* *New:* `40 -> 45 -> 50`
* **Advantage:** No global re-balancing is required, only local updates.

---

## 5. Redis Specific Optimizations (ZSET)
Redis implements a modified Skip List specifically for the `ZSET` data type.

### A. Backward Pointers (Doubly Linked)
Standard Skip Lists are singly linked. Redis adds a **Backward Pointer** at Level 0.
* **Why?** To support reverse iteration commands like `ZREVRANGE` (e.g., "Show me the top 10 players").

### B. Span (Rank Calculation)
Redis adds a `span` integer to every pointer.
* `span` = "How many nodes does this link skip?"
* **Working:** To find the **Rank** of a user, the engine sums the `span` values of all links traversed.
* **Result:** Rank calculation is $O(\log N)$, whereas in standard lists it is $O(N)$.

### C. Score vs. Value
ZSETs sort primarily by **Score** (floating point) and secondarily by **Value** (string).
* If Scores are tied, the engine compares the binary string of the Value to determine order.

---

## 6. Comparison: Skip List vs. B-Tree

| Feature | B-Tree (SQL/MongoDB) | Skip List (Redis) |
| :--- | :--- | :--- |
| **Complexity** | $O(\log N)$ | $O(\log N)$ |
| **Memory Usage** | Higher (Internal node overhead) | **Lower** (Just pointers) |
| **Implementation** | Complex (Rebalancing logic) | **Simple** (No rotations) |
| **Concurrency** | Hard (Locking complex nodes) | **Easier** (Local updates) |

**Why Redis chose Skip Lists:**
> *"They are not very memory intensive... and they are simpler to implement, debug, and so on."* — Salvatore Sanfilippo (Antirez)