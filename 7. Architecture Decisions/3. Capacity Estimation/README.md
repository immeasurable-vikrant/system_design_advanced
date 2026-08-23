# 03 — Capacity Estimation (Back-of-the-Envelope)

> **Is lesson me:** Ek design ke numbers kaise nikaalo — QPS (queries/sec), storage, bandwidth, memory — quick mental math se. Ye numbers har architecture decision drive karte hai (100 QPS vs 1M QPS = bilkul alag design). Interview me ye ek core skill hai.

Prerequisites: `02-Requirements-And-NFRs.md`

---

## 1. Estimation kyu — numbers drive design

Bina numbers, tum nahi jaante system ko kitna handle karna hai — aur isliye kaunsi cheezein chahiye:
```
100 requests/sec  → ek server + ek DB kaafi
100K requests/sec → LB + many servers + cache + read replicas
1M requests/sec   → + sharding + CDN + async everything
```

**Interview me estimation dikhana = tum scale samajhte ho.** Perfect numbers nahi chahiye — **right order of magnitude** (10x range) chahiye. "~50K QPS" bolna theek hai, "51,247 QPS" nahi.

---

## 2. Numbers you must memorize ⭐

Ye reference points ratne padte hai (estimation inpe chalti hai):

### Powers of 2 (data sizes)
```
2^10 = 1 Thousand  (KB)
2^20 = 1 Million   (MB)
2^30 = 1 Billion   (GB)
2^40 = 1 Trillion  (TB)
```

### Time (QPS conversion)
```
1 day = 86,400 seconds ≈ 100,000 (~10^5) — ye approximation use karo!
1 million/day  ÷ 100K = ~10 per second
1 billion/day  ÷ 100K = ~10,000 per second
```

### Latency numbers (Topic 1 lesson 01 recap)
```
Memory (RAM) read:     ~100 ns
SSD read:              ~100 µs (1000x slower than RAM)
Network round-trip:    ~500 µs (same DC) / ~150 ms (cross-country)
```

### Typical sizes
```
A character/int:  ~1-8 bytes
A tweet/message:  ~200-300 bytes
A metadata row:   ~1 KB
An image:         ~1-5 MB
A video (minute): ~50 MB
```

---

## 3. The estimation method — 4 quantities

Zyadatar designs me ye 4 nikaalo:
```
1. QPS (queries/sec)  → traffic (reads + writes alag)
2. STORAGE            → data volume over time
3. BANDWIDTH          → data transfer/sec
4. MEMORY (cache)     → hot data jo cache karna hai
```

### Step-by-step (Twitter example)
**Given:** 300M DAU, har user 2 tweets/day post karta, 100 tweets/day padhta.

**QPS:**
```
Writes: 300M users × 2 tweets = 600M tweets/day
        600M ÷ 100K (sec/day) = 6,000 writes/sec
        Peak (2-3x average) = ~15,000 writes/sec

Reads:  300M × 100 reads = 30B reads/day
        30B ÷ 100K = 300,000 reads/sec
        Peak = ~600,000 reads/sec

→ Read:write ratio ~50:1 (READ-HEAVY!)
  → ye decision drive karta hai: caching + read replicas zaroori (Topic 2, 3)
```

**Storage:**
```
Per tweet: ~300 bytes (text + metadata)
Per day:   600M × 300 bytes = 180 GB/day
Per year:  180 GB × 365 = ~66 TB/year
5 years:   ~330 TB
→ ye decision drive karta hai: sharding zaroori (ek machine me nahi aayega — Topic 2)
```

**Bandwidth:**
```
Write bandwidth: 6,000 tweets/sec × 300 bytes = ~1.8 MB/sec (small)
Read bandwidth:  300K reads/sec × 300 bytes = ~90 MB/sec
→ manageable, but media (images/video) add kare toh CDN zaroori (Topic 1)
```

**Cache memory (80/20 rule):**
```
80% traffic 20% tweets pe. Cache hot 20%.
Daily tweets: 600M. Cache ~20% of a day's = 120M tweets × 300 bytes = ~36 GB
→ ek Redis cluster me aa jaayega (Topic 3)
```

---

## 4. The key ratios that drive decisions ⭐

Estimation ka asli output = **ratios/insights** jo architecture decide karte hai:

| Finding | Decision it drives |
|---|---|
| Read:write = 50:1 | Caching + read replicas (read-heavy) — Topic 2, 3 |
| Storage > 1 TB | Sharding (single machine won't fit) — Topic 2 |
| Write QPS > ~10K | Sharding / write scaling — Topic 2 |
| Peak >> average | Auto-scaling + queue buffering — Topic 1, 4 |
| Media-heavy | Object storage (S3) + CDN — Topic 2, 1 |
| Global users | Multi-region + CDN + geo-routing — Topic 1, 5 |

**Interview line:** *"Estimate karke: ~300K read QPS vs ~6K write QPS — 50:1 read-heavy. Storage ~66 TB/year — single DB me nahi aayega. Ye do numbers mera design drive karte hai: read-heavy → aggressive caching + read replicas; large storage → sharding. Peak traffic 2-3x average, toh auto-scaling + queue buffering."*

---

## 5. FastAPI context — capacity per instance

Kyunki tum FastAPI use karoge, ek concrete reference (approx):
```
Ek FastAPI (async) instance, I/O-bound work (DB/API calls):
  → ~1,000-5,000 req/sec handle kar sakta (async concurrency ke saath)
  (exact hardware, work type pe depend)

CPU-bound work (heavy computation):
  → bahut kam (Python GIL — lesson 08), workers/processes chahiye

Estimation → instances:
  300K read QPS ÷ ~3K per instance = ~100 FastAPI instances
  → behind a load balancer (Topic 1), auto-scaled
```
> **Detail lesson 08 me** — FastAPI async I/O-bound ke liye great hai (jaise ye read-heavy API), but CPU-heavy kaam ke liye alag approach chahiye (GIL). Estimation ye batati hai kitne instances + kaunsa work model.

---

## 6. Estimation tips (don't overthink)

```
1. Round aggressively — 86,400 → 100,000; 24 → 25 → easy math
2. Powers of 10 — exact number nahi, order of magnitude (10x range)
3. Reads vs writes ALAG — ratio sabse important insight
4. Peak = 2-3x average — design for peak, not average
5. State assumptions — "maan lo har user 2 tweets/day" (interviewer correct karega agar galat)
6. Numbers → decisions — estimation ka point decisions drive karna hai, math dikhana nahi
```

**Frontend analogy:** ye bilkul performance budgeting jaisa hai — "bundle < 200KB, LCP < 2.5s, ye page X requests karega." Tum numbers estimate karke decide karte ho kya optimize karna hai. Backend estimation wahi, infrastructure scale pe.

---

## Quick Self-Check
1. Estimation kyu — numbers design kaise drive karte hai (100 vs 1M QPS)?
2. `1 day ≈ ? seconds` (approximation)? 1 billion/day = kitne per second?
3. QPS kaise nikaalte hai (users × actions ÷ seconds)? Peak kaise?
4. Read:write ratio kyu sabse important insight hai?
5. Storage estimate → kaunsa decision (sharding kab)?
6. 80/20 rule cache memory estimate me kaise?
7. FastAPI async instance approx kitna I/O-bound QPS? CPU-bound kyu kam (GIL)?
8. Estimation ke 3 tips (rounding, ratios, peak)?

---
**Next:** `04-Data-Storage-Decisions.md` — SQL vs NoSQL, which DB (+ FastAPI).