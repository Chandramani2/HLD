# 🔎 The Senior Elasticsearch Query Playbook

> **Target Audience:** Search Engineers, Data Architects, & Backend Leads  
> **Goal:** Master **The Boolean Logic**, **Relevance Tuning (Function Score)**, and **Aggregations at Scale**.

In a senior interview, `match_all` is forbidden. You need to understand the difference between **Filter Context (Yes/No)** and **Query Context (How well?)**, and how to handle **Nested Objects** without flattening your data.

---

## 📖 Table of Contents
1. [Part 1: The Core Logic (Filter vs. Query Context)](#-part-1-the-core-logic-filter-vs-query-context)
2. [Part 2: The `bool` Query (The Swiss Army Knife)](#-part-2-the-bool-query-the-swiss-army-knife)
3. [Part 3: Aggregations (SQL Group By on Steroids)](#-part-3-aggregations-sql-group-by-on-steroids)
4. [Part 4: Relevance Tuning (Function Score)](#-part-4-relevance-tuning-function-score)
5. [Part 5: Senior Level Q&A Scenarios](#-part-5-senior-level-qa-scenarios)

---

## ⚡ Part 1: The Core Logic (Filter vs. Query Context)

The most important concept in ES performance.

### 1. Query Context ("Must")
* **Question:** "How *well* does this document match?"
* **Output:** Calculates a `_score` (BM25 algorithm).
* **Performance:** Slower (Math involved). Not cached.
* **Use for:** Full-text search (`title: "Star Wars"`).

### 2. Filter Context ("Filter")
* **Question:** "Does this document match? (Yes/No)"
* **Output:** No score (`_score: 0`).
* **Performance:** **Fast**. Results are cached in a Bitset (0/1).
* **Use for:** IDs, Dates, Status, Enums (`status: "published"`).

> **💡 Senior Rule:** If you don't need ranking by relevance, ALWAYS use `filter`. It saves CPU.

---

## 🧰 Part 2: The `bool` Query (The Swiss Army Knife)

Real-world queries are complex combinations of conditions.

| Clause | Logic | Context |
| :--- | :--- | :--- |
| **`must`** | AND | Query (Scoring) |
| **`filter`** | AND | Filter (Caching) |
| **`should`** | OR | Query (Scoring) |
| **`must_not`** | NOT | Filter (Caching) |

### Example: E-Commerce Search
*"Find shoes (Relevance) that are Red (Exact) and priced under $100 (Exact), but NOT Nike."*

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "running shoes" } }  // Calculates Score
      ],
      "filter": [
        { "term": { "color": "red" } },           // Exact Match (Cached)
        { "range": { "price": { "lte": 100 } } }  // Range (Cached)
      ],
      "must_not": [
        { "term": { "brand": "nike" } }
      ]
    }
  }
}
```

---

## 📊 Part 3: Aggregations (SQL `GROUP BY` on Steroids)

ES is also an analytics engine.

### 1. Bucket Aggregations (`GROUP BY`)
Groups documents into buckets.
* **Terms:** Top N items (`GROUP BY category`).
* **Date Histogram:** Time-series (`GROUP BY day`).

### 2. Metric Aggregations (`SUM`, `AVG`)
Calculates stats inside a bucket.

### Example: "Average Price per Brand"
```json
GET /products/_search
{
  "size": 0,  // We don't want the actual documents, just stats
  "aggs": {
    "brands_bucket": {
      "terms": { "field": "brand.keyword" }, // Bucket by Brand
      "aggs": {
        "avg_price": { "avg": { "field": "price" } } // Metric inside bucket
      }
    }
  }
}
```

---

## 🎯 Part 4: Relevance Tuning (Function Score)

**Interviewer:** *"Search results are technically correct but useless. Old articles appear before new ones."*

### 1. Boosting (`^`)
Simple boosting at query time.
* `title^2`: Matches in Title are 2x more important than Description.
* `match: { "title": { "query": "iphone", "boost": 2 } }`

### 2. Function Score Query (Decay Functions)
Complex math to alter scoring.
* **Scenario:** Show articles about "Tech" (Text Match), but boost **Recent** articles and **Popular** articles.
* **Gauss Decay:** "If the article is 1 day old, score is 1.0. If 30 days old, score is 0.5."

```json
GET /news/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "content": "tech" } }, // Base Score
      "functions": [
        {
          "gauss": {
            "publish_date": {
              "origin": "now",
              "scale": "10d",
              "decay": 0.5
            }
          }
        },
        {
          "field_value_factor": {
            "field": "view_count", // Boost by popularity
            "modifier": "log1p",   // Smooth the curve
            "factor": 0.1
          }
        }
      ]
    }
  }
}
```

---

## 🧠 Part 5: Senior Level Q&A Scenarios

### Scenario A: The "Exact Match" Trap
**Interviewer:** *"I indexed a document `{ "status": "Active User" }`. When I query `term: { "status": "Active User" }`, I get zero results. Why?"*

* **The Reason:** **Analysis**.
  * Standard Analyzer tokenizes "Active User" into `["active", "user"]`.
  * The `term` query looks for the *exact byte sequence* "Active User".
  * It doesn't match `active` or `user`.
* ✅ **Senior Answer:**
  * **Fix 1:** Use `match` query (which analyzes the input string too).
  * **Fix 2 (Better):** Use the `.keyword` sub-field (`status.keyword`), which stores the string as an exact un-analyzed token.

### Scenario B: Deep Pagination (Result 10,000)
**Interviewer:** *"Users complain that Page 1000 is loading slowly."*

* **The Problem:** `from: 10000, size: 10`.
  * Each shard must fetch 10,010 docs, sort them, and send them to the coordinator. The coordinator merges $N \times 10010$ docs. Massive Heap usage.
* ✅ **Senior Answer:** "**Search After (Cursor).**"
  * Don't jump to page 1000.
  * Get Page 1. Take the `sort` value of the last item.
  * Ask ES: "Give me 10 items *after* this sort value."
  * This is stateless and efficient ($O(1)$ vs $O(N)$).

### Scenario C: Nested Objects vs. Flattening
**Interviewer:** *"We have a blog post with comments: `[{ user: "Bob", stars: 1 }, { user: "Alice", stars: 5 }]`. Searching for 'Bob AND 5 stars' returns this post. That's wrong."*

* **The Reason:** ES flattens arrays. It sees `user: ["Bob", "Alice"]` and `stars: [1, 5]`. It matches "Bob" and "5" independently.
* ✅ **Senior Answer:** "**Nested Type.**"
  * Map `comments` as `type: "nested"`.
  * Use the `nested` query. This treats each comment as a separate hidden document, preserving the relationship between "Bob" and "1 star".

### Scenario D: Autocomplete (Search as you type)
**Interviewer:** *"How do we implement Google-style autocomplete?"*

* ❌ **Bad Answer:** `wildcard: "inp*"` (Extremely slow, scans all terms).
* ✅ **Senior Answer:** "**Edge N-Grams or Completion Suggester.**"
  * **Edge N-Grams (Index Time):**
    * Word: "Apple" -> Tokens: `["A", "Ap", "App", "Appl", "Apple"]`.
    * A query for "App" hits an exact term in the index. Instant speed.
  * **Completion Suggester:**
    * Optimized in-memory FST (Finite State Transducer). Fastest, but consumes Heap.

---

### **Final Checklist**
1.  **Context:** `filter` for exact/cache, `must` for relevance.
2.  **Analyzers:** Know when text is tokenized vs. `keyword`.
3.  **Pagination:** `search_after` > `from/size`.
4.  **Relevance:** `function_score` with decay for "Trending" feeds.

**This concludes the Elasticsearch Query Playbook.**