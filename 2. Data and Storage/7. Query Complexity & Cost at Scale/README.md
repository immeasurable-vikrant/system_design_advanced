# 07 — Query Complexity & Cost at Scale (Big-O for Databases)

> **Is lesson me:** Har database operation ki "cost" (O(1), O(log N), O(N)) — kya scale karta hai, kya nahi. Postgres, DynamoDB, aur Redis teeno ke liye. JOIN algorithms. Aur ek decision framework: kaunsa store kis access pattern ke liye. Ye lesson tumhe "1ms vs 10 seconds at 100M rows" ka farak samjhaayega.

Prerequisites: `06-Indexing-Deep-Dive.md`

---

## 1. Query complexity kya hai (aur kyu care karo)

**Query complexity** = ek operation ka time/resource cost, **data size (N) ke relative**. Ye batata hai tumhara design **scale** karega ya nahi.

**Analogy:** Phone book me naam dhundhna:
- Sorted book (indexed) → binary search → **O(log N)**, million entries me ~20 flips
- Unsorted (no index) → har page → **O(N)**, potentially million flips
- Sirf page 1 chahiye (key lookup) → **O(1)**, instant

> **Big-O quick refresher (frontend se):** N = data size.
> - **O(1)** = constant. Data 1000x badhe, time same. (`array[5]`, `map.get(key)`)
> - **O(log N)** = data 1000x badhe, time sirf ~10x. (binary search) — bahut scalable.
> - **O(N)** = data 1000x badhe, time 1000x. (`.filter()`, full scan) — scale pe slow.
> - **O(N log N)** = sorting. **O(N²)** = nested loop (avoid at scale).

---

## 2. PostgreSQL complexity

### Without index (sequential scan)
| Operation | Complexity | Kyu |
|---|---|---|
| `WHERE email = ?` | O(N) | Har row check (full scan) |
| `WHERE age > 25` | O(N) | Range without index = full scan |
| `ORDER BY created_at LIMIT 10` | O(N log N) | Poori table sort, phir 10 |
| `COUNT(*)` | O(N) | Saari rows scan (MVCC — lesson 09) |
| `INSERT` | O(1) | Heap me append |
| `UPDATE ... WHERE id = ?` | O(N) | Row dhundho (scan) + update |

### With B-Tree index
| Operation | Complexity | Kyu |
|---|---|---|
| `WHERE id = ?` (PK) | O(log N) | B-tree traversal |
| `WHERE email = ?` (unique index) | O(log N) | Index seek + row fetch |
| `WHERE age > 25` (indexed) | O(log N + K) | Seek + K matching rows scan |
| `ORDER BY created_at LIMIT 10` (indexed) | O(log N) | Index already sorted, first 10 |
| `INSERT` (with indexes) | O(log N) per index | Row insert + har B-tree update |

> **`O(log N + K)` ka matlab:** `log N` = matching range ka start dhundhne me, `K` = kitne rows actually match hue unhe padhne me. Agar 1000 rows match kare, K=1000.

**Note:** `COUNT(*)` bina WHERE ke Postgres me still O(N) — MVCC ki wajah se exact count ke liye scan karna padta hai (lesson 09 me MVCC).

---

## 3. DynamoDB complexity (key-value at scale)

DynamoDB ek managed NoSQL key-value/document store hai. Iska core: **har cheez partition key (PK) ke through hoti hai.**

| Operation | Complexity | Notes |
|---|---|---|
| `GetItem(PK)` | O(1) | Direct hash lookup |
| `GetItem(PK, SK)` | O(1) | Hash + exact sort key |
| `Query(PK, SK begins_with/between)` | O(1) + O(K) | Hash to partition + sorted scan of K items |
| `Scan(full table)` | O(N) | **Har item padhta hai — production me AVOID** |
| `PutItem` / `UpdateItem` / `DeleteItem` | O(1) | Direct by primary key |

**Key rules:**
- Har query me **partition key MUST** — iske bina koi tarika nahi.
- **Sort key (SK)** ek partition ke andar range queries deta hai.
- **Scan hamesha O(N)** — keys is tarah design karo ki scan kabhi na karna paDe.

**GSI (Global Secondary Index):** ek alag attribute pe query karne deta hai. But har GSI ek async copy hai (eventual consistency) aur write cost badhata hai.

> **Postgres index vs DynamoDB GSI:** Postgres me tum ek table pe bahut indexes/queries laga sakte ho flexibly. DynamoDB me access patterns **pehle se** design karne padte hai keys/GSI ke through — ad-hoc queries nahi kar sakte. Trade-off: DynamoDB predictable O(1) at ANY scale deta hai, but flexibility kam.

---

## 4. Redis complexity (in-memory, data-structure-based)

Redis pure in-memory hai — sab RAM me, isliye base operations super fast. Alag data structures, alag complexities:

| Command | Structure | Complexity | Use case |
|---|---|---|---|
| `GET` / `SET` | String | O(1) | Cache, sessions |
| `HGET` / `HSET` | Hash | O(1) | User profile fields |
| `LPUSH` / `LRANGE 0 9` | List | O(1) / O(K) | Queue, recent-N |
| `SADD` / `SISMEMBER` | Set | O(1) | Membership, tags |
| `ZADD` / `ZRANGE 0 9` | Sorted Set | O(log N) / O(log N + K) | **Leaderboards** |
| `ZRANK` | Sorted Set | O(log N) | Rank of a member |
| `PFADD` / `PFCOUNT` | HyperLogLog | O(1) | Approx unique count |
| `GEOADD` / `GEORADIUS` | Geo | O(log N) / O(N+log N) | Nearby locations |

> **Sorted Set ka magic (interview favorite):** Redis Sorted Set do structures internally use karta hai — ek **skip list** (sorted range ops, O(log N)) + ek **hash table** (score lookup by member, O(1)). Isliye `ZSCORE` O(1) hai but `ZRANGE` O(log N + K). Cost: 2x memory per entry. Yahi wajah leaderboards ke liye perfect hai.

---

## 5. JOIN algorithms (Postgres) — interview depth

Jab tum `JOIN` karte ho, planner 3 algorithms me se chunta hai:

| Algorithm | Complexity | Kab |
|---|---|---|
| **Nested Loop** | O(N × M) worst; O(N × log M) with index | Chhoti tables, ya inner table pe index |
| **Hash Join** | O(N + M) | Chhoti table pe hash table banao, badi se probe. Bade unsorted joins ke liye best |
| **Merge Join** | O(N + M) if both sorted (else + sort cost) | Dono inputs already sorted ho (index scan se) |

Planner table statistics dekh ke best chunta hai. `EXPLAIN ANALYZE` se verify karo kaunsa use hua.

**Frontend analogy:** Nested loop = `arr1.forEach(a => arr2.find(...))` — O(N×M), slow for big. Hash join = ek array ka `Map` bana lo `{key → item}`, phir doosre se O(1) lookup — O(N+M), fast. Yahi optimization database khud karta hai.

---

## 6. Cost at scale — the reference table (memorize the shape)

| Operation | 1M rows | 100M rows | 1B rows |
|---|---|---|---|
| Postgres: indexed lookup | ~1ms | ~2ms | ~3ms |
| Postgres: full scan | ~500ms | ~50s | ~500s (unusable) |
| DynamoDB: GetItem | ~5ms | ~5ms | ~5ms (constant!) |
| DynamoDB: Scan | ~10s | ~16min | ~2.7hr (never!) |
| Redis: GET | ~0.1ms | ~0.1ms | ~0.1ms (in-memory) |
| Redis: ZRANGE top 10 | ~0.1ms | ~0.2ms | ~0.3ms |

**Takeaway:** indexed/key-based ops barely grow with N. Full scans grow linearly → unusable at scale. **Isi table ka "shape" interview me samjhaana = you understand scaling.**

---

## 7. Decision framework — kaunsa store, kis pattern ke liye

| Access Pattern | Best Store | Kyu |
|---|---|---|
| Get user by ID | DynamoDB / Redis | O(1) hash lookup |
| Get user by email | Postgres (unique index) / DynamoDB GSI | O(log N) / O(1) |
| Top 100 players by score | Redis Sorted Set | O(log N + 100), designed for this |
| Orders by user, sorted by date | DynamoDB (PK=user, SK=timestamp) | O(1) + sequential |
| Full-text search on product name | Elasticsearch | Inverted index |
| Count unique visitors | Redis HyperLogLog | O(1) approx, tiny memory |
| Nearby restaurants (5km) | Redis Geo / PostGIS | Geospatial index |
| Complex JOIN (orders+users+products) | PostgreSQL | Only relational handles multi-table JOINs |
| Time-series metrics (last 1hr) | Redis / TimescaleDB | Range query on time |
| Append-only event log | Kafka / DynamoDB (SK=time) | Sequential writes, range reads |

**Golden rule:** *"How will you query this?"* pehle poocho, phir store chuno. Access pattern → store, na ki store → phir pattern.

### When to use / NOT use (quick)
- **Postgres** ✅ complex queries, JOINs, ACID, flexible querying, < ~10TB. ❌ single-key lookups at >100K QPS, unbounded horizontal scale.
- **DynamoDB** ✅ key-value/key-range, predictable perf at any scale. ❌ ad-hoc queries, JOINs, changing access patterns.
- **Redis** ✅ sub-ms reads, leaderboards, sessions, cache, counters. ❌ data > RAM, complex queries, durability-critical primary store.

*(Caching aur Redis-as-primary-store ka poora detail lesson 14 me — jab consistency samajh loge.)*

---

## Quick Self-Check
1. O(1), O(log N), O(N), O(N log N) — har ek ka scale pe kya matlab (data 1000x badhe toh)?
2. Postgres me `WHERE email=?` bina index O(N) kyu, index ke saath O(log N) kyu?
3. `O(log N + K)` ka K kya represent karta hai?
4. DynamoDB me har query me kya MUST hota hai? Scan kyu avoid karte hai?
5. Redis Sorted Set O(log N) range aur O(1) score lookup dono kaise deta hai?
6. Hash Join O(N+M) kaise achieve karta hai — frontend analogy?
7. "Top 100 by score" ke liye Redis vs Postgres — kaunsa aur kyu?

---
**Next:** `08-Transactions-ACID.md` — ab correctness — jab paisa/inventory involved ho.