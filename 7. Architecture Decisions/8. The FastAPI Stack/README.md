# 08 — The FastAPI Stack (Concrete Decisions)

> **Is lesson me:** FastAPI-specific architecture decisions in depth — ASGI vs WSGI, `async def` vs `def` (aur GIL), workers vs async concurrency, background tasks, deployment, aur kab FastAPI use karo vs nahi. Ye tumhare actual build tool ka deep-dive hai, prior topics se connected.

Prerequisites: `04-05-06-07` (decision lessons), Topic 1 lesson 01-02 (ASGI/HTTP), lesson 09 (pooling)

---

## 1. FastAPI kya hai (stack ke terms me)

**FastAPI** = ek modern async Python web framework, teen cheezon pe bana:
```
Starlette  → ASGI framework (async HTTP, WebSockets, routing)
Pydantic   → data validation + serialization (type-based)
Uvicorn    → ASGI server (jo actually requests serve karta)
```
FastAPI in teeno ko combine karke ek fast, type-safe, auto-documented API framework deta hai.

---

## 2. ASGI vs WSGI — kyu FastAPI fast hai ⭐

Python web frameworks 2 tarah ke hai:
```
WSGI (old — Flask, Django sync):
  Synchronous. Ek request handle hone tak worker blocked.
  Concurrency ke liye = zyada workers/threads.

ASGI (new — FastAPI, async):
  Asynchronous. Ek worker MULTIPLE requests concurrently handle kare —
  jab ek request I/O (DB/API) ka wait kare, worker doosri request pe kaam kare.
```

> **Recap (Topic 1 lesson 01):** network/DB calls slow hai (I/O wait). WSGI me us wait me worker idle blocked rehta. ASGI (async) me worker us wait me doosra kaam karta — **I/O-bound workloads ke liye massive concurrency** ek hi process me.

**Frontend analogy (tumhe ye pata hai!):** JS event loop — `await fetch()` pe main thread block nahi hota, doosra kaam chalta hai, response aane pe callback. FastAPI async **exactly wahi** — Python me event loop, `await` pe worker doosri request serve karta. Tumhara JS async mental model directly transfer hota hai.

```python
@app.get("/user/{id}")
async def get_user(id: int):
    user = await db.fetch(id)       # I/O wait — worker MEANWHILE doosri request serve kare
    posts = await api.get_posts(id)  # aur I/O — concurrent
    return {"user": user, "posts": posts}
```

---

## 3. `async def` vs `def` — the critical decision ⭐

FastAPI dono support karta hai, but semantics ALAG hai:
```
async def endpoint:  event loop me chalta. I/O ke liye `await` use karo.
                     ⚠️ Blocking call (sync DB, time.sleep, heavy CPU) yahan
                        POORE event loop ko block karega → sab requests atak jaayengi!

def endpoint:        FastAPI ise ek THREADPOOL me chalata (event loop free rehta).
                     Sync libraries safe hai yahan (blocking thread me hai, loop me nahi).
```

**The rule:**
```
I/O-bound + async libraries available  → async def + await (best concurrency)
Sync-only libraries (no async version) → plain def (threadpool me safe)
CPU-heavy work                          → NEITHER endpoint — offload to workers (section 5)
```

> ⚠️ **#1 FastAPI mistake:** `async def` endpoint me ek **blocking/sync call** karna (jaise sync `requests.get()`, sync DB driver, `time.sleep()`, ya heavy computation). Ye event loop ko block karta hai → poora server freeze (sab concurrent requests atak jaati hai). **Async endpoint me sirf async (`await`) I/O, ya blocking cheez ko threadpool/worker me bhejo.**

**Examples:**
```python
# ✅ GOOD — async I/O
async def good(id: int):
    return await async_db.fetch(id)      # async driver (asyncpg)

# ❌ BAD — blocking in async
async def bad(id: int):
    return sync_db.query(id)              # sync driver blocks event loop!

# ✅ OK — sync endpoint, FastAPI uses threadpool
def also_ok(id: int):
    return sync_db.query(id)              # blocking, but in threadpool (loop free)
```

---

## 4. The GIL — Python's CPU limitation ⭐

**GIL (Global Interpreter Lock)** = Python me ek time pe sirf **ek thread** Python bytecode execute kar sakta hai. Matlab: **Python threads CPU-bound work ko parallelize nahi karte.**

```
I/O-bound work (DB/API/file):
  GIL released during I/O wait → async/threads GREAT concurrency dete hai ✅
  (FastAPI async yahan shine karta)

CPU-bound work (image processing, ML inference, heavy computation):
  GIL held → threads/async NAHI help karte (ek core hi use hota) ❌
  → SOLUTION: multiple PROCESSES (workers) — har process apna GIL
```

**Iska architecture impact:**
```
FastAPI I/O-bound APIs ke liye EXCELLENT (async concurrency, GIL released on I/O)
FastAPI CPU-bound work ke liye careful:
  → Offload to worker processes (Celery/Arq — lesson 05)
  → Ya ek alag service in Go/Rust (CPU-heavy microservice)
  → Ya multiprocessing / ProcessPoolExecutor
```

**Frontend analogy:** JS bhi single-threaded hai (like GIL-ish) — CPU-heavy kaam main thread ko block karta hai, isliye tum Web Workers me offload karte ho. Python me CPU-heavy = process workers me offload. Same pattern.

---

## 5. Workers vs async — dono chahiye ⭐

Confusion clear karo — ye do alag cheezein hai, dono use karo:
```
ASYNC (within a process):     I/O concurrency — ek worker kai I/O requests handle kare
WORKERS (multiple processes): CPU parallelism — har process ek core, GIL bypass

Deployment: multiple workers, HAR worker async
  gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
  → 4 processes (4 cores) × each handling many async I/O requests concurrently
  → best of both: CPU cores utilized + I/O concurrency per core
```

**Rule of thumb:** workers ≈ CPU cores (ya `2 × cores + 1`). Async concurrency handles I/O within each.

---

## 6. Deployment stack (FastAPI in production)

```
                    Internet
                       │
              ┌────────▼────────┐
              │  Load Balancer   │  (Topic 1 lesson 05 — Nginx/ALB)
              └────────┬────────┘
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  Container 1     Container 2     Container 3   (Docker, auto-scaled — Topic 1 lesson 04)
  (Gunicorn +     ...             ...
   4 Uvicorn
   workers)
        │
        ├──▶ PostgreSQL (async SQLAlchemy + pool — lesson 04)
        ├──▶ Redis (cache + sessions + rate limit — Topic 3)
        ├──▶ Celery/Arq workers (heavy async — lesson 05)
        └──▶ S3 + CDN (media — Topic 1, 2)
```

**Key production decisions:**
- **Uvicorn workers** behind Gunicorn (process management + graceful restarts)
- **Nginx/LB** in front (TLS termination — Topic 1 lesson 03, static files, buffering — Topic 1 lesson 06)
- **Stateless** containers (sessions in Redis — Topic 1 lesson 04) → horizontal scale
- **Health checks** (`/health` endpoint — Topic 1 lesson 05) for LB
- **Async DB pool** sized right (Topic 1 lesson 09)

---

## 7. Kab FastAPI, kab nahi

**✅ FastAPI great for:**
- REST APIs / microservices (its sweet spot)
- I/O-bound workloads (many DB/API calls — async concurrency)
- ML model serving (async request handling + Python ML ecosystem)
- Real-time (WebSockets native)
- Fast development (auto-docs, validation, type safety)

**⚠️ FastAPI careful/not-ideal for:**
- **CPU-heavy** core work → offload to workers, or a Go/Rust service (GIL)
- **Full batteries-included** needs (admin panel, built-in ORM, templating) → Django might fit better
- **Simple synchronous** app with no async benefit → Flask simpler (though FastAPI still fine)

**Interview line:** *"FastAPI mera default for APIs — async (ASGI) great for I/O-bound (DB/external calls), Pydantic validation, auto-docs, native WebSockets. I/O-bound endpoints async (asyncpg, async Redis); sync-only libs in plain `def` (threadpool). CPU-heavy work (encoding, ML training) offloaded to Celery/Arq workers or a separate service — GIL means Python threads don't parallelize CPU. Deploy: multiple Uvicorn workers (cores) behind Nginx + LB, stateless, auto-scaled."*

---

## Quick Self-Check
1. FastAPI kis 3 cheezon pe bana (Starlette/Pydantic/Uvicorn)?
2. ASGI vs WSGI — async concurrency ka fayda I/O-bound me kyu (JS event loop analogy)?
3. `async def` vs `def` endpoint — semantics? Blocking call async me kyu disaster?
4. GIL kya hai — CPU-bound vs I/O-bound pe impact?
5. Workers vs async — kaunsa CPU parallelism, kaunsa I/O concurrency? Dono kyu?
6. CPU-heavy work FastAPI me kaha (workers/separate service)?
7. FastAPI production deployment stack (LB → containers → workers → DB/Redis)?
8. Kab FastAPI great, kab careful (CPU-heavy, batteries-included)?

---
**Next:** `09-Decision-Playbook-Worked-Example.md` — blank se poora system design (FastAPI).