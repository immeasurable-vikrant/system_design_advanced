# 05 — Database Indexing: Complete Deep Dive 

> Ye original "Database Indexing - Complete Deep Dive" article hai, but har dense/jargon-heavy section ke turant baad ek **"🔍 Explained"** box dala hai jisme wahi cheez frontend-engineer-friendly tarike se, examples ke saath samjhayi gayi hai. Ab tumhe do alag files switch nahi karni — sab ek hi jagah hai, order me.

Prerequisites: `01-Hashing-Basics-Explained.md` (Hash Index section ke liye)
Used in: All system designs that use databases (every design on this site)

---

## What is an Index?

An index is a data structure that makes queries faster by avoiding full table scans. Instead of reading every row, the database uses the index to jump directly to matching rows.

**Real-world analogy:** A textbook index at the back of the book. Want to find "Binary Search"? Without an index, you'd flip through every page. With an index, you look up "Binary Search → page 142" and go directly there.

```
Without index (full table scan):
  Table: 10 million rows
  Query: SELECT * FROM users WHERE email = 'bob@mail.com'
  Operation: Scan all 10M rows, check each → O(N) → SLOW (seconds)

With index on email:
  Index: sorted structure mapping email → row location
  Query: SELECT * FROM users WHERE email = 'bob@mail.com'
  Operation: Binary search in index → O(log N) → FAST (milliseconds)
```

> ### 🔍 Explained: "row" aur "disk read" ka matlab
> **Row** = table ki ek single entry (jaise ek user): `{id: 1, name: "Bob", age: 25}`. Full table scan = database har aisi 10 million lines ko ek-ek karke check karta hai, chahe usse chahiye sirf 1 match.
>
> **Disk read itni badi deal kyu hai:** Frontend me data mostly **RAM** me hota hai (JS objects, React state) — RAM access ~100 nanoseconds. Database apna data **disk** pe rakhta hai (persistence ke liye — crash ke baad bhi data bacha rahe, yehi "Durability" hai). Disk access RAM se **~1000x slower** hota hai (SSD ~100,000 ns). Isliye database design ka core goal hai: **jitna kam ho sake utne kam disk reads karo.**
>
> Analogy: disk read for a database = network call for a frontend app — dono hi "expensive resources" hai jinhe minimize karna hai.

---

## How B-Tree Index Works (Default in Postgres/MySQL)

The most common index type. A balanced tree structure where:
- Each node holds multiple keys (sorted)
- Leaf nodes point to actual table rows
- Tree stays balanced (all leaves at same depth)

```
Root: 30 - 60
10 - 20 - 25          35 - 42 - 55          65 - 78 - 90
Leaf: data pointers

Query: WHERE age = 42
  Root: 42 > 30, 42 < 60 → go middle child
  Internal: 42 > 35, 42 = 42 → found!
  Leaf: points to row → fetch row data

Depth = ~3-4 for millions of rows
Each lookup = 3-4 disk reads = milliseconds
```

> ### 🔍 Explained: Node kya hai, key kya hai, aur ek concrete example
> **Node** = B-Tree ka ek disk block/page (ek chhota "box") jisme **100-500 sorted key entries** fit ho jaate hai — na ki sirf 1 key per node (jaise normal binary tree). Isi wajah se tree "wide" hota hai, aur depth (levels) bahut kam lagte hai:
> ```
> 1 million rows  → depth ~3
> 1 billion rows  → depth ~4-5 (barely increases!)
> ```
> Ye **logarithmic growth (O(log N))** hai — data 1000x badhe, depth sirf 1-2 level badhta hai.
>
> **Key** ek indexed column ki value hai (jaise `age=25`), jiske saath ek **pointer** hota hai jo batata hai row disk pe kaha padi hai. Key khud data nahi hai, sirf ek signpost hai.
>
> **Concrete example — `users` table:**
> ```
> Rows: {id:1, name:"Bob", age:25}, {id:2, name:"Alice", age:30}, {id:3, name:"Eve", age:25}
>
> Index on age:
>   key=25 → pointers: [row_id 1, row_id 3]   (Bob aur Eve dono age=25)
>   key=30 → pointer:  [row_id 2]             (Alice)
> ```
> **Tree form me:**
> ```
>                     [ age=27 ]                 ← root node
>                    /            \
>           [age=25]              [age=30]       ← leaf nodes (sorted keys)
>           pointers:              pointers:
>           → row_id 1 (Bob)       → row_id 2 (Alice)
>           → row_id 3 (Eve)
> ```
> Query `WHERE age=25`: root pe `25 < 27` → left child pe jao → `key=25` mila → seedha row_id 1, 3 fetch karo. **2 hops (2 disk reads)** lage, Alice ka row touch hua hi nahi.
>
> **Without index comparison:**
> ```
> Without index: har row check karna padega → 10 million rows → potentially 10M disk reads
>   → seconds ya usse zyada
> With index (B-Tree): sirf 3-4 disk reads → milliseconds
> ```
> Yehi `O(N)` (slow) vs `O(log N)` (fast) ka real meaning hai — Big-O directly disk-reads-count se correlate karta hai.

B-Tree supports:
- Equality: `WHERE age = 25`
- Range: `WHERE age BETWEEN 20 AND 40`
- Prefix: `WHERE name LIKE 'Bob%'`
- Sorting: `ORDER BY age`

---

## Hash Index

Maps key → exact row location using a hash function. O(1) lookups.

```
Hash Index on 'email' column:

hash('bob@mail.com')   → bucket 7  → row_id 42
hash('alice@mail.com') → bucket 3  → row_id 18
hash('eve@mail.com')   → bucket 7  → row_id 99 (collision, chaining)

Query: WHERE email = 'bob@mail.com'
  hash('bob@mail.com') = bucket 7 → check entries → found row 42
  O(1) lookup!
```

Hash index supports: **Equality only** (`WHERE email = 'x'`)
Hash index does NOT support: Range queries (`WHERE age > 25`), Sorting, Prefix matching

> ### 🔍 Explained: Hash index range queries kyu support nahi karta
> Hash function ki property yahi hai ki **similar inputs → completely unrelated/random outputs** dete hai ("avalanche effect"):
> ```
> hash(24) = 9123847   hash(25) = 302981   hash(26) = 8123049   (sab random, unrelated)
> ```
> Isliye `age > 25` (26, 27, 28...) ke liye hash values me koi order/pattern nahi milta — data completely scattered hai buckets me.
>
> **B-Tree range support karta hai kyunki wo data sorted order me rakhta hai** (hash nahi karta) — `25` dhundo, phir sequentially aage badho, sab contiguous milega:
> ```
> B-Tree (sorted):        [10, 15, 20, 25, 30, 35, 40]  → range easy, consecutive hai
> Hash Index (scattered): bucket7, bucket2, bucket9, bucket1  → koi order nahi, range impossible
> ```

---

## Composite Index (Multi-Column)

Index on multiple columns. Column order matters enormously.

```sql
CREATE INDEX idx_user_country_age ON users (country, age);
```

```
Composite Index (country, age):
  Sorted by country FIRST, then age within each country:

  ('India', 20) → row 5
  ('India', 25) → row 12
  ('India', 30) → row 8
  ('USA', 18)   → row 3
  ('USA', 22)   → row 15
  ('USA', 35)   → row 7
```

### The Left-Prefix Rule

A composite index on `(A, B, C)` can be used for:

| Query | Uses Index? | Why |
|---|---|---|
| `WHERE A = x` | Yes | Left-most column |
| `WHERE A = x AND B = y` | Yes | Left prefix |
| `WHERE A = x AND B = y AND C = z` | Yes | Full index |
| `WHERE B = y` | No | Skipped A (left-most) |
| `WHERE C = z` | No | Skipped A and B |
| `WHERE A = x AND C = z` | Partial | Uses A only, scans for C |
| `WHERE B = y AND C = z` | No | Left-most not included |

> ### 🔍 Explained: "Left-Prefix Rule" — ek analogy
> Socho phone book alphabetically sorted hai **last-name, phir first-name** se (jaise composite index `(country, age)`). Agar tumhe sirf "first name = Bob" wale log dhundhne hai, phone book ka order tumhe koi help nahi karega — tumhe har last-name group ke andar scan karna padega (`WHERE B = y` case). Lekin agar tumhe "last name = Sharma" chahiye, seedha us section pe jump kar sakte ho (`WHERE A = x` case). Index sirf **left se right, in order** hi useful hota hai — beech se skip nahi kar sakte.

Rule of thumb: Put the most selective (most unique) column first, equality conditions before range conditions.

```sql
-- GOOD: equality first, range last
CREATE INDEX idx ON orders (user_id, status, created_at);
-- Supports: WHERE user_id = 123 AND status = 'active' AND created_at > '2024-01-01'

-- BAD: range in the middle breaks subsequent columns
CREATE INDEX idx ON orders (user_id, created_at, status);
-- WHERE user_id = 123 AND created_at > '2024-01-01' AND status = 'active'
-- Only uses (user_id, created_at), can't use status efficiently
```

---

## Covering Index

An index that contains ALL columns needed by a query. The database can answer the query from the index alone without touching the table.

```sql
-- Query:
SELECT name, email FROM users WHERE age > 25;

-- Regular index on (age):
-- Step 1: Find matching rows in index → get row IDs
-- Step 2: Go to table, fetch 'name' and 'email' for each row (random I/O!)

-- Covering index on (age, name, email):
CREATE INDEX idx_covering ON users (age, name, email);
-- Step 1: Find matching rows in index
-- Step 2: name and email are already IN the index → return directly!
-- No table access needed → much faster (index-only scan)
```

Trade-off: Covering indexes are larger (store more data) and slower to update. Use for critical, frequently-run queries.

> ### 🔍 Explained: Covering index actually kya bachaata hai
> Normal index ke saath query 2 "hops" leti hai:
> ```
> Hop 1: Index pe jao → "age > 25" wale row-IDs nikaalo (index disk read)
> Hop 2: Har row-ID ke liye ALAG table location pe jao, name/email fetch karo
>        (1000 matching rows = 1000 EXTRA disk reads!)
> ```
> Covering index me name/email **index ke andar hi** stored hai — Hop 2 ki zarurat hi nahi. Ek poora "extra hop to table" (jo random, expensive disk locations pe hota hai) completely skip ho jata hai.

---

## Comparison Table

| Index Type | Lookup | Range | Sorting | Size | Best For |
|---|---|---|---|---|---|
| B-Tree | O(log N) | Yes | Yes | Medium | General purpose (default) |
| Hash | O(1) | No | No | Small | Exact match lookups |
| Composite | O(log N) | Yes (leftmost) | Yes | Larger | Multi-column queries |
| Covering | O(log N) | Yes | Yes | Large | Avoiding table lookups |
| GIN (inverted) | O(log N) | Depends | No | Large | Full-text search, arrays, JSON |
| GiST | O(log N) | Yes | Depends | Medium | Geospatial, range types |
| BRIN | O(1) | Yes | Yes | Tiny | Naturally ordered data (timestamps) |

---

## The Cost of Indexes (When NOT to Index)

Indexes are not free. Every index has costs:

```
1. WRITE OVERHEAD    — Every INSERT updates table + all indexes (5 indexes = 6 writes per INSERT)
2. STORAGE            — Indexes consume disk space; many indexes can be larger than the table
3. MEMORY              — Indexes should fit in RAM; more indexes = more RAM needed
4. MAINTENANCE        — Indexes can fragment - need REINDEX; schema migrations take longer
```

| Scenario | Why |
|---|---|
| Small tables (< 1000 rows) | Full scan is fast enough, index overhead not worth it |
| Write-heavy tables | Each write updates all indexes, slowing writes |
| Low-cardinality columns | Boolean, status (3 values) — index doesn't help much |
| Columns rarely queried | Index sits unused but costs storage/write perf |
| Frequently updated columns | Every update = index modification |
| Wide columns (TEXT, BLOB) | Index on large values is expensive |

> ### 🔍 Explained: "Low-cardinality" — ek chhota reminder
> Cardinality = kitne unique values ek column me ho sakte hai. `status` column agar sirf `"active"/"inactive"/"banned"` (3 values) ho — index banao bhi toh database ko 3 hi "groups" milenge, jinme se har ek me lakhs rows honge. Ye scanning se zyada fast nahi hoga — bas extra storage + write overhead hi milega. **High cardinality columns (userId, email) index ke liye best hote hai.**

---

## Index Selection Strategy for Interviews

```
Step 1: Identify hot queries (most frequent, slowest)
Step 2: For each query, identify WHERE and ORDER BY columns
Step 3: Create composite index following:
        - Equality columns first (left to right)
        - Range/sort column last
Step 4: Check if covering index is worth it (critical queries)
Step 5: Avoid over-indexing (max 5-7 indexes per table)
```

```sql
-- Example: E-commerce orders table
-- Hot query: "Get recent orders for a user filtered by status"
-- SELECT * FROM orders WHERE user_id = ? AND status = ? ORDER BY created_at DESC

-- Best index:
CREATE INDEX idx_orders_user_status_created
ON orders (user_id, status, created_at DESC);

-- user_id: equality (most selective)
-- status: equality
-- created_at: range/sort (last)
```

---

## Real-World Examples

| Company | Indexing Strategy |
|---|---|
| Uber | Geospatial indexes (PostGIS/H3) for "drivers near me" queries |
| Twitter | Composite indexes on (user_id, created_at) for timeline queries |
| Shopify | Covering indexes on hot product catalog queries to avoid table lookups |
| Stripe | B-tree indexes on payment_intent_id + idempotency_key for fast lookups |
| Netflix | BRIN indexes on timestamp columns for time-range content queries |

---

## Common Interview Questions

**Q: "How would you optimize a slow query?"**
A: First, run `EXPLAIN ANALYZE` to see if it's doing a full table scan. If yes, identify the WHERE/ORDER BY columns and create a composite index. Check if the query can use a covering index. Verify with `EXPLAIN` that the new index is actually being used.

**Q: "Should you index every column?"**
A: No. Indexes slow down writes and consume storage. Index only columns used in WHERE, JOIN, and ORDER BY clauses of frequent queries. Aim for 3-7 well-designed composite indexes per table.

**Q: "What's the difference between a primary key index and a secondary index?"**
A: Primary key index (clustered): the table data is physically stored in primary key order — only one per table. Secondary index: a separate structure pointing back to the primary key; lookups require an additional hop to fetch the full row.

> ### 🔍 Explained: Clustered vs Secondary — quick visual
> ```
> Clustered (primary key) index: table rows THEMSELVES are stored sorted by id
>   [id:1, Bob] [id:2, Alice] [id:3, Eve]   ← physical disk order = id order
>
> Secondary index (e.g., on email): a SEPARATE structure
>   email→id map:  "alice@..."→2, "bob@..."→1, "eve@..."→3
>   Query by email → find id in secondary index → THEN go to clustered index → get full row
>   (2 hops total — this is why secondary index lookups are slightly slower than primary key ones)
> ```

**Q: "How do indexes work with database sharding?"**
A: Each shard has its own local indexes. A query on the shard key routes to one shard and uses local indexes efficiently. A query on a non-shard column might need to hit ALL shards (scatter-gather — see `03-Sharding-Terms-Explained.md`), which is slow. Consider a global secondary index (like DynamoDB GSI) for cross-shard queries.

---

## Quick Self-Check
1. Disk read RAM access se kitna slower hota hai, aur database design me ye kyu matter karta hai?
2. B-Tree me ek "node" kya hai, aur "key" kya hai — apne words me example ke saath batao.
3. Hash index range queries kyu support nahi kar sakta — root cause kya hai?
4. Left-Prefix rule kya hai — composite index `(A,B,C)` par `WHERE B=y` kyu use nahi hota?
5. Covering index kaunsa extra "hop" bachata hai?
6. Clustered aur secondary index me difference kya hai?