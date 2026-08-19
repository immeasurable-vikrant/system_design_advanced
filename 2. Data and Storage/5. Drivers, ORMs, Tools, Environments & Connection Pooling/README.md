# 06 — Indexing Deep Dive (Making Queries Fast)

> **Is lesson me:** Index kya hai aur kyu queries ko dramatically fast karta hai, B-Tree/Hash/Composite/Covering indexes, disk reads ka connection, cardinality, aur sabse important — **kab index NAHI lagana chahiye**. Ye poore performance ka core hai.

Prerequisites: `04-How-A-Query-Travels.md` (disk vs RAM, query planner samajh liya ho)

---

## 1. Index kya hai — aur kyu

**Index** = ek extra data structure jo queries ko fast banata hai, **full table scan** avoid karke. Bina index, database har row check karta hai; index se seedha matching rows pe jump karta hai.

**Analogy:** Ek textbook ka index (peeche wala). "Binary Search" dhundhna hai? Bina index → har page palto. Index se → "Binary Search → page 142" dekho, seedha wahan jao.

```
Bina index (full table scan):
  Table: 10 million rows
  Query: SELECT * FROM users WHERE email = 'bob@mail.com'
  → saari 10M rows check karo → O(N) → SLOW (seconds)

Index ke saath (email pe):
  → index me binary-search jaisa lookup → O(log N) → FAST (milliseconds)
```

> **"Row" aur "full scan" quick recap:** Row = ek record (`{id:1, name:"Bob", age:25}`). Full table scan = database har ek row ko ek-ek karke check karta hai. 10M rows = potentially 10M disk reads = seconds. Index isliye chahiye.

> **Disk reads yaad hai? (lesson 04):** Disk RAM se ~1000x slow hai. Index ka poora point = **disk reads kam karna**. `O(N)` (slow) vs `O(log N)` (fast) directly disk-reads-count se correlate karta hai.

---

## 2. B-Tree Index — the default (Postgres/MySQL)

Sabse common index type. Ek **balanced tree** structure:
- Har node me multiple **sorted keys**
- Leaf nodes actual table rows ko point karte hai
- Tree balanced rehta hai (saare leaves same depth pe)

### Node aur key kya hai (concrete example)

Ek **node** = ek disk block/page (ek "box") jisme **100-500 sorted key entries** fit hote hai (na ki sirf 1 per node jaise normal binary tree). Isliye tree "wide" hota hai.

Ek **key** = indexed column ki value (jaise `age=25`) + ek **pointer** jo batata hai row disk pe kaha hai. Key khud data nahi, ek signpost hai.

```
Table: users
  {id:1, name:"Bob", age:25}, {id:2, name:"Alice", age:30}, {id:3, name:"Eve", age:25}

Index on age (tree form):
                    [ age=27 ]                 ← root node
                   /            \
          [age=25]              [age=30]        ← leaf nodes (sorted keys)
          pointers:              pointers:
          → row 1 (Bob)          → row 2 (Alice)
          → row 3 (Eve)

Query WHERE age=25:
  root pe 25 < 27 → left child → key=25 mila → pointers se row 1, 3 fetch
  → 2 hops (2 disk reads), Alice ka row touch hi nahi hua
```

### Depth aur "wide tree" ka magic

Kyunki har node 100-500 keys rakhta hai, tree bahut wide + shallow hota hai:
```
1 million rows  → depth ~3
1 billion rows  → depth ~4-5 (barely increases!)
```
Ye **logarithmic (O(log N))** growth hai — data 1000x badhe, depth sirf 1-2 level. **Har level = 1 disk read.** Isliye chahe billions rows ho, lookup sirf **3-5 disk reads = milliseconds**.

**B-Tree supports:**
```
Equality:  WHERE age = 25
Range:     WHERE age BETWEEN 20 AND 40
Prefix:    WHERE name LIKE 'Bob%'
Sorting:   ORDER BY age  (tree already sorted!)
```

---

## 3. Hash Index — O(1) but limited

Key ko hash function se seedha exact row location pe map karta hai. O(1) lookup.

```
Hash Index on email:
  hash('bob@mail.com')   → bucket 7 → row 42
  hash('alice@mail.com') → bucket 3 → row 18

Query WHERE email = 'bob@mail.com':
  hash('bob@mail.com') = bucket 7 → row 42 found → O(1)!
```

> **Hash function recap (detail lesson 11 me):** hash = koi bhi input ko ek fixed-size number me convert karta hai, deterministically. Similar inputs → completely unrelated outputs.

**Hash index supports:** sirf **Equality** (`WHERE email = 'x'`)
**Does NOT support:** Range (`age > 25`), Sorting, Prefix

**Kyu range nahi?** Hash "avalanche effect" — `hash(24), hash(25), hash(26)` bilkul unrelated random numbers. Koi order nahi bacha, isliye "range" nikalna impossible. B-Tree range support karta hai kyunki wo data **sorted** rakhta hai (hash nahi karta):
```
B-Tree (sorted):     [10,15,20,25,30,35,40]  → range easy, consecutive
Hash (scattered):    bucket7, bucket2, bucket9  → koi order nahi, range impossible
```

---

## 4. Composite Index — multiple columns

Ek index jo multiple columns pe ho. **Column order bahut matter karta hai.**

```sql
CREATE INDEX idx_country_age ON users (country, age);
```
```
Sorted by country FIRST, phir age within each country:
  ('India', 20), ('India', 25), ('India', 30),
  ('USA', 18), ('USA', 22), ('USA', 35)
```

### The Left-Prefix Rule (critical)

Index `(A, B, C)` in queries ke liye use hota hai:

| Query | Index use? | Why |
|---|---|---|
| `WHERE A = x` | ✅ | Left-most |
| `WHERE A = x AND B = y` | ✅ | Left prefix |
| `WHERE A = x AND B = y AND C = z` | ✅ | Full |
| `WHERE B = y` | ❌ | A skip kiya (left-most missing) |
| `WHERE C = z` | ❌ | A, B skip kiye |
| `WHERE A = x AND C = z` | ⚠️ Partial | Sirf A use, phir C ke liye scan |

**Analogy:** Phone book last-name phir first-name se sorted hai. "First name = Bob" dhundhna = koi help nahi (poora scan). "Last name = Sharma" = seedha jump. Index sirf **left-to-right, in order** useful hai.

**Rule of thumb:** most selective (unique) column pehle, **equality conditions pehle, range condition last:**
```sql
-- GOOD: equality first, range last
CREATE INDEX idx ON orders (user_id, status, created_at);
-- WHERE user_id=123 AND status='active' AND created_at > '2024-01-01' ✅

-- BAD: range beech me → uske baad wale columns index use nahi kar paate
CREATE INDEX idx ON orders (user_id, created_at, status);
-- created_at range ke baad status index se filter nahi ho paata
```

---

## 5. Covering Index — table touch hi mat karo

Ek index jisme **query ke saare needed columns** ho. Database poora answer index se hi de deta hai, table tak jaaye bina.

```sql
SELECT name, email FROM users WHERE age > 25;

-- Regular index on (age): 2 hops
--   Hop 1: index se age>25 wale row-IDs nikaalo
--   Hop 2: har row-ID ke liye TABLE pe jao, name/email fetch karo
--          (1000 rows match = 1000 EXTRA disk reads!)

-- Covering index on (age, name, email): 1 hop
CREATE INDEX idx_cov ON users (age, name, email);
--   name/email index me hi hai → table jaane ki zarurat nahi (index-only scan)
```

Ye "extra hop to table" (random disk locations, expensive) completely skip ho jaata hai.
**Trade-off:** covering index bada hota hai (zyada data store) aur writes slow. Sirf critical, frequently-run queries ke liye.

---

## 6. Index types quick comparison

| Type | Lookup | Range | Sort | Size | Best for |
|---|---|---|---|---|---|
| B-Tree | O(log N) | ✅ | ✅ | Medium | General (default) |
| Hash | O(1) | ❌ | ❌ | Small | Exact match |
| Composite | O(log N) | ✅ (leftmost) | ✅ | Larger | Multi-column |
| Covering | O(log N) | ✅ | ✅ | Large | Avoid table hops |
| GIN | O(log N) | Depends | ❌ | Large | Full-text, arrays, JSON |
| GiST | O(log N) | ✅ | Depends | Medium | Geospatial, ranges |
| BRIN | O(1) | ✅ | ✅ | Tiny | Naturally-ordered data (timestamps) |

---

## 7. The Cost of Indexes — kab NAHI lagana

Indexes free nahi hai:
```
1. WRITE OVERHEAD — har INSERT table + saare indexes update karta hai
                    (5 indexes = 6 writes per INSERT)
2. STORAGE        — indexes disk space lete hai (kabhi table se bhi bade)
3. MEMORY         — indexes RAM me fit hone chahiye (more indexes = more RAM)
4. MAINTENANCE    — fragment ho sakte hai (REINDEX), migrations slow
```

**Kab index NAHI lagao:**
| Scenario | Kyu |
|---|---|
| Chhoti tables (< 1000 rows) | Full scan already fast, overhead worth nahi |
| Write-heavy tables | Har write saare indexes update karta hai (slow writes) |
| Low-cardinality columns | Boolean, status (3 values) — index zyada help nahi karta |
| Rarely-queried columns | Index unused, but storage/write cost leta hai |
| Frequently-updated columns | Har update = index modification |
| Wide columns (TEXT/BLOB) | Bade values pe index expensive |

> **Cardinality reminder:** kitne unique values. `status` (3 values) = low cardinality — index banao bhi toh 3 hi groups, har ek me lakhs rows, scan se zyada fast nahi. `userId`/`email` (millions unique) = high cardinality = index ke liye best.

---

## 8. Index selection strategy (interview-ready)

```
Step 1: Hot queries identify karo (most frequent, slowest)
Step 2: Har query ke WHERE aur ORDER BY columns dekho
Step 3: Composite index banao:
        - equality columns pehle (left to right)
        - range/sort column last
Step 4: Covering index worth hai? (critical queries ke liye)
Step 5: Over-indexing avoid karo (max ~5-7 per table)
```
```sql
-- Hot query: "user ke recent orders, status se filter"
-- SELECT * FROM orders WHERE user_id=? AND status=? ORDER BY created_at DESC
CREATE INDEX idx ON orders (user_id, status, created_at DESC);
-- user_id: equality (selective), status: equality, created_at: range/sort last
```

**Real companies:** Uber (geospatial indexes for "drivers near me"), Twitter (composite on `(user_id, created_at)` for timelines), Stripe (B-tree on `payment_intent_id` + idempotency key), Netflix (BRIN on timestamps for time-range).

---

## 9. Clustered vs Secondary index (interview favorite)

- **Clustered (primary key) index:** table ki rows **physically** primary-key order me stored hoti hai. Ek hi per table.
- **Secondary index:** ek alag structure jo primary key ko point karta hai. Lookup me ek extra hop lagta hai (secondary → primary → full row).

```
Clustered (by id): rows themselves sorted by id on disk
  [id:1 Bob] [id:2 Alice] [id:3 Eve]

Secondary (on email): separate structure
  "alice@"→id 2, "bob@"→id 1  → phir clustered se full row (2 hops)
```

---

## Quick Self-Check
1. Index disk reads kaise kam karta hai — `O(N)` vs `O(log N)` ka disk-read se relation?
2. B-Tree me "node" aur "key" kya hai — apne words me example ke saath.
3. B-Tree depth billions rows pe bhi kam kyu rehti hai?
4. Hash index range queries kyu nahi kar sakta (root cause)?
5. Left-prefix rule kya hai — `(A,B,C)` pe `WHERE B=y` kyu use nahi hota?
6. Covering index kaunsa "hop" bachata hai?
7. Index kab NAHI lagana chahiye — 3 scenarios?
8. Clustered aur secondary index me difference?

---
**Next:** `07-Query-Complexity-Big-O.md` — har operation ki exact cost, aur DB-by-DB comparison.