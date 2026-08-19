# 13 — Sharding (Different Data, Different Machines)

> **Is lesson me:** Partitioning vs sharding, kab shard karna chahiye (aur kab nahi), sharding strategies, shard key kaise choose karo, hot partitions, cross-shard queries (scatter-gather, denormalize, 2PC, saga), rebalancing, aur sharding+replication together. Ye "scale writes + storage" ka solution hai.

Prerequisites: `11-Hashing-And-Consistent-Hashing.md`, `12-Replication.md`

---

## 1. Partitioning vs Sharding (pehle terms clear)

**Partitioning** = data ko **logical chunks** me todna, kisi rule ke basis pe (user_id range, date, hash, etc.):
```
100M users → Partition 1: id 1-25M, Partition 2: 25M-50M, ...
```
Ye abhi sirf **logical split** hai — chunks physically kaha honge, wo sharding decide karta hai.

**Sharding** = partitioning ka wo version jaha har partition **alag physical server** pe rakha jaata hai. Har shard ek **independent, self-contained database** hai — apna storage, compute, I/O.

```
          [ Router / Query Coordinator ]
          (decides which shard to hit)
         /          |            \
   [Shard 1]    [Shard 2]     [Shard 3]
   (Server A)   (Server B)    (Server C)
   id 1-25M      id 25M-50M    id 50M-75M
```

**Ek line me:** partitioning = logical split; sharding = logical split + physically alag machines pe.

---

## 2. Sharding kya solve karta hai (replication se alag)

Replication har node pe **poora data** rakhta hai — storage nahi badhta, writes still ek leader pe. Sharding:
- **Storage scaling** — total data N machines me split, ek machine ki disk limit se free
- **Write throughput scaling** — writes bhi N machines me distribute (replication me saare writes ek leader pe the!)

```
Without sharding: 1 billion rows → 1 server → CPU maxed, disk full, slow
With sharding (4 shards): har server ke paas 250M rows → manageable
```

---

## 3. Kab shard karna chahiye (aur kab NAHI)

**Abhi NAHI, agar:** database < 1TB aur < 10K queries/sec — single server + read replicas kaafi hai. **Prematurely shard mat karo** (complexity bahut badh jaati hai).

**Shard karo jab:**
- Single DB disk capacity exceed kare (> 1-5 TB)
- Write throughput single server se zyada (> 10-50K writes/sec)
- Read replicas kaafi nahi (write-heavy workload)
- Query latency badh rahi despite indexes

**Rule of thumb:** *Pehle reads ko replicas se scale karo. Shard sirf tab jab WRITES bottleneck ho.*

---

## 4. Sharding strategies

### (a) Hash-Based
```
shard_id = hash(userId) % num_shards
hash("user_001") % 4 = 2 → Shard 2
```
**Pros:** even distribution, no hotspots from sequential keys. **Cons:** range queries impossible (scattered), adding shards = rehash (consistent hashing se fix — lesson 11).
**Best for:** key-value lookups, user data, sessions.

> **`hash % N` recap (lesson 11):** shard key ko hash karke number banao, mod se shard-index (0 to N-1) nikaalo. Adding shards pe consistent hashing use karo taaki sirf O(K/N) keys move ho.

### (b) Range-Based
```
Shard 0: userId 0-999,999    Shard 1: 1M-2M    Shard 2: 2M-3M
```
**Pros:** range queries natural ("all users 1M-1.5M"). **Cons:** hotspots (naye users sab last shard pe), uneven sizes.
**Best for:** time-series (shard by month), geographic (by region).

### (c) Directory-Based
```
Lookup table: user_001 → Shard 2, user_002 → Shard 0, ...
Query: pehle directory dekho, phir us shard pe jao.
```
**Pros:** flexible (any key → any shard), easy rebalancing. **Cons:** directory single point of failure, extra hop per query.

### (d) Geographic
```
Shard "US" (us-east-1), Shard "EU" (eu-west-1), Shard "APAC" (ap-south-1)
```
**Pros:** data locality (low latency), compliance (EU data in EU for GDPR). **Cons:** uneven (US 5x bigger), cross-region queries expensive.

---

## 5. Shard key choose karna (most important decision)

Shard key decide karta hai kaunsi row kaunse shard pe jayegi.

**Good shard key properties:**
- **High cardinality** — bahut unique values (userId good, country bad)
- **Even distribution** — koi single value dominate na kare
- **Matches query patterns** — queries me shard key ho → single-shard lookup

| Shard Key | Good for | Bad for |
|---|---|---|
| userId | User-centric (get all of one user's data) | Cross-user queries |
| orderId | Order lookups | "all orders for user X" (scan all shards) |
| timestamp | Time-series | Hot partition (all writes → "now" shard) |
| country | Geo | Uneven (US 60% traffic) |
| hash(userId) | Even distribution | Range queries, debugging |

> **Cardinality recap (lesson 06):** kitne unique values. `country` (~195) = low → agar shard key ho, kuch shards overloaded. `userId` (millions) = high → evenly spread ho sakta hai (especially hashed).

---

## 6. Hot Partition Problem

**Problem:** ek shard disproportionately zyada traffic paata hai.

**Examples:** celebrity user (10M followers → unka shard har post pe hammered), time-based key (saare current writes "today" shard pe), popular product (flash sale → us product ka shard overwhelmed).

**Solutions:**
| Solution | How |
|---|---|
| Salt/suffix to key | `shard_key = userId + "_" + random(0,9)` → 10 sub-shards. Reads fan-out karne padte hai. |
| Dedicated shard for hot keys | Hot key detect → dedicated high-capacity shard |
| Caching in front | Hot data aggressively cache → most reads shard tak pahunche hi na (lesson 14) |
| Further split hot shard | Ek shard ko 4 me toDo |

---

## 7. Cross-Shard Queries (the biggest pain)

Multiple shards ko span karne wali queries expensive hai.

```
"Top 10 orders across ALL users, sorted by amount"
→ ALL shards query karo → har ek apna top 10 de → merge → global top 10
→ SCATTER-GATHER pattern (slow)
```

> **Scatter-gather explained:**
> ```
> SCATTER: Shard1,2,3,4 ko PARALLEL query bhejo ("give me your top 10")
> Har shard apna local top 10 return kare
> GATHER: app layer pe combine + re-sort → global top 10
> ```
> **Frontend analogy:** `Promise.all()` se 4 APIs parallel call karke `.flat().sort()` se merge. **Response time = slowest shard + merge overhead.**

**Handling patterns:**
| Pattern | When | Trade-off |
|---|---|---|
| Denormalize | Duplicate data store, cross-shard JOIN avoid | Write amplification |
| Application-side join | Query both shards, app me merge | Complex, higher latency |
| Scatter-gather | Fan out to all shards, aggregate | Latency = slowest shard |
| Secondary index (Elasticsearch) | Alag search index, differently sharded | Extra infra, lag |

> **Denormalize + write amplification (recap from lesson 02):** `orders` me `userName` copy kar do taaki `users` se JOIN na karna paDe. But Bob → "Robert" rename kare toh **har order row** update karni padegi (one logical update → many physical writes = write amplification).

**Interview best practice:** *"Main userId se shard karunga taaki ek user ka saara data ek shard pe ho (most queries single-shard). Cross-user queries (leaderboards, analytics) ke liye ek alag denormalized read store ya Elasticsearch use karunga (CDC se synced)."*

---

## 8. Cross-shard transactions

ACID transactions cross-shard easily nahi ho sakti (2 independent databases). Options:
1. **Single-shard design** — transactional entity se shard karo taaki transaction ek shard pe rahe (best)
2. **Saga pattern** — transaction ko chhote local steps + compensating (undo) actions me toDo
3. **Two-Phase Commit (2PC)** — coordinator dono shards se confirm le, phir commit (slow, avoid)

> **2PC vs Saga (recap from lesson 09 concepts):**
> - **2PC:** coordinator "ready?" poochta hai, dono lock+yes bolte hai, phir "commit". Problem: coordinator crash → shards **locked** reh jaate hai. Blocking, fragile.
> - **Saga:** har step ka undo define karo. Step 2 fail → step 1 ka undo chalao (compensation). Eventually consistent, no long locks, but app complexity zyada.
> **Frontend analogy:** multi-step checkout — payment fail → reserved inventory release karo (undo), instead of ek giant atomic transaction.

---

## 9. Rebalancing

Jab shards uneven ho jaye (ek bahut bada), data move karna padta hai.

| Strategy | How | Downtime |
|---|---|---|
| Fixed partitions | Pre-create many (e.g., 1000), groups ko nodes me assign. Rebalance = groups move | Minimal |
| Dynamic splitting | Shard threshold exceed kare → do me split | Zero (background) |
| Consistent hashing | Naye server ke virtual nodes add, sirf ~K/N keys move | Zero |

- **DynamoDB:** auto-splits partitions (>10GB ya 3000 RCU/1000 WCU). You don't manage manually.
- **Cassandra:** consistent hashing + vnodes, adding node = auto rebalance.

---

## 10. Sharding + Replication together (real systems)

Real systems **dono** use karte hai: shard for writes/storage, phir har shard ko replicate for reads/HA.

```
Shard 1: Primary → Replica 1A, Replica 1B
Shard 2: Primary → Replica 2A, Replica 2B
Shard 3: Primary → Replica 3A, Replica 3B
```
Matlab ek "shard" khud andar se ek leader + followers ho sakta hai.

**Sharding vs Replication summary:**
| | Sharding | Replication |
|---|---|---|
| Purpose | Scale writes + storage | Scale reads + availability |
| Data | Har shard DIFFERENT data | Har replica SAME data |
| Failure | Shard down = that data unavailable | Replica down = others serve |
| When | Write-heavy, large data | Read-heavy, high availability |

---

## Quick Self-Check
1. Partitioning aur sharding me exact difference?
2. Sharding replication se alag kya solve karta hai (storage + writes)?
3. Kab shard NAHI karna chahiye? Rule of thumb (reads vs writes)?
4. 4 sharding strategies + ek-ek best use case?
5. Good shard key ke 3 properties? userId good, country bad kyu?
6. Hot partition kya hai + 2 solutions?
7. Scatter-gather ka response time kis pe depend karta hai?
8. Cross-shard transaction ke liye 2PC vs saga — difference?

---
**Next:** `14-CAP-Consistency-Models-And-Choosing.md` — CAP theorem, ACID vs BASE, caching, aur final "which DB for what" framework.