# 🕸️ The Senior Graph Query Playbook

> **Target Audience:** Data Engineers, Recommendations Engineers, & Backend Leads  
> **Goal:** Master **Cypher (Neo4j)**, **Gremlin (Apache)**, **Variable Length Paths**, and **Query Profiling**.

In a senior interview, you must demonstrate how to **Anchor** your search efficiently, how to aggregate paths using `COLLECT`, and how to differentiate between **Declarative** and **Imperative** graph traversals.

---

## 📖 Table of Contents
1. [Part 1: The Languages (Cypher vs. Gremlin)](#-part-1-the-languages-cypher-vs-gremlin)
2. [Part 2: Pattern Matching (Drawing with Text)](#-part-2-pattern-matching-drawing-with-text)
3. [Part 3: Variable Length Paths (Recursion)](#-part-3-variable-length-paths-recursion)
4. [Part 4: Aggregation & Performance (Collect & Unwind)](#-part-4-aggregation--performance-collect--unwind)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🗣️ Part 1: The Languages (Cypher vs. Gremlin)

Just like SQL dominates relational, these two dominate Graph.

| Feature | **Cypher (Neo4j, Memgraph)** | **Gremlin (CosmosDB, Neptune, Janus)** |
| :--- | :--- | :--- |
| **Paradigm** | **Declarative** (Like SQL). You describe *what* pattern to find. | **Imperative** (Like Java/Groovy). You describe *how* to traverse step-by-step. |
| **Readability** | High (ASCII Art). | Medium (Method chaining). |
| **Use Case** | Data Analysis, Whiteboarding, Rapid Dev. | Application Integration, Complex algorithmic logic. |

### The Comparison
**Goal:** "Find the names of Alice's friends."

**Cypher:**
```cypher
MATCH (p:Person {name: "Alice"})-[:FRIEND]->(f:Person)
RETURN f.name
```

**Gremlin:**
```groovy
g.V().has('name', 'Alice').out('FRIEND').values('name')
```

---

## 🎨 Part 2: Pattern Matching (Drawing with Text)

In Cypher, you "draw" the graph structure in the query.

### 1. Anatomy of a Pattern
* **`()`**: A Node.
* **`[]`**: A Relationship.
* **`->`**: Direction.

**Pattern:** `(User)-[Purchase]->(Product)`

### 2. Filtering in the Pattern
Don't use `WHERE` clauses if you can filter inline. This allows the engine to pick the correct index immediately (Index Seek).

* **Bad:** `MATCH (p:Person) WHERE p.id = 123` (Scans all people).
* **Good:** `MATCH (p:Person {id: 123})` (Uses Index on ID).

### 3. Complex Shapes
**Scenario:** "Find Movie Recommendation." (Users who bought what I bought, also bought X).

```cypher
MATCH (me:User {id: 1})
      -[:BOUGHT]->(commonProduct:Product)
      <-[:BOUGHT]-(otherUser:User)
      -[:BOUGHT]->(recommendation:Product)
WHERE NOT (me)-[:BOUGHT]->(recommendation)
RETURN recommendation.name, count(*) as frequency
ORDER BY frequency DESC
```

---

## 🌀 Part 3: Variable Length Paths (Recursion)

This is the killer feature of GraphDBs. SQL cannot do this efficiently.

### 1. Unbounded Traversal
**Scenario:** "Find all parts required to build a Car (Sub-parts of sub-parts...)."

```cypher
MATCH (car:Part {name: "Car"})-[:CONTAINS*]->(subPart)
RETURN subPart.name
```
* The `*` means: Keep going until you hit a leaf node.

### 2. Bounded Traversal (Safety)
**Scenario:** "Find extended network (Friend of Friend) up to 3 hops."

```cypher
MATCH (me:Person)-[:FRIEND*1..3]->(extendedNetwork)
RETURN extendedNetwork.name
```
* **Senior Tip:** Never run `*` (unbounded) on a massive graph (e.g., Twitter). It will traverse the whole web and crash the server. Always set a limit (`*1..5`).

### 3. Shortest Path
**Scenario:** "How is Kevin Bacon connected to Tom Hanks?"

```cypher
MATCH path = shortestPath(
    (p1:Person {name:"Kevin Bacon"})-[*]-(p2:Person {name:"Tom Hanks"})
)
RETURN path
```
* Algorithms like **Dijkstra** or **A*** run internally.

---

## 📦 Part 4: Aggregation & Performance (Collect & Unwind)

Graph aggregation is different from SQL `GROUP BY`.

### 1. `COLLECT` (Fold)
Takes multiple rows and collapses them into a **List (Array)** within a single row.

**Scenario:** "Return one row per Director, with a list of their movies."
```cypher
MATCH (d:Director)-[:DIRECTED]->(m:Movie)
RETURN d.name, collect(m.title) as movies
```
* **Result:** `{"d.name": "Nolan", "movies": ["Inception", "Tenet", "Dunkirk"]}`

### 2. `UNWIND` (Unfold)
Takes a List and expands it into multiple rows (Like `flatMap`).

**Scenario:** "Input is a list of UserIDs. Find them all."
```cypher
UNWIND [1, 2, 3] AS uid
MATCH (u:User {id: uid})
RETURN u.name
```

### 3. Profiling (DbHits)
**Interviewer:** *"This query is slow. Fix it."*
* Run `PROFILE MATCH ...`
* Look at **DbHits**.
* **Cartesian Product:** Did you disconnected patterns?
  * `MATCH (a), (b)` -> This matches EVERY node against EVERY node ($N^2$).
  * Always connect patterns: `MATCH (a)-->(b)`.

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Supply Chain Impact Analysis
**Interviewer:** *"A factory in China shutdown. Which products sold in the USA are affected?"*

* **Data Model:** `(Factory)-[:MAKES]->(Part)-[:USED_IN]->(Product)`
* ✅ **Senior Answer:** "**Upstream Traversal.**"
    ```cypher
    MATCH (f:Factory {location: "China"})-[:MAKES|USED_IN*]->(p:Product)
    WHERE (p)-[:SOLD_IN]->(:Location {name: "USA"})
    RETURN DISTINCT p.name
    ```
  * Usage of `|` (OR) allows traversing different relationship types in one chain.
  * `DISTINCT` is crucial because a product might use 5 parts from that factory; we only want to list the product once.

### Scenario B: Authorization (ACLs)
**Interviewer:** *"Can User A read File X? They might inherit permission via groups, or groups of groups."*

* ✅ **Senior Answer:** "**Path Existence Check.**"
    ```cypher
    MATCH (u:User {name: "A"}), (f:File {name: "X"})
    RETURN EXISTS( (u)-[:MEMBER_OF*]->(:Group)-[:HAS_ACCESS]->(f) )
    ```
  * `EXISTS` is highly optimized. It stops searching as soon as it finds **one** path. It doesn't find *all* paths.

### Scenario C: Deleting a Supernode
**Interviewer:** *"Delete 'Katy Perry' (User with 100M relationships)."*

* **The Problem:** `MATCH (u {name: "Katy"}) DETACH DELETE u` creates one massive transaction. It will blow the Heap Memory trying to load 100M relationships into RAM to delete them.
* ✅ **Senior Answer:** "**Batch Deletion (APOC).**"
  * Use `apoc.periodic.iterate` (Neo4j plugin) or batch manual commits.
  * "Find first 10,000 edges -> Delete -> Commit -> Repeat."

### Scenario D: Fraud Detection (Spider Web)
**Interviewer:** *"Find users who share the same Phone, Email, OR IP address."*

* ✅ **Senior Answer:** "**The Weakly Connected Component.**"
    ```cypher
    MATCH (u1:User)-[:HAS_IP|HAS_EMAIL|HAS_PHONE]->(identifier)<-[:HAS_IP|HAS_EMAIL|HAS_PHONE]-(u2:User)
    RETURN u1, u2, identifier
    ```
  * This finds shared resources instantly.
  * If `u1` and `u2` share an IP, and `u2` and `u3` share an Email, graph traversal reveals they are all the same person (Fraud Ring).

---

### **Final Checklist**
1.  **Variable Length:** Use `*` for recursion, but limit hops (`*1..5`) in production.
2.  **Aggregation:** `collect()` creates lists, `UNWIND` flattens them.
3.  **Performance:** `PROFILE` your queries. Watch out for Cartesian Products (`MATCH (a), (b)`).
4.  **Syntax:** Know the difference between `()` (Node) and `[]` (Edge).

**This concludes the GraphDB Query Playbook.**