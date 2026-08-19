# 06 — Redis vs Memcached (Choosing the Cache Engine)

> **Is lesson me:** Do sabse common distributed caches (Layer 4) — Redis aur Memcached — ka comparison, kab kaunsa, aur Redis itna default kyu ban gaya. DB lesson 07/14 me Redis dekha tha; yahan cache-engine ke perspective se.

Prerequisites: `02-Caching-Fundamentals-And-Layers.md`, helpful: DB lesson 07 (Redis structures)

---

## 1. Dono kya hai

Dono **in-memory key-value stores** hai jo Layer 4 (distributed cache) me use hote hai — data RAM me, sub-millisecond access.

- **Memcached** (2003) — pure, simple, multi-threaded cache. Sirf strings. Kuch nahi karta except fast key-value.
- **Redis** (2009) — "data structure server". Strings + Lists + Sets + Sorted Sets + Hashes + Streams + pub/sub + optional persistence + replication. Bahut zyada features.

---

## 2. Head-to-head comparison

| Feature | Redis | Memcached |
|---|---|---|
| **Data structures** | Strings, Lists, Sets, Sorted Sets, Hashes, Streams, HLL, Geo | Strings only |
| **Persistence** | Optional (RDB snapshots, AOF) | None (pure cache) |
| **Replication** | Primary-replica built-in | No built-in |
| **Pub/Sub** | Yes | No |
| **Lua scripting** | Yes (atomic operations) | No |
| **Max value size** | 512 MB | 1 MB |
| **Threading** | Single-threaded core (6.0+ I/O threads) | Multi-threaded |
| **Use case** | Cache + data-structure server + queues + pub/sub | Pure high-throughput cache |

---

## 3. Redis kyu itna versatile — data structures ka fayda

Memcached me sab kuch "string" hai — koi bhi complex cheez tumhe serialize/deserialize karni padti hai. Redis native structures deta hai jo **is module ke teeno tools** power karte hai (yaad hai DB lesson 07):

```
String       → simple cache (GET/SET)          → caching
Hash         → object fields (user profile)     → caching
List         → queues, recent-N                 → feeds
Sorted Set   → leaderboards, sliding-window     → rate limiting (lesson 08)!
Set          → unique membership, tags          → dedup
HyperLogLog  → approximate unique count         → analytics (lesson 09)
Bitmap       → bloom-filter-style bits          → bloom filters (lesson 09)
Pub/Sub      → cache invalidation events (CDC)  → invalidation (lesson 04)
Lua script   → atomic rate-limit check          → rate limiting (lesson 08)!
```

Matlab Redis sirf cache nahi — ye poore performance-module ka **Swiss Army knife** hai. Isliye ye teeno topics (caching, rate limiting, bloom-ish) ek hi engine pe baithte hai.

---

## 4. Persistence — cache vs primary store (recap)

Yaad hai DB lesson 10/14 — Redis ki durability:
- **RDB (snapshots):** periodically poora dataset disk pe dump. Fast restart, but snapshots ke beech ka data crash pe lost.
- **AOF (Append-Only File):** har write log karo (WAL jaisa). Better durability, replay se recover.

Memcached me **kuch persistence nahi** — pure cache, crash = sab gaya (aur wo theek hai, kyunki wo sirf cache hai). Redis me persistence **optional** hai — chaho toh pure cache, chaho toh durable primary store.

---

## 5. Kab kaunsa — the verdict

**Interview me: hamesha Redis bolo.** Reasoning: *"Redis wo sab karta hai jo Memcached karta hai, plus data structures, persistence, replication, pub/sub. Flexibility zyada, aur mere caching + rate-limiting + other needs ek hi engine se puri ho jaati hai."*

**Memcached sirf tab** jab specifically poocha jaye: *"agar tumhe pure, simple caching chahiye maximum throughput ke saath aur koi data structures nahi chahiye?"* — tab Memcached ka multi-threaded simplicity edge de sakta hai (raw string GET/SET pe thoda faster, simpler ops).

```
Default choice                    → Redis (99% cases)
Sirf dumb string cache, max QPS,  → Memcached
  multi-core squeeze
```

---

## 6. Practical notes

- **Redis single-threaded** (core) — ek command ek time. Isliye ek slow command (e.g., `KEYS *` on huge dataset, ya `SMEMBERS` on giant set) **poore Redis ko block** kar deta hai. Production me aise O(N) commands avoid karo (DB lesson 07 ki complexities yaad karo).
- **Redis scaling:** ek instance ki RAM/throughput limit ho toh — **Redis Cluster** (sharding across nodes — DB lesson 13 concept) ya read replicas (DB lesson 12). Consistent hashing (DB lesson 11) yahan key distribution ke liye use hoti hai.
- **Memory management:** `maxmemory` + eviction policy (lesson 04) set karo, warna Redis RAM bhar ke crash/swap kar sakta hai.

---

## Quick Self-Check
1. Redis aur Memcached ka core difference (ek line)?
2. Redis ke 3 aise features batao jo Memcached me nahi hai.
3. Redis ke data structures is module ke teeno tools (cache/rate-limit/bloom) me kaise use hote hai?
4. RDB vs AOF persistence — difference? Memcached me persistence kyu nahi?
5. Interview me default kya bolna chahiye, aur Memcached kab?
6. Redis single-threaded hone se kaunsa practical danger (slow commands)?
7. Ek Redis instance chhota pad jaye toh scale kaise (2 tarike, DB module se)?

---
**Next:** `07-Rate-Limiting-Algorithms.md` — ab system ko overload/abuse se bachana.