# 01 — Performance Fundamentals (The Mindset)

> **Is lesson me:** "Performance" ka asli matlab — latency vs throughput vs bandwidth, wo famous "numbers every engineer should know" (kaunsa operation kitna slow), bottleneck kaha hota hai, aur kyu caching/rate-limiting/bloom-filters teeno exist karte hai. Ye poore module ka foundation hai.

Prerequisites: DB Topic 2 (especially lesson 04 — disk vs RAM). Warna bhi chalega.

---

## 1. "Fast" ka matlab kya — 3 alag cheezein

Log "system fast hai" bolte hai but iske 3 alag meanings hai. Inhe mix mat karo:

| Term | Matlab | Analogy |
|---|---|---|
| **Latency** | Ek single request ko complete hone me kitna time | Ek car ko A→B pahunchne me kitne minute |
| **Throughput** | Per second kitne requests handle ho sakte hai | Highway pe per hour kitni cars guzri |
| **Bandwidth** | Ek time me kitna data move ho sakta hai | Highway kitni lane chaudi hai |

**Key insight:** Ye independent hai. Ek system **low latency** (har request fast) but **low throughput** (ek time me kam requests) ho sakta hai, ya ulta. Design me tum decide karte ho kis ko optimize karna hai.

**Frontend analogy:**
- Latency = ek API call ko response aane me kitne ms (tumhare Network tab me "Time" column).
- Throughput = tumhara server per second kitni API calls handle kar sakta hai before slowing down.
- Ek slow API (high latency) bhi high throughput de sakta hai agar parallel bahut handle kare; ek fast API low throughput ho sakta hai agar ek-ek karke kare.

---

## 2. Numbers every engineer should know (the latency hierarchy)

Ye internalize karo — **kaunsa operation kitna slow hai** — kyunki har performance decision inhi pe based hai:

```
Operation                          Approx time        RAM ke mukable
─────────────────────────────────────────────────────────────────
L1 cache reference                 ~1 ns              baseline
Main memory (RAM) reference        ~100 ns            1x
Read 1MB sequentially from RAM     ~10,000 ns (10µs)
SSD random read                    ~100,000 ns        ~1,000x slower than RAM
Read 1MB from SSD                  ~1,000,000 ns (1ms)
Network round trip (same DC)       ~500,000 ns        ~5,000x
HDD seek (spinning disk)           ~10,000,000 ns     ~100,000x
Network round trip (cross-country) ~150,000,000 ns    ~1,500,000x
```

**Do bade takeaways:**
1. **RAM ≈ 1000x faster than disk.** Isliye caching (data ko RAM me rakhna) itna powerful hai. Yahi lesson pura caching module drive karta hai.
2. **Network round-trips mehenge hai** — especially cross-region. Isliye "ek request me kitne network hops" minimize karna performance ka core hai (CDN edge, connection pooling, batching — sab isi liye).

> **DB lesson 04 recap:** waha humne dekha tha DB apna hot data RAM buffer-cache me rakhta hai kyunki disk slow hai. Caching wahi principle ek layer upar le jaata hai — application ke saamne ek dedicated RAM layer (Redis).

---

## 3. Bottleneck kaha hota hai (jaha optimize karna hai)

Ek typical web request ka time kaha jaata hai:

```
Client ──(network)── Load Balancer ──(network)── App Server ──(network)── Database
  1ms         50ms                    1ms            2ms          5-50ms (query)
                                                                  ▲
                                                          usually YAHI slowest
```

99% cases me **database (disk-bound queries)** ya **network round-trips** bottleneck hote hai — CPU nahi. Isliye:
- **Caching** → database ko baar-baar chhune se bachao (biggest win)
- **Rate limiting** → database/servers ko overload se bachao
- **Bloom filters** → useless database/disk lookups avoid karo

**Golden rule:** *Optimize the bottleneck, not the fast part.* CPU-level micro-optimization tab tak bekaar hai jab tak tumhara real time database/network me ja raha hai. Pehle measure karo (jaise DB ka `EXPLAIN ANALYZE`, ya APM tools) — phir sahi jagah fix karo.

---

## 4. Ye teen tools kyu — ek unified picture

Poora ye module 3 tools sikhata hai, aur teeno ek hi latency-hierarchy problem ke alag pehlu solve karte hai:

```
Problem                              Tool             Kaise
───────────────────────────────────────────────────────────────────
"Same slow query baar-baar"          CACHING          Result RAM me rakho
                                                       (disk skip = 1000x faster)

"Too many requests, system dubega"   RATE LIMITING    Extra requests reject karo
                                                       (bottleneck ko bachao)

"Non-existent cheez ke liye         BLOOM FILTER     "Definitely nahi hai" turant
 expensive lookup"                                    RAM me answer (disk/DB skip)
```

Teeno **Redis pe** implement hote hai (yaad hai DB lesson 07/14 — Redis in-memory + rich data structures). Isliye ye module Redis-heavy hai.

---

## 5. Performance optimization ka order (mental checklist)

Jab koi bole "system slow hai / scale karna hai", is order me socho:

```
1. MEASURE     → bottleneck kaha? (profiling, APM, slow query logs)
2. INDEX       → DB queries indexed hai? (DB lesson 06)
3. CACHE       → repeated reads cache ho sakte hai? (is module)
4. REPLICATE   → read load replicas pe? (DB lesson 12)
5. SHARD       → data/writes bahut zyada? (DB lesson 13)
6. PROTECT     → rate limit + graceful degradation (is module)
7. PRECOMPUTE  → bloom filters, materialized views, denormalize
```

Caching (step 3) usually **sabse bada bang-for-buck** deta hai — isliye wo is module ka dil hai.

---

## Quick Self-Check
1. Latency, throughput, bandwidth — teeno me difference, ek analogy se?
2. RAM disk se kitni guna fast (order of magnitude)? Ye caching ke liye kyu matter karta hai?
3. Ek typical web request me bottleneck usually kaha hota hai — CPU ya DB/network?
4. "Optimize the bottleneck, not the fast part" ka matlab?
5. Caching, rate limiting, bloom filters — teeno kaunsa common problem solve karte hai, aur teeno kis tech pe run karte hai?

---
**Next:** `02-Caching-Fundamentals-And-Layers.md` — caching ko full depth me.