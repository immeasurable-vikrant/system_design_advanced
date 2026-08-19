# 14 — CAP, Consistency Models, Caching & Choosing a Database

> **Is lesson me:** CAP theorem (the famous trade-off), ACID vs BASE, consistency spectrum, caching (kya, kyu, Redis kaise, patterns), Redis-as-primary-store, aur sabse important — **ek master decision framework: kaunsa database kis kaam ke liye.** Ye Part 5 ka capstone hai jo sab scaling concepts ko tie karta hai.

Prerequisites: `12-Replication.md`, `13-Sharding.md`, `07-Query-Complexity-Big-O.md`

---

## 1. CAP Theorem

Jab data distributed ho (multiple nodes — replication/sharding), teen properties chahte ho, but **network partition ke time sirf 2 me se 2 mil sakti hai:**

```
C — Consistency:  har read latest write dekhega (saare nodes agree)
A — Availability: har request ko response milega (chahe latest na ho)
P — Partition tolerance: nodes ke beech network toot jaye toh bhi system chale
```

**Key insight:** Distributed system me **P optional nahi hai** — network kabhi bhi fail ho sakta hai, aur tumhe uske liye ready rehna hi hai. Toh real choice hai: **jab partition ho, C ya A?**

```
              Network partition ho gaya (nodes baat nahi kar paa rahe)
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
   CP: Consistency choose             AP: Availability choose
   → jo node sure nahi ki latest      → har node respond karta hai
     data hai, wo REFUSE karta hai      (chahe thoda purana data ho)
   → some requests fail (unavailable)  → sab requests succeed
   → but no wrong/stale data           → but stale data possible
```

### CP vs AP examples
| Type | Behavior | Databases | Use case |
|---|---|---|---|
| **CP** (Consistency + Partition) | Partition me stale-risk wali requests reject | MongoDB (default), HBase, Zookeeper, etcd | Banking, inventory, config |
| **AP** (Availability + Partition) | Partition me sab respond, eventually consistent | Cassandra, DynamoDB, CouchDB | Social feeds, carts, likes |

**Interview line:** *"CAP is really about the partition case — jab network toot jaye. Ek CP system consistency ke liye availability sacrifice karta hai (kuch requests fail); ek AP system availability ke liye stale reads allow karta hai. Normal operation me (no partition) tum dono achhe se de sakte ho — CAP sirf failure ke time bite karta hai."*

> **Replication connection (lesson 12):** async replication = AP-leaning (fast, stale possible). Synchronous quorum reads (`R+W>N`) = CP-leaning (consistent, but partition me block/fail ho sakta hai).

> **⚠️ Reminder:** CAP ka "C" ≠ ACID ka "C" (lesson 08). CAP-C = "har read latest dekhega". ACID-C = "constraints/invariants valid rehte hai". Alag cheezein.

---

## 2. ACID vs BASE

Do philosophies:

**ACID (traditional RDBMS):** Atomicity, Consistency, Isolation, Durability — strong guarantees (lesson 08).

**BASE (many NoSQL):**
```
BA — Basically Available: system hamesha respond kare (chahe stale)
S  — Soft state:          state time ke saath change ho sakta hai (bina input ke bhi, sync se)
E  — Eventual consistency: agar naye writes ruk jaye, sab nodes eventually agree
```

**"Eventual consistency" ka matlab:** turant nahi, but **kuch time baad** saare replicas same value pe converge kar jaate hai. (Yaad karo replication lag — lesson 12. Wo lag window = "not yet consistent"; lag khatam = "eventually consistent".)

| | ACID | BASE |
|---|---|---|
| Consistency | Strong (immediate) | Eventual |
| Availability | Lower (may block) | High |
| Best for | Money, inventory, correctness-critical | Scale, social, analytics |
| Examples | Postgres, MySQL | Cassandra, DynamoDB |

**Real systems mix karte hai:** orders/payments ACID (Postgres), feed/likes BASE (Cassandra) — same product me.

---

## 3. Consistency spectrum (not just binary)

"Consistency" ek slider hai, on/off switch nahi:

```
STRONG ◄─────────────────────────────────────────► WEAK
  │           │              │             │           │
Linearizable  Sequential   Read-your-    Monotonic   Eventual
(latest,      (global      writes        reads       (converges
 always)      order)       (own writes   (never go    someday)
                            visible)      backward)
```

- **Strong/Linearizable:** har read absolute latest (jaise ek single copy). Costly.
- **Read-your-writes:** tum apna khud ka write dekhoge (dusron ka lag ho sakta hai). Common practical target.
- **Monotonic reads:** ek session me time peeche nahi jaayega.
- **Eventual:** eventually converge, no timing guarantee. Cheapest, most scalable.

Design me tum **strongest-jitni-zaroori** choose karte ho, blindly strongest nahi (kyunki strong = slow + less available).

---

## 4. Caching — kya aur kyu

**Cache** = frequently-accessed data ki copy ek **fast layer** (usually RAM) me, taaki slow source (database/disk) baar-baar na touch karna paDe.

**Problem it solves:** Database reads (disk se) milliseconds lete hai. Agar same data 10,000 baar/sec maanga jaye, database overwhelm ho jaata hai. Cache (RAM) us data ko ~0.1ms me deta hai aur database ka load hata deta hai.

> **Caching se pehle log kya karte the (yeh context zaroori hai):**
> - **Vertical scaling** — bada DB server (limit + mehenga)
> - **Read replicas** (lesson 12) — reads distribute (but still disk-based, lag)
> - **Materialized views** — pre-computed query results table me store
> - **Denormalization** (lesson 02/13) — JOINs avoid karke reads fast
> Caching in sab ke upar ek aur fast layer add karta hai — RAM-speed, database ke saamne.

### Cache hit/miss
```
Request aati hai → cache me data hai?
   HIT  → cache se return (fast, ~0.1ms) ✅
   MISS → database se fetch (slow) → cache me daal do → return
```
**Cache hit ratio** (hits / total) = key metric. 90%+ good — matlab 90% requests database tak pahunchti hi nahi.

### Redis kyu itna fast
- **In-memory** — sab RAM me (disk ~1000x slower — lesson 04)
- **Simple data structures** — O(1)/O(log N) ops (lesson 07)
- **Single-threaded event loop** — no lock overhead, predictable

---

## 5. Cache patterns

### Cache-Aside (Lazy Loading) — most common
```
READ:  cache check → HIT return; MISS → DB se lo → cache me daalo → return
WRITE: DB update karo → cache me us key ko INVALIDATE (delete) karo
       (agli read miss hogi → fresh data cache hoga)
```
**Frontend analogy:** React Query exactly ye karta hai — cached data serve karta hai, `staleTime` baad refetch, mutation pe `invalidateQueries`. Redis cache-aside = "React Query for your backend".

### Other patterns (awareness)
- **Write-through:** write cache + DB dono ek saath (cache hamesha fresh, writes slow)
- **Write-behind:** cache turant, DB async baad me (fast, risky)
- **Read-through:** cache khud DB se load karta hai on miss (library-managed)

### TTL + invalidation (the hard part)
```
SET user:123 {...} EX 3600    # 1 ghante baad auto-expire (TTL)
```
> *"There are only two hard things in CS: cache invalidation and naming things."*

Stale cache se bachne ke tarike:
- **TTL** — har entry ki expiry (simple, but TTL window me stale possible)
- **Explicit invalidation** — write pe cache key delete (accurate, but har write-path pe yaad rakhna)
- **Event-based** — DB change event (CDC) se cache update

---

## 6. Redis as a PRIMARY database (not just cache)

Redis usually cache hai (data DB me bhi safe hai, cache uda toh rebuild ho jayega). But Redis ko **primary store** bhi bana sakte ho — tab durability chahiye (kyunki ab source of truth Redis hai):

**Persistence options (lesson 10 WAL se relate):**
- **RDB (snapshots):** periodically poora dataset disk pe dump. Fast restart, but snapshot ke beech ka data crash pe lose.
- **AOF (Append-Only File):** har write operation ko log karo (WAL jaisa). Better durability, replay se recover, but files bade + restart slow.
- **AOF + RDB dono** — production me common (snapshot for fast restart + AOF for durability).

**Redis-as-primary kab THEEK hai:**
| ✅ Fine | ❌ Risky |
|---|---|
| Sessions (loss = re-login, ok) | Money/payments (loss = disaster) |
| Rate-limiting counters | Orders/transactions (need ACID) |
| Leaderboards (Sorted Sets) | Data > RAM (Redis RAM-bound) |
| Real-time analytics (approximate) | Complex queries/JOINs (Redis can't) |
| Pub/sub, queues | Relational integrity needed |

**Rule:** Redis-as-primary sirf tab jab (a) data RAM me fit ho, (b) occasional loss tolerable ho ya AOF acceptable ho, (c) access simple key/structure-based ho. Warna Postgres/DynamoDB primary + Redis cache.

---

## 7. 🎯 MASTER Decision Framework — kaunsa DB kis kaam ke liye

Ye poori series ka practical nichoD hai. **Pehle sawaal: "How will I query this data?"** — phir choose:

### By data + access pattern
| Need | Choose | Kyu |
|---|---|---|
| Structured data, JOINs, transactions, correctness | **PostgreSQL/MySQL** | ACID, relational, flexible queries |
| Key-value at massive scale, predictable latency | **DynamoDB** | O(1) any scale, auto-scale |
| Sub-ms reads, cache, leaderboards, sessions | **Redis** | In-memory, data structures |
| Flexible/evolving schema, documents | **MongoDB** | Document model, flexible |
| Massive write throughput, feeds, time-series | **Cassandra** | Leaderless, write-optimized, AP |
| Full-text / faceted search | **Elasticsearch** | Inverted index |
| Files/images/videos/backups | **S3 (object storage)** | Cheap, durable, unlimited (lesson 15) |
| Event log, streaming, pub/sub | **Kafka** | Append-only log, durable |
| Graph/relationships ("friends of friends") | **Neo4j** | Native graph traversal |
| Analytics/OLAP on huge data | **Snowflake/BigQuery/Redshift** | Columnar, aggregation-optimized |

### By property priority
```
Correctness/transactions critical?  → ACID (Postgres) — CP-leaning
Availability/scale critical?         → BASE (Cassandra/Dynamo) — AP-leaning
Speed critical (reads)?              → Redis cache in front of anything
Storage huge + writes high?          → Shard (lesson 13) ± right base DB
Reads high + availability?           → Replicate (lesson 12)
```

### Polyglot persistence — real systems mix
Ek realistic e-commerce:
```
Orders/payments   → PostgreSQL (ACID, correctness)
Product catalog   → MongoDB or Postgres (flexible attributes)
Search            → Elasticsearch (full-text)
Cart/sessions     → Redis (fast, ephemeral ok)
Product images    → S3 + CDN (lesson 15)
Activity feed     → Cassandra (write-heavy, AP)
Event pipeline    → Kafka (order events, async processing)
Cache layer       → Redis (hot data in front of Postgres)
```
**No single database is "best" — best is per-access-pattern.** Yahi mature system design hai.

---

## Quick Self-Check
1. CAP theorem — teeno letters, aur "real choice" kyu sirf partition ke time hai?
2. CP vs AP — behavior difference during partition, ek-ek example DB?
3. BASE ka har letter, aur "eventual consistency" ka replication-lag se relation?
4. Consistency spectrum me "read-your-writes" aur "eventual" me difference?
5. Cache hit/miss aur cache-aside pattern (read + write path)?
6. Redis fast kyu hai (3 reasons)? Cache invalidation kyu "hard" hai?
7. Redis-as-primary kab theek, kab risky — AOF/RDB ka role?
8. Ek e-commerce ke liye 4 alag data types + unke liye 4 alag stores batao (polyglot).

---
**Next:** `15-Object-Storage-S3.md` — files/images/videos ke liye (relational world ke bahar).