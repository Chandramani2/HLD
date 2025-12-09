# 🕸️ The Senior Graph Database Design Playbook

> **Target Audience:** Systems Architects & Database Engineers  
> **Goal:** Design a **Native Graph Storage Engine** from scratch (High Level & Low Level).

In a senior interview, the key differentiator is understanding **Index-Free Adjacency**. In a Relational DB, joining is a Search ($O(\log N)$). In a Graph DB, joining is a Pointer Dereference ($O(1)$).

---

## 📖 Table of Contents
1. [Part 1: High-Level Architecture (The Engine)]( #-part-1-high-level-architecture-the-engine)
2. [Part 2: Data Structures (Fixed-Size Records)]( #-part-2-data-structures-fixed-size-records)
3. [Part 3: Low-Level Code (The Node & Relationship Class)]( #-part-3-low-level-code-the-node--relationship-class)
4. [Part 4: The Traversal Algorithm (BFS)]( #-part-4-the-traversal-algorithm-bfs)
5. [Part 5: Handling Supernodes (The Edge Case)]( #-part-5-handling-supernodes-the-edge-case)

---

## 🏗️ Part 1: High-Level Architecture (The Engine)

A GraphDB consists of three files on the disk and a memory manager.

### 1. The Components
1.  **Node Store:** Stores the vertices (Entities).
2.  **Relationship Store:** Stores the edges (Connections).
3.  **Property Store:** Stores Key-Value data (e.g., `name="Alice"`).
4.  **Index (B-Tree):** Maps `String` ("Alice") -> `Node ID` (105). *Used ONLY to find the starting point.*

### 2. The Logic Flow
1.  **Index Seek:** User asks for "Alice". B-Tree returns `NodeID: 105`.
2.  **Loader:** System calculates file offset for ID 105 (`105 * Node_Record_Size`). Reads bytes.
3.  **Traversal:** The Node Record contains a pointer to the **First Relationship ID**.
4.  **Hop:** System calculates file offset for that Relationship ID. Reads bytes.
5.  **Next Hop:** The Relationship Record contains pointers to the **Next Relationship**.



[Image of Graph Database Architecture Diagram]


---

## 💾 Part 2: Data Structures (Fixed-Size Records)

Why is GraphDB fast? **Pointer Arithmetic.**
If every node record is exactly **15 Bytes**, we don't need to scan the file to find Node 1000. We just jump to byte `15,000`.

### 1. The Node Record (e.g., 15 Bytes)
| Field | Size | Description |
| :--- | :--- | :--- |
| **InUse** | 1 Byte | Is this record deleted? |
| **NextRelID** | 4 Bytes | **Pointer** to the first relationship connected to this node. |
| **PropID** | 4 Bytes | Pointer to the first property (Key-Value). |
| **Label** | 5 Bytes | Internal ID for label (e.g., `:Person`). |

### 2. The Relationship Record (e.g., 34 Bytes)
This is a **Doubly Linked List** stored on disk.

| Field | Size | Description | 
| :--- | :--- | :--- |
| **InUse** | 1 Byte | Deleted flag. |
| **StartNode** | 4 Bytes | ID of the source node. |
| **EndNode** | 4 Bytes | ID of the target node. |
| **StartPrev** | 4 Bytes | Ptr to previous edge on Start Node. |
| **StartNext** | 4 Bytes | Ptr to next edge on Start Node. |
| **EndPrev** | 4 Bytes | Ptr to previous edge on End Node. |
| **EndNext** | 4 Bytes | Ptr to next edge on End Node. |

> **💡 Senior Insight:**
> This structure allows us to walk the chain of edges for *any* node without ever looking at an index. This is **Index-Free Adjacency**.



---

## 💻 Part 3: Low-Level Code (The Node & Relationship Class)

Here is a simplified Python/C++ pseudocode representation of the in-memory structures.

```python
class Node:
    def __init__(self, id):
        self.id = id
        self.first_relationship_id = None # Pointer to the head of the LL
        self.properties = {}

class Relationship:
    def __init__(self, id, start_node_id, end_node_id, type):
        self.id = id
        self.start_node_id = start_node_id
        self.end_node_id = end_node_id
        self.type = type
        
        # The Doubly Linked List Pointers
        # These allow us to traverse edges attached to the Start Node
        self.start_node_prev_id = None
        self.start_node_next_id = None
        
        # These allow us to traverse edges attached to the End Node
        self.end_node_prev_id = None
        self.end_node_next_id = None

class GraphEngine:
    def __init__(self):
        # In reality, these are ByteArrays on Disk (mmap)
        self.nodes = {} 
        self.relationships = {} 

    def get_node(self, node_id):
        # O(1) lookup
        return self.nodes[node_id]

    def get_relationship(self, rel_id):
        # O(1) lookup
        return self.relationships[rel_id]
```

---

## 🏃 Part 4: The Traversal Algorithm (BFS)

How do we implement: `MATCH (n)-[:FRIEND]->(m) RETURN m`?

We simply walk the Linked List of relationships attached to the node.

```python
def get_neighbors(self, start_node_id, rel_type):
    neighbors = []
    
    # 1. Get the Node Record
    node = self.get_node(start_node_id)
    
    # 2. Get the first edge (Head of the list)
    current_rel_id = node.first_relationship_id
    
    # 3. Walk the Linked List
    while current_rel_id is not None:
        rel = self.get_relationship(current_rel_id)
        
        # Check if this relationship matches the Type (e.g., FRIEND)
        if rel.type == rel_type:
            # Identify the "Other" node
            if rel.start_node_id == start_node_id:
                neighbors.append(rel.end_node_id)
                # Move to next edge in Start Node's chain
                current_rel_id = rel.start_node_next_id
            else:
                neighbors.append(rel.start_node_id)
                # Move to next edge in End Node's chain
                current_rel_id = rel.end_node_next_id
        else:
            # Move next regardless (Skip this type)
            if rel.start_node_id == start_node_id:
                current_rel_id = rel.start_node_next_id
            else:
                current_rel_id = rel.end_node_next_id
                
    return neighbors
```

**Complexity Analysis:**
* **GraphDB:** $O(k)$ where $k$ is the number of friends.
* **SQL:** $O(\log N)$ to find the rows in the B-Tree index + $O(k)$ to fetch them.

---

## 🧠 Part 5: Handling Supernodes (The Edge Case)

**The Problem:**
If "Katy Perry" has 100 Million followers, her `Node Record` points to a Linked List of 100 Million `Relationship Records`.
Walking this list (linear scan) just to find "Followers who are also Presidents" is slow.

**The Senior Solution: Vertex-Centric Indexes**
* Instead of one giant linked list, we split the relationships into **buckets** based on Type or Property.
* **Structure:**
    * Katy Perry Node -> Ptr to **Index Block**.
    * Index Block:
        * `Type: FOLLOWS, Region: USA` -> Ptr to Linked List A (10k items).
        * `Type: FOLLOWS, Region: UK` -> Ptr to Linked List B (5k items).
* **Result:** When querying "Katy Perry followers in UK", we skip millions of USA followers entirely.

---

## 🧪 Part 6: Senior Level Q&A Scenarios

### Scenario A: Deletion (Linked List Repair)
**Interviewer:** *"What happens when we delete a Relationship?"*

* ✅ **Senior Answer:**
    * We must update **Four Pointers**.
    * 1. Go to the `StartNode`, find the `prev` relationship, point its `next` to the deleted relationship's `next`.
    * 2. Go to the `EndNode`, do the same.
    * 3. Mark the Relationship Record as `InUse = False` (Soft Delete) so the space can be reused by a new relationship later (Free List).

### Scenario B: Distributed Graphs (Sharding)
**Interviewer:** *"The graph is too big for one disk. How do we shard it?"*

* ✅ **Senior Answer:** "**The Edge Cut vs. Vertex Cut Problem.**"
    * **Edge Cut:** Put Node A on Server 1, Node B on Server 2. The Relationship spans the network.
        * *Con:* Traversals are slow (Network Hopping).
    * **Vertex Cut:** Keep the Relationship on one server, but split the *Node* across servers.
    * **Reality:** Most high-performance GraphDBs (Neo4j) are **not sharded** (Scale Up). They use Read Replicas (Causal Clustering). If you strictly need Sharding, use **JanusGraph** (on top of Cassandra) but accept slower traversals.

### Scenario C: Consistency
**Interviewer:** *"Is a GraphDB ACID?"*

* ✅ **Senior Answer:**
    * **Yes (Neo4j):** It uses a Write-Ahead Log (WAL) and locking.
    * Because relationships are physical pointers, you cannot have a "dangling pointer." The Transaction Manager ensures that creating a node and linking it happens atomically.

---

### **Final Checklist**
1.  **Storage:** Fixed-size records allow $O(1)$ offset calculation.
2.  **Structure:** Relationships are **Doubly Linked Lists** on disk.
3.  **Performance:** Traversal speed depends on the *degree of the node*, not the size of the DB.
4.  **Limits:** Supernodes require specialized local indexing.

**This concludes the GraphDB Internals Playbook.**