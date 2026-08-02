# 08 — Caching & Redis: Why, When, and "What Did We Do Before?"

> Ye file specifically tumhare sawaal ka jawab hai: "how can a cache be used as a DB, Redis cache why/when, and what were we doing before?" Chalo poora concept scratch se banate hai.

Prerequisites: `05-Indexing-Terms-Explained.md` (disk vs RAM speed concept yaha reuse hoga), `07-DB-RDBMS-SQL-Query-Journey.md`

---

## 1. Pehle: "What were we doing before caching?" — the actual problem

Socho Instagram ka ek popular post hai — "Cristiano Ronaldo ki nayi post", jisko **10 million log per second dekh rahe hai** (like counts, comments count refresh ho rahe hai).

Bina cache ke, har single user jo page refresh karega:

```
User 1 request  → App Server → Postgres: "SELECT like_count FROM posts WHERE id = X" → disk read → return
User 2 request  → App Server → Postgres: "SELECT like_count FROM posts WHERE id = X" → disk read → return
User 3 request  → App Server → Postgres: "SELECT like_count FROM posts WHERE id = X" → disk read → return
... 10 million times, EXACT SAME QUERY, EXACT SAME ANSWER, again aur again
```

Problem: (a) Postgres server pe **massive unnecessary load** — same exact query baar baar chal rahi hai jab answer already pata hai, (b) har query disk read involve karti hai jo slow hai (`05-Indexing-Terms-Explained.md` yaad karo — disk access RAM se ~1000x slower hota hai), (c) database eventually **overload ho ke crash/slow ho jayega** kyunki ek hi machine (ya shard) itna traffic handle nahi kar sakta.

**"What did we do before caching existed" (ya jaha cache nahi use kiya jata):**
- **Vertical scaling** — bigger/faster database machine le lo (mehenga, limit hai)
- **Read Replicas** — jaisa `04-Replication-Terms-Explained.md`/doc B2 me discuss hua, multiple copies bana ke read load distribute karo — but replicas bhi utne hi disk-based hote hai, still slower than memory, aur replica lag ka risk bhi hai
- **Materialized views** — periodically ek "pre-computed" summary table banao (jaise "har ghante like_count ka snapshot table"), but ye bhi stale ho sakta hai aur extra storage/compute leta hai
- **Denormalization** — data ko duplicate/precompute karke rakho taaki query simpler ho (jaisa `03-Sharding-Terms-Explained.md` me discuss hua)

Ye sab approaches **kaam karte hai but limited** hai — kyunki fundamentally, disk-based database har baar disk hi touch kar raha hai. **Caching ek fundamentally different approach hai: disk ko touch hi mat karo, agar answer already RAM me hai.**

---

## 2. Cache kya hota hai — core idea

**Cache** = ek chhota, super-fast **in-memory (RAM-based)** storage jo **recently/frequently accessed data ki ek copy** rakhta hai, taaki agla request usi RAM se serve ho sake, disk tak jaane ki zarurat na pade.

```
                       ┌─────────────────┐
   Request 1  ────────▶│   Cache (Redis)  │
                       │   [EMPTY]         │
                       └────────┬────────┘
                                │ cache MISS (data nahi mila)
                                ▼
                       ┌─────────────────┐
                       │  Postgres (disk)  │  → fetch actual data (slow, disk read)
                       └────────┬────────┘
                                │
                                ▼
                       Result ko Cache me BHI store kar do
                       (taaki agli baar seedha yahi se mile)

   Request 2  ────────▶┌─────────────────┐
   (same data)         │   Cache (Redis)  │
                       │  [data present!]  │  → cache HIT! (fast, RAM se seedha return)
                       └─────────────────┘
                                (Postgres ko touch hi nahi kiya is baar!)
```

**Cache Hit** = data cache me mil gaya, disk tak jaane ki zarurat nahi (fast, ~microseconds).
**Cache Miss** = data cache me nahi tha, database se fetch karna pada (slow, but ab cache me daal diya future ke liye).

**Frontend analogy:** Ye bilkul `useMemo`/`React Query`'s cache jaisa concept hai — pehli baar ek expensive computation/API call hoti hai, result "memoize"/cache ho jata hai, agli baar wahi input aane pe seedha cached value return ho jaati hai bina dobara compute/fetch kiye. Redis = "useMemo, but shared across your ENTIRE backend infrastructure, not just one component."

---

## 3. Redis specifically kya hai, aur itna fast kyu hai

**Redis** = ek **in-memory key-value store**. "In-memory" ka matlab: data **RAM me** store hota hai (disk pe nahi, by default) — isliye access **RAM-speed** pe hota hai, disk-speed pe nahi.

```
Postgres:  SELECT like_count FROM posts WHERE id = 'post_101'
           → disk read involved → ~1-10 milliseconds (with index)

Redis:     GET post_101_likes
           → pure RAM lookup → ~0.1 milliseconds (10-100x faster!)
```

Redis ka data model bahut simple hai — **key → value** (bilkul JS `Map`/`Object` jaisa):

```
SET post_101_likes 45230
GET post_101_likes → "45230"

SET user_session_abc123 '{"userId": 5, "loggedIn": true}'
GET user_session_abc123 → '{"userId": 5, "loggedIn": true}'
```

Ye direct hash-table lookup hai — `01-Hashing-Basics-Explained.md` yaad karo, ye woh hi concept hai jo JS `Object`/`Map` internally use karta hai — Redis basically ek **network-accessible, shared, super-fast hash table** hai jise tumhare poore backend infrastructure (multiple servers) access kar sakte hai.

---

## 4. "Cache as a Database" — matlab kya hai, aur kab karte hai

Tumhara exact sawaal: **"how can a cache use as a DB"**

Normally Redis ek **cache** hoti hai — matlab "source of truth" (asli, permanent data) Postgres me hi hota hai, Redis sirf ek **temporary, fast copy** rakhti hai. Agar Redis crash ho jaye aur saara data khatam ho jaye, **koi baat nahi** — Postgres se dobara populate kar sakte ho.

**Lekin Redis ko actual "primary database" (source of truth) ki tarah bhi use kiya ja sakta hai**, kuch specific cases me:

```
Redis Persistence Options (jo isko "just a cache" se "usable as a DB" banate hai):

1. RDB (snapshotting) — periodically (e.g., every 5 min) poora dataset
   disk pe ek snapshot file me save karta hai
   → Fast, but agar crash ho crash se pehle wale 5 min ka data lose ho sakta hai

2. AOF (Append-Only File) — har write operation ko ek log file me
   likhta hai (WAL jaisa concept — `04-Replication-Terms-Explained.md`
   yaad karo), crash ke baad replay karke data rebuild kar sakta hai
   → Zyada durable, but thoda slower writes
```

Inn options ke saath, Redis crash-resistant ban jaata hai — matlab ab isko "sirf disposable cache" nahi, ek "real database jiska data crash survive karta hai" ki tarah treat kar sakte ho.

**Kab Redis-as-primary-store use karna sensible hai (real examples):**

| Use case | Kyu Redis theek hai as primary store |
|---|---|
| **Session storage** | Session data temporary hoti hai anyway (login expire ho jata hai) — agar Redis crash ho bhi jaye, users bas dobara login kar lenge, koi permanent business data lose nahi hua |
| **Rate limiting counters** | "User X ne last 1 min me kitni API calls ki" — ye purely transient hai, kabhi bhi reset ho sakta hai bina harm ke |
| **Real-time leaderboards** | Gaming leaderboard (score rankings) — Redis ke "Sorted Sets" data structure specifically iske liye design hui hai, super fast ranking queries |
| **Real-time counters** | Live view counts, like counts jo "eventually" Postgres me bhi sync ho jaate hai — Redis fast real-time number deta hai, Postgres periodically true source of truth update hoti hai |

**Kab Redis-as-primary-store RISKY hai (avoid karo):**

| Use case | Kyu Redis primary store ke liye galat hai |
|---|---|
| **Bank account balances** | Agar Redis data lose ho jaye (crash between snapshots), **real money ka data lose ho sakta hai** — yaha strict ACID guarantee (Postgres) zaroori hai |
| **Order/payment records** | Legal/compliance requirement hoti hai permanent, durable storage ki — Redis ka "mostly in RAM" nature isse risky banata hai |
| **Complex relational queries** | Redis me JOINs, complex filtering nahi hota jaisa SQL me hota hai — agar tumhe "sabhi orders jinka status='pending' AND amount > 100 AND user.country='India'" chahiye, Redis iske liye design hi nahi hua |

**One-line rule of thumb:** *Agar data lose hone se "annoyance" hoti hai (user dobara login kare) → Redis primary store theek hai. Agar data lose hone se "real damage/legal problem" hoti hai (paisa, orders) → Postgres/RDBMS hi primary source of truth rahegi, Redis sirf cache ki tarah uske aage baithegi.*

---

## 5. Where does the cache physically sit — full picture

```
┌──────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Browser  │─────▶│  App Server   │─────▶│  Redis (Cache) │      │  Postgres     │
│  (Client) │      │ (Node/Python) │      │  in-memory     │      │  (disk, source│
└──────────┘      └──────┬───────┘      └──────┬───────┘      │  of truth)    │
                          │                      │ cache HIT?    └───────▲──────┘
                          │                      │ yes → return          │
                          │                      │ no → fall through ────┘
                          │◀─────────────────────┘
                          │
                    Response to browser
```

Typical flow (called **"cache-aside" pattern**, most common):

```
1. App gets request "get post 101 details"
2. App checks Redis: GET post_101 → agar mila (HIT), return directly, DONE.
3. Agar Redis me nahi mila (MISS):
   a. Query Postgres: SELECT * FROM posts WHERE id = 101
   b. Result ko Redis me store karo: SET post_101 <result> (with expiry, e.g., 5 min)
   c. Result client ko return karo
4. Agli request (same post) → ab Redis me hit milega, Postgres touch hi nahi hoga
```

**Cache Invalidation (common gotcha):** Agar post ka data update ho (jaise like count badha), purana cached value **stale** ho jaata hai. Isliye har cached entry ko **TTL (Time To Live / expiry)** diya jaata hai (jaise "5 minutes ke baad automatically expire ho jao"), ya explicitly update ke time cache ko bhi update/clear kiya jaata hai. Ye itna tricky problem hai ki ek famous CS quote hai: *"There are only two hard things in Computer Science: cache invalidation and naming things."*

---

## Quick Self-Check
1. Caching ke bina, popular content pe traffic scale karne ke liye pehle kya approaches thi?
2. Cache hit aur cache miss me difference kya hai?
3. Redis itna fast kyu hai Postgres ke comparison me (fundamental reason)?
4. Redis ko "primary database" ki tarah kab use karna sensible hai, aur kab risky?
5. "Cache-aside" pattern ka poora flow kya hai (miss hone pe)?
6. Cache invalidation itna "hard problem" kyu mana jata hai?