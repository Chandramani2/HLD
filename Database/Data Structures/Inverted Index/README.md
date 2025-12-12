# Internal Working of Inverted Indices in NoSQL Databases

This document details the internal engineering of the **Inverted Index**, the core data structure that powers search capabilities in NoSQL systems like **Elasticsearch**, **Apache Solr**, and **MongoDB** (Text Search).

---

## 1. The Core Concept: Inverting the Relationship
To understand the Inverted Index, compare it to a standard database index (Forward Index).

### Forward Index (Standard DB)
* **Maps:** `Document ID` $\rightarrow$ `Content`
* **Structure:** "Doc 1 contains words [apple, banana, cherry]"
* **Problem:** To find the word "banana", the engine must scan **every** document (Full Table Scan). This is $O(N)$, which is too slow for large datasets.

### Inverted Index (Search Engine)
* **Maps:** `Content` $\rightarrow$ `Document IDs`
* **Structure:** "The word 'Banana' is found in [Doc 1, Doc 5, Doc 9]"
* **Result:** To find "banana", the engine jumps straight to the list. The complexity is near $O(1)$ regarding the total number of documents.

> **[Diagram: Visual comparison between Forward Index mapping and Inverted Index mapping]**

---

## 2. Internal Components
An Inverted Index consists of two primary internal structures:

### A. The Term Dictionary (Vocabulary)
A sorted list of every unique word (term) that appears across the entire dataset.
* **Storage:** Usually kept in memory (RAM) or memory-mapped files.
* **Structure:** Often stored as a **B-Tree** or **FST (Finite State Transducer)** to allow rapid prefix lookups (e.g., finding words starting with "data*").

### B. The Postings List
For every term in the dictionary, there is a list of data attached to it called the **Postings List**.
* **Basic Version:** A simple list of Document IDs.
    * `"database": [1, 5, 20, 99]`
* **Production Version:** Stores metadata for relevance scoring:
    * **Term Frequency (TF):** How often the word appears in the doc (for scoring).
    * **Positions:** Location of the word (e.g., "word 5"). Required for **Phrase Search** (e.g., "United" followed immediately by "States").
    * **Offsets:** Character start/end indices (for highlighting matches).

---

## 3. Internal Working: The Write Path (Index Construction)
When a document is inserted, it goes through an **Analysis Pipeline** before being written to the inverted index.

**Example Document:** `Doc 1: "The quick brown fox jumps."`

### Step 1: Character Filtering
Strips out HTML tags or converts special characters.
* *Input:* `<b>The</b>` $\rightarrow$ *Output:* `The`

### Step 2: Tokenization
Breaks the text string into individual terms (tokens).
* *Input:* `"The quick brown fox"`
* *Output:* `[The, quick, brown, fox]`

### Step 3: Token Filtering (Normalization)
Standardizes text so variations match the same query.
* **Lowercasing:** `The` $\rightarrow$ `the`
* **Stop Word Removal:** Removes common words like `the`, `is` (optional).
* **Stemming:** Reduces words to their root form.
    * `jumps` $\rightarrow$ `jump`

### Step 4: Indexing
The engine adds the Document ID to the Postings List for every generated token.

| Term | Postings List (Doc IDs) |
| :--- | :--- |
| `brown` | `[1]` |
| `fox` | `[1]` |
| `jump` | `[1]` |
| `quick` | `[1]` |

---

## 4. Internal Working: The Read Path (Search Execution)
The speed of the inverted index comes from **Set Theory** (Intersections and Unions).

**Query:** `Find documents with "Brown" AND "Fox"`

1.  **Lookup:** Engine finds "brown" in the Term Dictionary.
    * Result: `[Doc 1, Doc 5, Doc 8]`
2.  **Lookup:** Engine finds "fox" in the Term Dictionary.
    * Result: `[Doc 1, Doc 3]`
3.  **Intersection:** Engine performs a set intersection on the two lists.
    * `[1, 5, 8]` $\cap$ `[1, 3]` = `[1]`
4.  **Result:** Document 1 is returned.

### Optimization: Skip Lists
If postings lists are massive (millions of IDs), linear intersection is slow. Engines use **Skip Lists** inside the postings list to jump over large chunks of Doc IDs that cannot possibly match.

---

## 5. Compression: Frame of Reference (Delta Encoding)
To reduce disk usage, engines use **Delta Encoding** for Doc IDs.

Instead of storing raw IDs: `[100, 105, 108]`
The index stores the *difference* (delta) between the current ID and the previous one:
* Start: `100`
* Next: `5` (105 - 100)
* Next: `3` (108 - 105)
* **Stored Data:** `[100, 5, 3]`

Small numbers require fewer bits, allowing massive compression (using algorithms like **PForDelta** or **BitPacking**).

---

## 6. Usage in NoSQL Systems

| Database | Usage |
| :--- | :--- |
| **Elasticsearch** | Built entirely on **Apache Lucene** (a library of Inverted Indices). |
| **Apache Solr** | Also built on Lucene. Identical internal structure. |
| **MongoDB** | Uses Inverted Indices for its specific **Text Search** feature. |
| **Cassandra** | Uses **SASI** (SSTable Attached Secondary Index) for text matching. |