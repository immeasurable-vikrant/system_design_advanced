# 10 — The Performance Playbook (Putting It All Together)

> **Is lesson me:** Poore module ka nichoD — ek repeatable playbook jisse kisi bhi system me performance layer design karo, phir 3 worked examples (URL shortener, news feed, public API) jinme caching + rate limiting + bloom filters + CDN saath me kaam karte hai. Ye tumhara interview + real-work reference hai.

Prerequisites: **saare pichle lessons (01-09).**

---

## 1. The Performance Playbook — 6 steps

Jab koi bole "make this fast / handle scale", is order me socho:

```
STEP 1 — MEASURE: bottleneck kaha? (DB query? network? which endpoint?)
         → optimize the bottleneck, not the fast part (lesson 01)

STEP 2 — READ PATH fast karo:
         → Frequently-read, slow-to-compute data → CACHE (lesson 02-03)
         → Cache-aside default; TTL + LRU (lesson 03-04)
         → Static/media → CDN (lesson 02)

STEP 3 — CACHE ko robust banao:
         → Stampede? → lock / stale-while-revalidate (lesson 05)
         → Hot keys? → local cache (L3) (lesson 05)
         → Penetration? → bloom filter / cache negatives (lesson 05, 09)

STEP 4 — PROTECT the system:
         → Rate limit at edge (token bucket / sliding window) (lesson 07)
         → Distributed → Redis + Lua (lesson 08)
         → Graceful degradation + circuit breaker (lesson 05, 08)

STEP 5 — AVOID expensive lookups:
         → "Definitely absent?" checks → bloom filter (lesson 09)
         → "How many unique?" → HyperLogLog (lesson 09)

STEP 6 — VERIFY: hit ratio 90%+? p99 latency down? DB load down?
         → warna TTL / strategy / key-design tune karo
```

**Golden principle:** *Cache the read path, protect the write path, skip the impossible path.* Caching reads fast karta hai, rate limiting writes/system bachata hai, bloom filters bekaar lookups skip karte hai.

---

## 2. Where each tool sits (the mental map)

```
                         ┌─── RATE LIMITER (edge) ──── requests cap (lesson 7-8)
                         │
Client ──▶ CDN ──▶ Load Balancer ──▶ App Server ──▶ Redis Cache ──▶ Database
          (L2       (lesson 2)         │            (L4, lesson 2)    │
          cache)                       │                              │
                                       └── BLOOM FILTER check ────────┘
                                           "definitely absent?" (lesson 9)
                                           → skip cache+DB
```
- **CDN** — static/media, user ke paas
- **Rate limiter** — edge pe, overload rok
- **Redis cache** — hot data, DB se pehle
- **Bloom filter** — DB/cache se pehle, absent cheezein filter

---

## 3. Worked Example A — URL Shortener (redirect-heavy)

**Bottleneck:** massive reads (redirects), har redirect ek `code → long_url` lookup. (DB Topic 2 lesson 16 me isko DB-side design kiya tha; ab performance layer add karte hai.)

**Performance design:**
```
1. CACHE (cache-aside):  "code → url" Redis me. Viral link = 1 DB read,
   phir lakhon requests cache se. Hit ratio 99%+ (popular links dominate).
2. CDN:                  redirect response bhi edge-cacheable (short TTL) —
   popular links CDN se hi resolve, origin tak pahunche hi na.
3. BLOOM FILTER:         (a) generation: "code already exists?" — bloom "no"
   → skip DB check (lesson 09). (b) lookup: "code exist karta?" — bloom "no"
   → 404 turant, DB/cache bypass (cache penetration rok — lesson 05).
4. RATE LIMIT:           per-IP link creation cap (abuse/spam rok) — token
   bucket at API gateway (lesson 07-08).
```
**Result:** 99% redirects cache/CDN se (sub-ms), invalid codes bloom se instantly rejected, spam rate-limited. DB barely touched.

**Tools:** caching (02-03) + CDN (02) + bloom (09) + rate limit (07-08).

---

## 4. Worked Example B — News Feed (read + write heavy)

**Challenge:** har user ka personalized feed (followed users ke recent posts). Reads huge, "seen" dedup chahiye, celebrities hot.

**Performance design:**
```
1. CACHE feed:           precomputed feed Redis List me ("feed:user:123").
   Cache-aside + short TTL. Read = ek Redis call.
2. HOT KEY (celebrity):  celebrity ka data har app server pe local cache
   (L3, 10s TTL) — Redis bottleneck rok (lesson 05).
3. BLOOM FILTER (seen):  "user ne ye post dekha?" — bloom per user. "No"
   → naya post dikhao. Dedup without storing every seen-ID (lesson 09).
4. STAMPEDE guard:       trending post ka cache expire → stale-while-revalidate
   (purana feed serve, background refresh) — DB flood rok (lesson 05).
5. RATE LIMIT:           post creation + API calls per user (lesson 07-08).
6. HYPERLOGLOG:          "is post ke unique viewers" — HLL, ~12KB (lesson 09).
```
**Tools:** caching (02-05) + hot-key (05) + bloom for dedup (09) + HLL (09) + rate limit (07-08).

---

## 5. Worked Example C — Public API (protection-heavy)

**Challenge:** third-party developers tumhari API use karte hai. Abuse rok, fair usage, backend bachao.

**Performance design:**
```
1. RATE LIMIT (core):    per-API-key tiered limits (Free: 100/day,
   Pro: 10K/day) — token bucket, centralized Redis + Lua for global
   accuracy across servers (lesson 08).
2. HTTP semantics:       429 + Retry-After + X-RateLimit-* headers so
   clients self-throttle (lesson 08).
3. CACHE responses:      idempotent GET responses cache-aside (lesson 03),
   TTL per endpoint. Repeated queries DB tak na jaaye.
4. CDN:                  cacheable public responses at edge (lesson 02).
5. FAIL OPEN:            rate-limiter Redis down → allow + alert, coarse
   per-server fallback limit (lesson 08).
6. BLOOM FILTER:         invalid resource IDs → instant 404, DB bypass
   (lesson 09).
```
**Tools:** rate limiting (07-08, star here) + caching (02-03) + CDN (02) + bloom (09).

---

## 6. The interview checklist (bolne layak)

Jab bhi performance/scale poocha jaye, ye 6 poocho:

```
1. BOTTLENECK?      → measure first (DB? network? which path?)
2. CACHEABLE READS? → cache-aside + TTL + right layer (02-04)
3. CACHE RISKS?     → stampede/hot-key/penetration handled? (05)
4. STATIC/MEDIA?    → CDN (02)
5. PROTECTION?      → rate limit at edge + graceful degradation (07-08)
6. EXPENSIVE MISSES?→ bloom filter / HLL (09)
```

---

## 7. Full module concept map (ek nazar)

```
FOUNDATION      performance fundamentals: latency/throughput, numbers (01)
CACHING         fundamentals+layers (02) → strategies (03) → eviction/
                invalidation (04) → failure modes (05) → Redis vs Memcached (06)
RATE LIMITING   algorithms (07) → distributed + HTTP (08)
PROBABILISTIC   bloom filters + HyperLogLog (09)
SYNTHESIS       this playbook (10)
```

**Aur Topic 2 (Database) se connection:** caching database ke saamne baithti hai (lesson 02); write-behind ko WAL/AOF durability chahiye (03, DB lesson 10); distributed cache consistent hashing use karta hai (06, DB lesson 11); rate-limit atomicity same as double-spend fix (08, DB lesson 09); bloom filter LSM-trees ko power karta hai (09, DB lesson 10). **Do topics ek dusre me interlock karte hai.**

---

## Quick Self-Check (capstone)
1. 6-step playbook bina dekhe? "Optimize the bottleneck" pehle kyu?
2. "Cache the read path, protect the write path, skip the impossible path" — teeno kaunse tool?
3. URL shortener me bloom filter ke 2 alag jagah use — kaunse?
4. News feed me hot celebrity key kaise handle, aur "seen" dedup kaise?
5. Public API me rate limiting global-accurate kaise (lesson 08)?
6. Har tool architecture me kaha baithta hai (CDN/rate-limiter/cache/bloom)?
7. Is module ka koi ek concept jo DB Topic 2 se interlock karta hai — batao.

---

**🎉 Topic 3 complete!** Ab tum kisi bhi system ko fast + resilient bana sakte ho — caching se reads, rate limiting se protection, bloom filters se efficiency. Playbook (10) + concept map ko reference rakho.

**Aage:** tumne bola Topic 1 (jo database se pehle aata hai) next doge — wo foundation layer hoga (shayad networking / load balancing / system design basics). Jab wo aayega, main usko in dono topics ke saath tie kar dunga taaki poora picture bane. Filhaal: Topic 2 (Database) + Topic 3 (Caching & Performance) — dono solid ho gaye.