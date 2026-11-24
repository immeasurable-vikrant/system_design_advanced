# 07 — Query Complexity & Cost at Scale (Big-O for Databases)
 
> **Is lesson me:** Har database operation ki "cost" (O(1), O(log N), O(N)) — kya scale karta hai, kya nahi. Postgres, DynamoDB, aur Redis teeno ke liye (aur andar se kyu). JOIN algorithms ek chhote example se. Common traps (N+1, OFFSET pagination, function on column). Aur ek decision framework: kaunsa store kis access pattern ke liye. Ye lesson tumhe "1ms vs 10 seconds at 100M rows" ka farak samjhaayega.
 
Prerequisites: `06-Indexing-Deep-Dive.md`
 
---
 
## 1. Query complexity kya hai (aur kyu care karo)
 
**Query complexity** = ek operation ka time/resource cost, **data size (N) ke relative**. Ye batata hai tumhara design **scale** karega ya nahi. Exact milliseconds nahi batata — batata hai ki **data badhne pe time kaise badhega**.
 
**Analogy:** Phone book me naam dhundhna:
- Sorted book (indexed) → binary search → **O(log N)**, million entries me ~20 flips
- Unsorted (no index) → har page → **O(N)**, potentially million flips
- Sirf page 1 chahiye (key lookup) → **O(1)**, instant
> **Big-O quick refresher (frontend se):** N = data size.
> - **O(1)** = constant. Data 1000x badhe, time same. (`array[5]`, `map.get(key)`)
> - **O(log N)** = data 1000x badhe, time sirf ~10x. (binary search, B-Tree) — bahut scalable.
> - **O(N)** = data 1000x badhe, time 1000x. (`.filter()`, full scan) — scale pe slow.
> - **O(N log N)** = sorting (`.sort()`). **O(N²)** = nested loop (avoid at scale).
> - **K** = kitni rows **result me** aayi (N = table me kitni hai). `LIMIT 10` → K = 10.
 
> **DB me "time" ≈ kitne pages padhe** (lesson 06). O(log N) = tree ke 3-4 pages; O(N) = table ke saare pages. Isliye Big-O yahan seedha disk reads ka hisaab hai.
 
---
 
## 2. PostgreSQL complexity
 
### Without index (sequential scan)
| Operation | Complexity | Kyu |
|---|---|---|
| `WHERE email = ?` | O(N) | Har row check (full scan) |
| `WHERE age > 25` | O(N) | Range without index = full scan |
| `ORDER BY created_at LIMIT 10` | O(N log N)* | Poori table padho + sort, phir 10 |
| `COUNT(*)` | O(N) | Saari rows scan (MVCC — neeche) |
| `INSERT` | O(1) | Heap (table) ke end me append |
| `UPDATE ... WHERE x = ?` (x pe index nahi) | O(N) | Row dhundho (scan) + update |
 
*\*Postgres `LIMIT` ke saath "top-N heapsort" use karta hai (sirf top 10 yaad rakhta hai) — phir bhi poori table padhni padti hai, toh O(N) se kam nahi hota.*
 
### With B-Tree index
| Operation | Complexity | Kyu |
|---|---|---|
| `WHERE id = ?` (PK) | O(log N) | B-tree traversal (primary key pe index hamesha hota hai) |
| `WHERE email = ?` (unique index) | O(log N) | Index seek + row fetch |
| `WHERE age > 25` (indexed) | O(log N + K) | Seek + K matching rows scan |
| `ORDER BY created_at LIMIT 10` (indexed) | O(log N + 10) | Index already sorted, pehli 10 entries padh ke ruko |
| `INSERT` (with indexes) | O(log N) per index | Row insert + har B-tree update (lesson 06 ka write overhead) |
 
> **`O(log N + K)` ka matlab:** `log N` = matching range ka **start** dhundhne me (tree me neeche utarna), `K` = kitne rows actually match hue unhe padhne me (leaves pe side me chalna). Agar 1000 rows match kare, K=1000. Isliye index tab hi fayda deta hai jab K chhota ho (lesson 06 ka cardinality point).
 
**`COUNT(*)` O(N) kyu (MVCC ek line me):** Postgres me ek hi waqt pe alag transactions ko table ke **alag versions** dikh sakte hai (kisi ne abhi row daali, commit nahi kiya). Toh "kitni rows hai" ka ek fixed number kahin stored nahi — har query ko rows gin ke dekhna padta hai ki **uske liye** kaunsi visible hai. (Detail lesson 09.) **Practical fix:** approx count chahiye toh `pg_class.reltuples` (estimate), exact baar-baar chahiye toh alag counter table/Redis counter.
 
---
 
## 3. DynamoDB complexity (key-value at scale)
 
DynamoDB ek managed NoSQL key-value/document store hai (AWS). Iska core: **har cheez partition key (PK) ke through hoti hai.**
 
### Andar se kaise kaam karta hai (2 lines)
1. **Partition key** ko **hash** karke decide hota hai ki item kis **partition** (ek storage machine ka hissa) pe jaayega — lesson 06 ke hash index jaisa, but machines ke level pe.
2. Ek partition ke andar items **sort key (SK) se sorted** rehte hai — B-Tree jaisa.
```
Table: Orders    PK = user_id      SK = created_at
 
hash("u123") → Partition 7:   (u123, 2026-01-05) (u123, 2026-03-10) (u123, 2026-11-20)  ← SK se sorted
hash("u456") → Partition 2:   (u456, 2026-02-01) (u456, 2026-06-15)
```
- `GetItem(u123, 2026-03-10)` → hash se seedha Partition 7 → wahan exact item. **Data 10x badhe → bas partitions badhte hai**, ek request phir bhi **ek** partition pe jaati hai. Isliye "O(1) at any scale".
- `Query(u123, SK between Jan..Jun)` → Partition 7 pe jao, sorted items me range padho → O(1) + O(K).
- `Scan` → **saare partitions ka saara data** padho → O(N).
| Operation | Complexity | Notes |
|---|---|---|
| `GetItem(PK)` | O(1) | Direct hash lookup |
| `GetItem(PK, SK)` | O(1) | Hash + exact sort key |
| `Query(PK, SK begins_with/between)` | O(1) + O(K) | Hash to partition + sorted scan of K items |
| `Scan(full table)` | O(N) | **Har item padhta hai — production me AVOID** |
| `PutItem` / `UpdateItem` / `DeleteItem` | O(1) | Direct by primary key |
 
**Key rules:**
- Har query me **partition key MUST** — bina PK ke DynamoDB ko pata hi nahi kis partition pe jaaye → sirf Scan bachta hai.
- **Sort key (SK)** ek partition ke andar range queries deta hai ("is user ke last 30 din ke orders").
- **Scan hamesha O(N)** — keys is tarah design karo ki scan kabhi na karna pade.
**GSI (Global Secondary Index):** ek alag attribute pe query karne deta hai. Example: orders ko `status` se dhundhna hai → GSI banao jiska PK = `status`, SK = `created_at`. DynamoDB andar se **data ki ek aur copy** usi naye key se rakhta hai. Isliye: (1) GSI **async** update hota hai → thodi der purana (eventual consistency), (2) har write ab **2 jagah** → write cost double.
 
> **Postgres index vs DynamoDB GSI:** Postgres me tum ek table pe bahut indexes/queries laga sakte ho flexibly, aur koi bhi naya WHERE kabhi bhi likh sakte ho. DynamoDB me access patterns **pehle se** design karne padte hai keys/GSI ke through — ad-hoc queries nahi kar sakte. Trade-off: DynamoDB predictable O(1) at ANY scale deta hai, but flexibility kam.
 
---
 
## 4. Redis complexity (in-memory, data-structure-based)
 
Redis pure in-memory hai — sab RAM me, disk ka wait nahi, isliye base operations super fast. Redis ek "key → **data structure**" store hai; har structure ki apni complexity:
 
| Command | Structure (kya hai) | Complexity | Use case |
|---|---|---|---|
| `GET` / `SET` | **String** — ek value | O(1) | Cache, sessions |
| `HGET` / `HSET` | **Hash** — key ke andar field→value map (JS object jaisa) | O(1) | User profile fields |
| `LPUSH` / `LRANGE 0 9` | **List** — dono taraf se push/pop wali list | O(1) / O(K) | Queue, recent-N |
| `SADD` / `SISMEMBER` | **Set** — unique values, no order | O(1) | Membership, tags |
| `ZADD` / `ZRANGE 0 9` | **Sorted Set** — member + score, score se sorted | O(log N) / O(log N + K) | **Leaderboards** |
| `ZRANK` | Sorted Set | O(log N) | Rank of a member |
| `PFADD` / `PFCOUNT` | **HyperLogLog** — sirf "kitne unique" ka approx count (~12KB me, ~0.8% error) | O(1) | Approx unique visitors |
| `GEOADD` / `GEOSEARCH` | **Geo** — lat/long points (andar Sorted Set) | O(log N) / O(N + log N) | Nearby locations |
 
*(`GEORADIUS` purana command hai; naye Redis me `GEOSEARCH` use karo.)*
 
> **Sorted Set ka magic (interview favorite):** Redis Sorted Set do structures internally use karta hai — ek **skip list** (sorted range ops, O(log N)) + ek **hash table** (score lookup by member, O(1)). Isliye `ZSCORE` O(1) hai but `ZRANGE` O(log N + K). Cost: 2x memory per entry. Yahi wajah leaderboards ke liye perfect hai.
>
> **Skip list (1 line):** ek sorted linked list jisme upar "express lanes" bhi hai — upar ki lane me lambi chhalaang, phir neeche utar ke precise. B-Tree jaisa O(log N), but memory me implement karna simple.
 
> ⚠️ **Redis me bhi O(N) commands hai** — `KEYS *`, bina limit ke `LRANGE 0 -1`, bade Set pe `SMEMBERS`. Redis **single-threaded** hai, toh ek O(N) command baaki **saare** clients ko rok deta hai. Production me `KEYS` ki jagah `SCAN`.
 
---
 
## 5. JOIN algorithms (Postgres) — interview depth
 
Example: `users` (1,000 rows) aur `orders` (1,000,000 rows), query: `SELECT ... FROM orders JOIN users ON orders.user_id = users.id`.
 
Planner 3 algorithms me se chunta hai:
 
**1. Nested Loop** — har outer row ke liye inner me match dhundho:
```js
for (const o of orders)                    // 1M baar
  for (const u of users)                   // × 1000 → 1 billion comparisons ❌
    if (o.user_id === u.id) out.push([o, u]);
// Agar users.id pe index hai: inner loop ki jagah index lookup (log M) → O(N × log M) ✅
```
 
**2. Hash Join** — chhoti table ka hash map banao, badi table se probe karo:
```js
const byId = new Map(users.map(u => [u.id, u]));  // build: 1000 (chhoti table)
for (const o of orders) {                          // probe: 1M × O(1)
  const u = byId.get(o.user_id);
  if (u) out.push([o, u]);
}  // O(N + M) — bade, unsorted joins ke liye best
```
 
**3. Merge Join** — dono sorted ho toh do pointers saath-saath chalao (merge sort ke merge step jaisa):
```js
// orders aur users dono user_id se sorted (index se aaye ya sort kiye)
let i = 0, j = 0;
while (i < orders.length && j < users.length) {
  if (orders[i].user_id < users[j].id) i++;
  else if (orders[i].user_id > users[j].id) j++;
  else { out.push([orders[i], users[j]]); i++; }   // match
}  // O(N + M) agar pehle se sorted; warna + sort cost
```
 
| Algorithm | Complexity | Kab |
|---|---|---|
| **Nested Loop** | O(N × M) worst; O(N × log M) with index | Chhoti tables, ya inner table pe index (OLTP me bahut common) |
| **Hash Join** | O(N + M) | Chhoti table hash map me fit ho; bade unsorted joins |
| **Merge Join** | O(N + M) if both sorted (else + sort cost) | Dono inputs already sorted ho (index scan se) |
 
Planner table statistics (rows count, cardinality — lesson 06) dekh ke best chunta hai. `EXPLAIN ANALYZE` se verify karo kaunsa use hua (`Nested Loop`, `Hash Join`, `Merge Join` likha aata hai).
 
**Frontend analogy:** Nested loop = `arr1.forEach(a => arr2.find(...))` — O(N×M), slow for big. Hash join = ek array ka `Map` bana lo `{key → item}`, phir doosre se O(1) lookup — O(N+M), fast. Yahi optimization database khud karta hai.
 
---
 
## 6. Common traps — index hai, phir bhi O(N)
 
| Trap | Kyu slow | Fix |
|---|---|---|
| **N+1 queries** (ORM) | `users = db.query(User).all()` phir loop me `user.orders` → 1 + N alag queries, har ek ka network round-trip | SQLAlchemy `selectinload(User.orders)` / `joinedload` — 1-2 queries |
| **`OFFSET` pagination** | `LIMIT 20 OFFSET 100000` → DB 100,020 rows padh ke 100,000 phenk deta hai → page 5000 pe O(N) | **Keyset/cursor pagination:** `WHERE created_at < :last_seen ORDER BY created_at DESC LIMIT 20` → O(log N + 20) |
| **Column pe function** | `WHERE lower(email) = 'x'` — index `email` pe hai, `lower(email)` pe nahi | Expression index `CREATE INDEX ON users (lower(email))` |
| **Leading wildcard** | `LIKE '%bob%'` — B-Tree se jump nahi (lesson 06) | GIN + `pg_trgm`, ya search engine |
| **Type mismatch** | `WHERE phone = 9876543210` jabki column text hai → cast, index nahi lagta | Same type pass karo |
| **`SELECT *` bade rows pe** | Covering index bekaar, zyada bytes network pe | Sirf zaroori columns |
 
> N+1 sabse common real-world bug hai — `EXPLAIN` me har query fast dikhti hai, but page pe 200 queries × 2ms = 400ms. ORM logs (`echo=True`) on karke query count dekho.
 
---
 
## 7. Cost at scale — the reference table (memorize the shape)
 
*Numbers rough/illustrative hai — hardware, cache aur row size pe depend karte hai. **Shape** yaad rakho, exact number nahi.*
 
| Operation | 1M rows | 100M rows | 1B rows |
|---|---|---|---|
| Postgres: indexed lookup | ~1ms | ~2ms | ~3ms |
| Postgres: full scan | ~500ms | ~50s | ~500s (unusable) |
| DynamoDB: GetItem | ~5ms | ~5ms | ~5ms (constant!) |
| DynamoDB: Scan | ~10s | ~16min | ~2.7hr (never!) |
| Redis: GET | ~0.1ms | ~0.1ms | ~0.1ms (in-memory) |
| Redis: ZRANGE top 10 | ~0.1ms | ~0.2ms | ~0.3ms |
 
**Takeaway:** indexed/key-based ops barely grow with N (log N me sirf ek extra tree level). Full scans grow linearly → unusable at scale. **Isi table ka "shape" interview me samjhaana = you understand scaling.**
 
> **Network bhi cost hai (lesson 01):** Redis server pe 0.1ms, but app ko network RTT milake ~0.5-1ms milta hai. DynamoDB ka ~5ms zyada tar HTTPS network hai, query nahi. Isliye ek loop me 100 Redis/DB calls = 100 RTTs — **batch karo** (Redis pipeline/`MGET`, DynamoDB `BatchGetItem`, SQL `WHERE id IN (...)`).
 
---
 
## 8. Decision framework — kaunsa store, kis pattern ke liye
 
| Access Pattern | Best Store | Kyu |
|---|---|---|
| Get user by ID | DynamoDB / Redis | O(1) hash lookup |
| Get user by email | Postgres (unique index) / DynamoDB GSI | O(log N) / O(1) |
| Top 100 players by score | Redis Sorted Set | O(log N + 100), designed for this |
| Orders by user, sorted by date | DynamoDB (PK=user, SK=timestamp) | O(1) + sequential |
| Full-text search on product name | Elasticsearch | Inverted index (lesson 06 ka GIN jaisa idea, but distributed + ranking) |
| Count unique visitors | Redis HyperLogLog | O(1) approx, tiny memory |
| Nearby restaurants (5km) | Redis Geo / PostGIS | Geospatial index |
| Complex JOIN (orders+users+products) | PostgreSQL | Only relational handles multi-table JOINs well |
| Time-series metrics (last 1hr) | Redis / TimescaleDB | Range query on time |
| Append-only event log | Kafka / DynamoDB (SK=time) | Sequential writes, range reads |
| Semantic search (embeddings) | pgvector (HNSW) / vector DB | Approx nearest neighbour (lesson 06) |
 
> **Kafka (1 line):** ek distributed, append-only log — events end me judte jaate hai, consumers apni jagah se padhte hai. Queue + storage dono (aage messaging lessons me).
 
**Golden rule:** *"How will you query this?"* pehle poocho, phir store chuno. Access pattern → store, na ki store → phir pattern.
 
### When to use / NOT use (quick)
- **Postgres** ✅ complex queries, JOINs, ACID, flexible querying, < ~10TB. ❌ single-key lookups at >100K QPS, unbounded horizontal scale.
- **DynamoDB** ✅ key-value/key-range, predictable perf at any scale. ❌ ad-hoc queries, JOINs, changing access patterns.
- **Redis** ✅ sub-ms reads, leaderboards, sessions, cache, counters. ❌ data > RAM, complex queries, durability-critical primary store.
*(Caching aur Redis-as-primary-store ka poora detail lesson 14 me — jab consistency samajh loge.)*
 
---
 
## Quick Self-Check
1. O(1), O(log N), O(N), O(N log N) — har ek ka scale pe kya matlab (data 1000x badhe toh)? N aur K me farak?
2. Postgres me `WHERE email=?` bina index O(N) kyu, index ke saath O(log N) kyu?
3. `O(log N + K)` ka K kya represent karta hai? `COUNT(*)` O(N) kyu?
4. DynamoDB partition key aur sort key andar se kya karte hai? Har query me PK kyu MUST? GSI ki 2 costs?
5. Redis Sorted Set O(log N) range aur O(1) score lookup dono kaise deta hai? Redis me `KEYS *` kyu dangerous?
6. Nested loop / hash join / merge join — teeno ka chhota pseudo-code? Hash Join O(N+M) kaise?
7. N+1 problem kya hai aur SQLAlchemy me fix? OFFSET pagination page 5000 pe slow kyu?
8. "Top 100 by score" ke liye Redis vs Postgres — kaunsa aur kyu?
---
 
## Flow — ek query ki cost kaise estimate karo
1. **Access pattern likho:** kaunsa column se dhundh rahe ho (key / range / text / geo), aur kitni rows (K) chahiye.
2. **Store aur structure match karo:** key → hash (DynamoDB/Redis, O(1)); sorted/range → B-Tree / SK / Sorted Set (O(log N + K)); text/JSON → inverted index; kuch match nahi → **scan O(N)** = red flag.
3. **Postgres me planner ka choice dekho:** `EXPLAIN ANALYZE` — `Seq Scan` vs `Index Scan`, aur JOIN me `Nested Loop` / `Hash Join` / `Merge Join`.
4. **Hidden O(N) dhundho:** N+1 loops, OFFSET, column pe function, leading `%`, Redis `KEYS`, DynamoDB `Scan`.
5. **Network add karo:** har call ka RTT bhi cost hai — calls kam karo (batch, join, pipeline), phir scale pe bhi query "shape" flat rahegi.
---
**Next:** `08-Transactions-ACID.md` — ab correctness — jab paisa/inventory involved ho.
 