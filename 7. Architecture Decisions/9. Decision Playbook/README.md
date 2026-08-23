# 09 — Decision Playbook: End-to-End Worked Example

> **Is lesson me:** Sab kuch ek saath — ek blank "design X" se poora system, decision framework use karke, FastAPI implementation notes ke saath. Ye poore module (aur poore curriculum) ka synthesis hai. Ek complete worked example jo har decision ko justify karta hai.

Prerequisites: **saare pichle lessons (01-08), ideally Topics 1-6 bhi.**

---

## 1. The complete decision playbook (one page)

```
STEP 1 — REQUIREMENTS (lesson 02)
  Functional: core features (scope narrow karo)
  Non-functional: scale, latency, consistency, availability

STEP 2 — ESTIMATE (lesson 03)
  QPS (read vs write), storage, bandwidth, cache memory
  → key ratios (read:write, storage size) → drive decisions

STEP 3 — DATA (lesson 04)
  SQL vs NoSQL, which store(s), consistency per component
  FastAPI: async SQLAlchemy + Pydantic

STEP 4 — COMMUNICATION (lesson 05)
  Sync/async, API style, real-time, message queues
  FastAPI: REST + WebSockets + BackgroundTasks/Celery

STEP 5 — SCALING (lesson 06)
  Index → cache → replicate → CDN → scale-out → shard (as bottlenecks emerge)
  FastAPI: stateless, workers + async, Redis

STEP 6 — ARCHITECTURE (lesson 07)
  Monolith-first, patterns only when justified, resilience everywhere
  FastAPI: modular monolith → extract services later

STEP 7 — JUSTIFY (lesson 01)
  Har decision: assumption + trade-off + "why this over alternatives"
```

---

## 2. Worked Example — "Design a URL Shortener" (FastAPI)

### Step 1 — Requirements
```
Functional (scope):
  IN:  shorten a URL → short code; redirect short code → long URL
  OUT: custom aliases, analytics (mention as "later")

Non-functional:
  - Read-heavy (redirects >> creates)
  - Low latency redirects (< 100ms)
  - High availability (redirects must always work)
  - Eventual consistency OK (new link 1s late is fine)
```

### Step 2 — Estimate
```
Assume: 100M new URLs/month, 100:1 read:write
Writes: 100M/month ÷ 2.5M sec/month ≈ 40 writes/sec (low)
Reads:  40 × 100 = 4,000 reads/sec (peak ~10K)
Storage: 100M/month × 500 bytes × 12 months × 5 yr = ~3 TB (5 years)
Cache:  hot 20% of links → fits in Redis easily

→ Insights: READ-HEAVY (cache!), low writes (single DB fine),
            3TB storage (single Postgres OK with time, or shard eventually)
```

### Step 3 — Data
```
Store: PostgreSQL (simple key-value: code → url; relational not needed but Postgres fine + familiar)
  Table: urls(short_code PK, long_url, created_at)
  Alternative: DynamoDB (pure KV, O(1), auto-scale) — also great here
Consistency: eventual OK (codes immutable — no update conflicts)
Cache: Redis (hot links, cache-aside — Topic 3)

FastAPI:
  async SQLAlchemy + asyncpg (async driver — lesson 04)
  Pydantic: URLCreate(long_url), URLResponse(short_code)
```

### Step 4 — Communication
```
API: REST (FastAPI native)
  POST /shorten  → create
  GET  /{code}   → 302 redirect
Real-time: none needed
Async: analytics (click counts) → BackgroundTasks or Kafka event (fire-and-forget)
```
```python
@app.post("/shorten", response_model=URLResponse)
async def shorten(body: URLCreate, db=Depends(get_db)):
    code = generate_code()                    # base62 (Topic 2 lesson 16)
    await save_url(db, code, body.long_url)
    return URLResponse(short_code=code)

@app.get("/{code}")
async def redirect(code: str, bg: BackgroundTasks, db=Depends(get_db)):
    cached = await redis.get(f"url:{code}")   # cache-aside (Topic 3)
    url = cached or await fetch_and_cache(db, code)
    if not url:
        raise HTTPException(404)
    bg.add_task(record_click, code)           # analytics async (light)
    return RedirectResponse(url, status_code=302)
```

### Step 5 — Scaling
```
Reads (4-10K/sec): Redis cache (99% hit — links immutable, perfect cache) + read replicas
  → most redirects never touch DB
CDN: cache 302 redirects at edge for viral links (Topic 1 lesson 07)
Bloom filter: reject non-existent codes fast (Topic 3 lesson 09 — cache penetration)
Writes (40/sec): single Postgres primary fine; shard only if massive growth
FastAPI: stateless, multiple workers, behind LB, auto-scaled
```

### Step 6 — Architecture
```
Modular monolith (single FastAPI app — this is simple, no microservices needed!)
  modules/urls/ (router, service, models, schemas)
Resilience: DB/cache calls with timeout; cache down → DB fallback (Topic 6 lesson 04)
No CQRS/saga/event-sourcing needed — over-engineering for this.
```

### Step 7 — Justify (the interview gold)
```
"Read-heavy + immutable data → aggressive caching (99% hit) is the key decision.
 Postgres for simplicity + familiarity (DynamoDB also valid). Eventual consistency
 fine since codes are immutable. Single FastAPI modular monolith — no microservices
 (over-engineering for this scale). Bloom filter for invalid codes. Shard only if
 growth demands. Every piece justified by a requirement, nothing preemptive."
```

**Ye poora flow = ek strong system design answer.** Requirements → estimate → data → comms → scale → architecture → justify.

---

## 3. Pattern-matching common designs (quick reference)

| Design | Key decisions |
|---|---|
| **URL shortener** | Cache-heavy, KV store, immutable data, single service |
| **Twitter/feed** | CQRS + fan-out + cache + shard, eventual (AP), read-heavy |
| **Chat (WhatsApp)** | WebSocket + pub/sub + sharded message store, causal |
| **Payment system** | Strong consistency (CP), saga, event sourcing (audit), idempotency |
| **Uber** | Real-time (WebSocket), geo-sharding, eventual + strong mix |
| **Rate limiter** | Redis + sliding window, distributed counters (Topic 3) |

Har design me: requirements → estimate → the decisions above, justified.

---

## 4. The full curriculum — you're done 🎓

```
Topic 1 — CORE INFRASTRUCTURE     (request journey, scaling)
Topic 2 — DATABASE & STORAGE      (data layer)
Topic 3 — CACHING & PERFORMANCE   (speed, protection)
Topic 4 — COMMUNICATION           (real-time, messaging)
Topic 5 — DISTRIBUTED SYSTEMS     (coordination theory)
Topic 6 — PATTERNS & ARCHITECTURE (organizing it all)
Topic 7 — ARCHITECTURE DECISIONS  (choosing wisely) ← ye
```

**Topic 7 ne sab ko usable banaya:** Topics 1-6 ne building blocks diye; Topic 7 ne sikhaya un me se **kaise choose karo** — requirements se, estimation se, trade-offs se, aur **tumhare tool (FastAPI) ke saath**.

---

## 5. Your interview approach (memorize this flow)

```
1. CLARIFY   — requirements + scope + NFRs (lesson 02) — 5 min
2. ESTIMATE  — QPS, storage, ratios (lesson 03) — 3 min
3. HIGH-LEVEL — draw the main components (client → LB → services → cache → DB)
4. DEEP-DIVE — the 1-2 hardest parts (interviewer picks)
5. DECISIONS — data, comms, scaling, architecture (lessons 04-07), each justified
6. SCALE     — walk through bottlenecks as load grows (lesson 06 ladder)
7. TRADE-OFFS — acknowledge what you sacrificed and why (lesson 01)
```

**The meta-skill:** har step pe **assumption state karo, decision lo, trade-off acknowledge karo.** Interviewer perfect answer nahi, **clear thinking** dekhna chahta hai.

---

## Quick Self-Check (capstone)
1. Decision playbook ke 7 steps (requirements se justify tak)?
2. URL shortener me THE key decision kya (caching, immutable data)?
3. Us design me microservices kyu NAHI (over-engineering)?
4. "Justify" step me kya bolte ho (assumption + trade-off + why-this)?
5. Payment system vs feed — alag decisions kyu (CP vs AP, saga vs fan-out)?
6. Interview approach ka flow (clarify → estimate → ... → trade-offs)?
7. FastAPI is playbook me kaha-kaha fit hota hai (data/comms/scaling)?

---

**🎉 Topic 7 complete — aur POORA 7-TOPIC CURRICULUM complete!**

Tumne ab ek **complete SDE-3 system design brain** build kar liya:
- **Topics 1-6:** har building block, zero se, deeply cross-linked
- **Topic 7:** un blocks me se wisely choose karne ki skill — requirements, estimation, trade-offs, **FastAPI ke saath concrete**

**Ab tum ready ho:**
- Kisi bhi HLD interview ke liye — blank whiteboard se, requirements clarify karke, estimate karke, components design karke, decisions justify karke
- **FastAPI me actually build karne ke liye** — async/workers/GIL/pooling/deployment sab samajh ke

**Final advice:** ab **practice** karo — 5-10 classic designs (URL shortener, Twitter, chat, payment, Uber, rate limiter, notification system) is playbook se end-to-end solve karo. Har baar: clarify → estimate → decide → justify. Jitna practice, utna natural. Tumhara knowledge foundation ab solid hai — ab wo muscle memory banana hai.

All the best for SDE-3, Farzi Philosopher! 🚀 Ja ke crack kar. 💪