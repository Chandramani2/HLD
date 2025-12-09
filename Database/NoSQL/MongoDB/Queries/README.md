# 🍃 The Senior MongoDB Query Playbook

> **Target Audience:** Backend Engineers, Data Architects, & NoSQL Developers  
> **Goal:** Master **Aggregation Pipelines**, **Complex Array Operations**, and **Query Performance Analysis**.

In a senior interview, you aren't just fetching JSON. You are reshaping data on the fly, performing **Graph Traversals** inside the DB, and optimizing indexes using the **ESR Rule**.

---

## 📖 Table of Contents
1. [Part 1: The Aggregation Pipeline (SQL to Mongo Mapping)](#-part-1-the-aggregation-pipeline-sql-to-mongo-mapping)
2. [Part 2: Advanced Array Operations ($unwind & $elemMatch)](#-part-2-advanced-array-operations-unwind--elemmatch)
3. [Part 3: Performance Tuning (The ESR Rule)](#-part-3-performance-tuning-the-esr-rule)
4. [Part 4: Powerful Stages ($facet & $graphLookup)](#-part-4-powerful-stages-facet--graphlookup)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## 🏗️ Part 1: The Aggregation Pipeline (SQL to Mongo Mapping)

The Aggregation Framework is a conveyor belt. Documents enter, pass through stages, and exit.

**The Golden Rule:** Filter early. Always put `$match` as the first stage to reduce the dataset.

| SQL Concept | MongoDB Stage | Description |
| :--- | :--- | :--- |
| `WHERE` | **`$match`** | Filters documents. |
| `GROUP BY` | **`$group`** | Aggregates data (Sum, Avg). |
| `SELECT` | **`$project`** | Reshapes the doc (Rename/Remove fields). |
| `JOIN` | **`$lookup`** | Pulls in data from another collection. |
| `HAVING` | **`$match`** | Filters *after* a `$group` stage. |

### Example: "Find total sales per product for 2023."

**Table: `orders`**

| _id | product | total | status | date |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Laptop | 1000 | paid | 2023-01-01 |
| 2 | Mouse | 50 | paid | 2023-02-01 |
| 3 | Laptop | 1000 | returned | 2023-03-01 |

```javascript
db.orders.aggregate([
  // 1. Filter first (Index friendly)
  { $match: { 
      status: "paid",
      date: { $gte: new Date("2023-01-01") } 
  }},
  // 2. Group
  { $group: {
      _id: "$product",
      totalRevenue: { $sum: "$total" },
      count: { $sum: 1 }
  }},
  // 3. Sort (Highest revenue first)
  { $sort: { totalRevenue: -1 } }
])
```

---

## 🧬 Part 2: Advanced Array Operations (`$unwind` & `$elemMatch`)

SQL struggles with arrays. MongoDB thrives on them, but they are tricky.

### 1. Flattening Arrays (`$unwind`)
**Scenario:** "We have a Blog Post with an array of Tags. We want to count how many posts exist for *each* tag."

**Document:**
```json
{ "_id": 1, "title": "Mongo Tips", "tags": ["db", "coding"] }
```

**Query:**
```javascript
db.posts.aggregate([
  { $unwind: "$tags" }, // Splits doc into 2 docs: one for 'db', one for 'coding'
  { $group: {
      _id: "$tags",
      count: { $sum: 1 }
  }}
])
```

### 2. Matching Inside Arrays (`$elemMatch`)
**Scenario:** "Find students who scored > 80 in Math."
**Document:**
```json
{ 
  "name": "Alice", 
  "scores": [ 
    { "subject": "Math", "val": 90 }, 
    { "subject": "English", "val": 70 } 
  ] 
}
```

* **Wrong:** `find({ "scores.subject": "Math", "scores.val": { $gt: 80 } })`
  * *Why:* This matches if *any* subject is Math and *any* score is > 80 (even if it's English).
* **Correct:**
```javascript
db.students.find({
  scores: { 
    $elemMatch: { subject: "Math", val: { $gt: 80 } } 
  }
})
```

---

## 🏎️ Part 3: Performance Tuning (The ESR Rule)

**Interviewer:** *"Why is this query slow?"*

### 1. The ESR Rule (Equality, Sort, Range)
When creating a compound index, order matters.
1.  **Equality:** Place fields you match exactly first (`status: "A"`).
2.  **Sort:** Place fields you sort by next (`date: -1`).
3.  **Range:** Place fields you range filter last (`price: { $gt: 10 }`).

**Index:** `{ status: 1, date: -1, price: 1 }`

### 2. Covered Queries
The fastest query is one that **never touches the document**.
* If your Index contains all the fields you need, Mongo returns data directly from the Index (RAM).
* **Requirement:** You must explicitly exclude `_id` unless it's in the index.

```javascript
// Index: { username: 1 }
db.users.find(
  { username: "john" }, 
  { username: 1, _id: 0 } // Projection
)
// ExecutionStats: "totalDocsExamined": 0 (Pure Index Scan)
```

---

## 💎 Part 4: Powerful Stages (`$facet` & `$graphLookup`)

### 1. Single-Pass Analytics (`$facet`)
**Scenario:** "On an E-commerce search page, show results + counts by Brand + counts by Color."
* **Naive:** Run 3 separate queries. (Slow).
* **Senior:** Use `$facet` to run parallel pipelines on the same input.

```javascript
db.products.aggregate([
  { $match: { category: "Shoes" } },
  { $facet: {
      "brands": [ { $group: { _id: "$brand", count: { $sum: 1 } } } ],
      "colors": [ { $group: { _id: "$color", count: { $sum: 1 } } } ],
      "top_products": [ { $sort: { price: -1 } }, { $limit: 5 } ]
  }}
])
```

### 2. Recursive Search (`$graphLookup`)
**Scenario:** "Find the Org Chart hierarchy (Manager of Manager)."
* Mongo can do recursive graph traversal!

```javascript
db.employees.aggregate([
  { $match: { name: "Alice" } },
  { $graphLookup: {
      from: "employees",
      startWith: "$manager_id",
      connectFromField: "manager_id",
      connectToField: "_id",
      as: "management_chain"
  }}
])
```

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: Pagination Optimization
**Interviewer:** *"We have 10 million logs. `skip(500000).limit(10)` is taking 5 seconds."*

* **Analysis:** `skip()` forces the database to iterate 500k docs and discard them.
* ✅ **Senior Answer:** "**Keyset Pagination ($gt).**"
    ```javascript
    // Instead of Page 5000, ask for data AFTER the last ID you saw.
    db.logs.find({ _id: { $gt: last_seen_id } }).limit(10)
    ```

### Scenario B: Updating a Specific Array Item
**Interviewer:** *"A user updated their comment inside a post. The post contains an array of 100 comments. How do we update just that one?"*

* **Document:** `{ comments: [ {id: 1, text: "A"}, {id: 2, text: "B"} ] }`
* ✅ **Senior Answer:** "**The Positional Operator ($).**"
    ```javascript
    db.posts.updateOne(
      { "comments.id": 2 }, 
      { $set: { "comments.$.text": "Updated Text" } }
    )
    ```
  * The `$` acts as a placeholder for the index of the element that matched the query.

### Scenario C: Moving Data (ETL)
**Interviewer:** *"We need to archive old orders to a separate collection named `orders_archive`."*

* ✅ **Senior Answer:** "**$merge (Materialized View).**"
    ```javascript
    db.orders.aggregate([
      { $match: { date: { $lt: new Date("2022-01-01") } } },
      { $merge: { into: "orders_archive", whenMatched: "replace" } }
    ])
    ```
  * `$merge` (introduced in Mongo 4.2) allows an aggregation to write results out to a collection.

### Scenario D: "Explain" Analysis
**Interviewer:** *"This query is slow. Debug it."*

* ✅ **Senior Answer:** Run `explain("executionStats")`.
  * **Look for `totalKeysExamined` vs `totalDocsExamined`.**
  * **Ideal:** Keys == Docs (Index used perfectly).
  * **Bad:** Keys << Docs (Index missed, collection scan occurred).
  * **Terrible:** `COLLSCAN` stage (No index used at all).

---

### **Final Checklist**
1.  **Pipeline:** Match first, Project last.
2.  **Performance:** Use ESR Rule for indexes.
3.  **Arrays:** Know when to `$unwind` and when to `$elemMatch`.
4.  **Complex Logic:** Use `$facet` for dashboards.

**This concludes the MongoDB Query Playbook.**