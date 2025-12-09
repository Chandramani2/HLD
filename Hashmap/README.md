# HashMap Internals: Design & Implementation (Deep Dive)

## 1. High-Level Architecture
A HashMap is not a magic black box; it is essentially an **Array of Nodes** (often called "Buckets").

### The Components
1.  **The Bucket Array (`Node<K,V>[] table`):** The physical storage. It is an array where each index corresponds to a hash bucket.
2.  **The Node (Entry):** Each object stored is not just the Value. It is a linked-list node containing:
    * `final int hash;` (Cached hash code)
    * `final K key;`
    * `V value;`
    * `Node<K,V> next;` (Pointer to the next node in case of collision)



---

## 2. The Mechanics: PUT and GET

### Step 1: Hashing (The "Perturbation" Function)
When you call `map.put(key, value)`, the system does not just use `key.hashCode()`. It applies a supplemental hash function to defend against poor quality hash codes.

```java
// Java 8 Implementation
static final int hash(Object key) {
    int h;
    // XORs the higher 16 bits with the lower 16 bits
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

* **Why?** If the table size is small (e.g., 16), only the last 4 bits matter for the index. If `hashCode` only differs in the upper bits, collisions would be massive. The XOR shift spreads the entropy to the lower bits.

### Step 2: Index Calculation (Bitwise Magic)
How do we map a huge Hash Integer to a small Array Index (0 to 15)?
* **Junior Approach:** `index = hash % n` (Modulo is slow).
* **Senior Approach:** `index = hash & (n - 1)`.
    * This works **only** because the HashMap capacity is always a **Power of 2**.
    * Bitwise AND is significantly faster than Modulo arithmetic.

### Step 3: Collision Handling (Separate Chaining)
If two keys land on the same index:
1.  **Java 7:** It appends the new node to a **LinkedList**. Worst case lookup: **O(n)**.
2.  **Java 8+ (The Senior Detail):**
    * If the collision count in a single bucket exceeds **8 (TREEIFY_THRESHOLD)**...
    * AND the total array size is > 64...
    * The LinkedList converts into a **Red-Black Tree**.
    * **Impact:** Worst-case performance improves from **O(n)** to **O(log n)**. This prevents "Hash Flooding" DoS attacks.



---

## 3. Scaling: Resizing & Load Factor

HashMap is not infinite. It must grow.

* **Load Factor (`0.75`):** The trade-off between Time and Space.
    * If the map is 75% full, it triggers a resize.
    * *Lower (0.5):* Wastes space, fewer collisions.
    * *Higher (1.0):* Saves space, more collisions (slower lookup).

* **The Resize Process (Expensive):**
    1.  Create a new array with **double** the capacity ($OldCapacity \times 2$).
    2.  **Re-hashing:** Iterate through *every single node* in the old array and recalculate its position in the new array.
    3.  **Optimization:** Since size doubles (power of 2), a node simply stays at the same index `j` OR moves to `j + oldCap`. No complex math required.

---

## 4. Senior Interview Q&A

### Q1: Why must the Key be Immutable (e.g., String)?
**Senior Answer:**
"If the key is mutable and you change a field that affects the `hashCode()` *after* putting it in the map, you corrupt the data structure.
1.  You put `User(id=1)` $\to$ Hash is 100 $\to$ Bucket 5.
2.  You change `User.id` to 2 $\to$ New Hash is 200.
3.  When you call `get()`, HashMap looks for Hash 200 (Bucket 8), but the object is sitting in Bucket 5.
    Result: **Memory Leak** (object exists but is unreachable). Strings are immutable and cache their hash code, making them perfect keys."

### Q2: What happens if two Objects have the same HashCode but are not Equal?
**Senior Answer:**
"This is a **Hash Collision**.
1.  The HashMap calculates the index and finds a node already exists.
2.  It traverses the LinkedList/Tree at that bucket.
3.  It calls `key.equals(existingKey)` on every node.
4.  If `equals()` returns false for all, it appends the new node.
5.  If `equals()` returns true, it **overwrites** the value.
    *Takeaway: You must always override `equals()` and `hashCode()` together.*"

### Q3: Why is HashMap not Thread-Safe? What happens in a Race Condition?
**Senior Answer:**
"HashMap lacks synchronization. In a multi-threaded environment:
1.  **Lost Updates:** Two threads try to `put()` key A and key B into the same empty bucket simultaneously. One might overwrite the other without linking them.
2.  **Infinite Loop (Famous Java 7 Bug):** During resizing, if two threads try to re-hash the linked list at the same time, the pointers can get reversed, forming a cycle (`A.next = B` and `B.next = A`). Any future `get()` will cause the CPU to spike to 100% in an infinite loop.
    *Fix: Use `ConcurrentHashMap`.*"

### Q4: How does ConcurrentHashMap work differently?
**Senior Answer:**
"It doesn't lock the whole map (like `Hashtable` or `Collections.synchronizedMap`).
* **Java 7:** Used 'Segment Locking' (16 separate locks).
* **Java 8+:** Uses **CAS (Compare-And-Swap)** for optimistic updates on empty buckets and `synchronized` blocks **only on the specific head node** of a bucket for collisions. This allows extremely high concurrent read/write throughput."

### Q5: Can we use a Custom Object as a Key?
**Senior Answer:**
"Yes, but strictly adhering to the contract:
1.  **Override `hashCode()`:** Spread bits evenly to minimize collisions.
2.  **Override `equals()`:** Consistency with hashCode.
3.  **Immutability:** Ideally, make the fields used for hashing `final`.
4.  **Performance:** If the object is complex, cache the `hashCode` (like String does) so you don't recalculate it on every lookup."