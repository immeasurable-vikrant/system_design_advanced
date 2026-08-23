# 06 — Scaling & Performance Decisions (+ FastAPI)

> **Is lesson me:** Scaling decisions ka framework — kab cache, kab replicate, kab shard, kab CDN — ek clear progression (Topics 1-3 ko decision-form me). Plus FastAPI apps ko scale karna (workers, caching, rate limiting, stateless). "Kaunsa bottleneck pe kya" pe focus.

Prerequisites: `03-Capacity-Estimation.md`, Topic 1-3 (infra, DB scaling, caching)

---

## 1. The scaling ladder — one step at a time ⭐

> **Recap (Topic 1 lesson 04, Topic 3 lesson 01):** har step ek specific bottleneck pe trigger, preemptively nahi.

```
STAGE 1: Single server (app + DB)              → 100s of users
STAGE 2: App aur DB alag machines              → 1000s
STAGE 3: Load balancer + multiple app servers  → 10K (stateless! Topic 1)
STAGE 4: Add cache (Redis) + CDN               → 50K (read-heavy relief)
STAGE 5: Read replicas                         → 200K (read scaling — Topic 2)
STAGE 6: Shard database                        → write-heavy (Topic 2 lesson 13)
STAGE 7: Multi-region + async everything        → global scale
```

**Golden rule (repeat):** *estimation (lesson 03) batati hai tum kaunsi stage pe ho. Uske bottleneck ka jawaab do — future stages ko preemptively mat build karo.*

---

## 2. The decision cascade — cheapest fix first

Jab "slow / can't handle load", is order me socho (cheapest → most complex):
```
1. INDEX?        → DB query slow? Index laga hai? (Topic 2 lesson 06) — cheapest
2. CACHE?        → Same data baar-baar? Cache it (Topic 3) — huge win, low cost
3. REPLICATE?    → Read-heavy? Read replicas (Topic 2 lesson 12)
4. CDN?          → Static/media/global? CDN (Topic 1 lesson 07)
5. SCALE OUT?    → App CPU-bound? More instances + LB (Topic 1)
6. SHARD?        → Write/storage bottleneck? Shard (Topic 2 lesson 13) — last resort
7. ASYNC?        → Spiky/heavy work? Queue + workers (Topic 4)
```

> **Recap (Topic 3 lesson 01):** "optimize the bottleneck, not the fast part." Pehle measure (profiling, slow query logs, APM) — phir sahi jagah fix. App tier scale karna jabki DB bottleneck hai = galat.

**Interview line:** *"Bottleneck pehle measure karunga. DB query slow → index. Repeated reads → cache (biggest bang). Read-heavy → replicas. Static/global → CDN. App CPU-bound → scale out. Sirf jab writes/storage bottleneck ho → shard (last resort, complex). Har step ek measured bottleneck ka jawaab."*

---

## 3. Caching decision (Topic 3 recap)

> **Recap (Topic 3 lesson 02-05):**
```
Cache karo:     read-heavy, expensive-to-compute, staleness-tolerant data
Mat karo:       write-heavy, always-fresh-needed, personalized-unique data
Strategy:       cache-aside (default), TTL + LRU eviction
Watch out:      stampede, hot keys, penetration (Topic 3 lesson 05)
```

**FastAPI caching:**
```python
import redis.asyncio as redis   # async Redis client (FastAPI async!)
r = redis.from_url("redis://localhost")

@app.get("/products/{id}")
async def get_product(id: int, db=Depends(get_db)):
    cached = await r.get(f"product:{id}")     # cache-aside (Topic 3 lesson 03)
    if cached:
        return json.loads(cached)              # HIT
    product = await fetch_product(db, id)      # MISS → DB
    await r.set(f"product:{id}", json.dumps(product), ex=300)  # TTL 5min
    return product
```
> ⚠️ **Async Redis client** (`redis.asyncio`) use karo, sync nahi — warna event loop block (lesson 08).

---

## 4. Rate limiting decision (Topic 3 recap)

> **Recap (Topic 3 lesson 07-08):** system ko overload/abuse se bachao. Token bucket / sliding window, distributed via Redis.

**FastAPI rate limiting:**
```python
# slowapi (FastAPI-friendly rate limiter, Redis-backed)
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address, storage_uri="redis://...")

@app.get("/api/data")
@limiter.limit("100/minute")     # per-IP; or per-user with custom key
async def get_data():
    ...
```
Distributed (multiple FastAPI instances) → Redis-backed limiter (Topic 3 lesson 08 — centralized counter). API gateway pe bhi enforce kar sakte ho (Topic 1 lesson 08).

---

## 5. FastAPI app scaling — the specifics ⭐

FastAPI ek async app hai. Use scale karna:

### Stateless (Topic 1 lesson 04 — the foundation)
```
FastAPI instances STATELESS rakho:
  Sessions → Redis/JWT (not in-process)
  Files → S3
  Cache → shared Redis
→ Phir kitne bhi instances LB ke peeche, koi bhi request handle kare
```

### Workers (process-level parallelism)
```bash
# Multiple Uvicorn workers behind Gunicorn (or uvicorn --workers)
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
# 4 worker processes = 4 CPU cores use (Python GIL bypass — lesson 08)
```
> **Recap (Topic 1 lesson 09 + lesson 08 preview):** ek Python process (GIL) ek time pe ek CPU core hi use karta hai. Multiple **workers** (processes) = multiple cores. Async concurrency (within a worker) I/O-bound ke liye; workers CPU parallelism ke liye. **Dono chahiye:** async for I/O concurrency + multiple workers for CPU cores.

### Horizontal scale (Topic 1)
```
Container (Docker) → multiple FastAPI containers → Load balancer (Topic 1 lesson 05)
→ Auto-scale on CPU/QPS (Topic 1 lesson 04)
```

---

## 6. The scaling decision checklist

```
1. Measure bottleneck (don't guess)
2. Index → Cache → Replicate → CDN → Scale-out → Shard (cheapest first)
3. FastAPI stateless (sessions/files/cache external)
4. FastAPI: async client for cache/DB (don't block event loop)
5. Workers (processes) for CPU cores + async for I/O concurrency
6. Rate limit (Redis-backed) + graceful degradation (Topic 3, 6)
```

**Interview line (FastAPI):** *"Estimation se stage identify karke bottleneck attack karunga: index → cache (async Redis) → read replicas → CDN. FastAPI stateless (sessions in Redis), behind a load balancer, multiple Uvicorn workers per instance (CPU cores) + async I/O (concurrency), auto-scaled. Rate limiting Redis-backed. Writes bottleneck hone pe hi shard."*

---

## Quick Self-Check
1. Scaling ladder ke stages — kaunsa bottleneck kaunsi stage trigger karta?
2. Decision cascade (cheapest first) — index se shard tak order?
3. "Optimize the bottleneck" — pehle kya karo (measure)?
4. FastAPI me async Redis client kyu (sync ka problem)?
5. FastAPI rate limiting distributed kaise (Redis-backed — Topic 3)?
6. FastAPI stateless kyu (Topic 1) — sessions/files/cache kaha?
7. Uvicorn workers vs async — kaunsa CPU cores, kaunsa I/O concurrency? Dono kyu?

---
**Next:** `07-Architecture-Style-Decisions.md` — monolith/microservices, patterns (+ FastAPI structure).