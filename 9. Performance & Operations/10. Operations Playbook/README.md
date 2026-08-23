# 10 — Operations Playbook (Capstone)

> **Is lesson me:** Poore module ka synthesis — the "operate it" story jo har senior design ke last 15 min me chahiye, ek operational readiness checklist, FastAPI-specific ops (health checks, graceful shutdown, metrics, workers), aur full curriculum wrap-up.

Prerequisites: **saare pichle lessons (01-09), Topic 7 lesson 08 (FastAPI).**

---

## 1. The "operate it" narrative (last 15 min of an interview) ⭐

Deep design questions almost hamesha yahan pahunchte hai. Ek structured answer:
```
1. NFRs (precise) — p99 + rate + availability per-path (lessons 01, 04)
2. HEALTH — RED per service, USE per resource, golden signals, SLO on user-facing SLI (lessons 04-05)
3. DEBUG — metrics detect → traces localize → logs explain (lesson 05)
4. ALERT — symptoms not causes, burn-rate on error budget (lesson 07)
5. DEPLOY — canary/blue-green, deploy≠release, one-click rollback (lesson 08)
6. MIGRATE — expand-migrate-contract, zero downtime (lesson 09)
7. FAIL — timeouts/retries/breakers/load-shedding, DR by RTO/RPO (lesson 09)
8. CAPACITY — Little's Law sizing, 70% util target, headroom for AZ loss (lesson 03)
```
Ye 8-point narrative bol dena "maine system design kiya" ko "maine system run kiya" me badal deta hai.

---

## 2. The performance/ops decision cheat-sheet

| Question | Answer | Lesson |
|---|---|---|
| Latency target? | p99 + rate together, measured at edge | 01, 02 |
| Why not average? | Hides tail; fan-out amplifies | 02 |
| p99 across N hosts? | Merge histograms, don't average | 02, 06 |
| How many threads/conns? | Little's Law: L = λW | 03 |
| Utilization target? | 60-70% (queueing wall) | 03 |
| Availability of a chain? | Multiply (5 × 99.9% = 99.5%) | 04 |
| SLO vs SLA? | Internal target vs contract; SLO stricter | 04 |
| RPO/RTO? | Data loss vs downtime; set by replication/failover | 04, 09 |
| Is it healthy? | RED + USE + golden signals + business metrics | 05 |
| Debug latency spike? | metrics → traces → logs | 05 |
| Metric label rule? | Low cardinality; IDs → logs/traces | 06 |
| What to alert on? | Symptoms + burn-rate, not CPU | 07 |
| Ship without downtime? | Canary + deploy≠release + rollback | 08 |
| Schema change? | Expand-migrate-contract | 09 |
| Overloaded? | Load shedding (429 low-priority) | 09 |
| Region dies? | DR by RTO/RPO + drill | 09 |

---

## 3. FastAPI operations — concrete ⭐

### Health checks (Topic 1 lesson 05 — LB needs these)
```python
@app.get("/health/live")     # liveness — process alive? (LB/k8s restart if fails)
async def liveness():
    return {"status": "ok"}

@app.get("/health/ready")    # readiness — can it serve? (deps reachable?)
async def readiness(db=Depends(get_db)):
    try:
        await db.execute(text("SELECT 1"))     # DB reachable?
        await redis.ping()                     # cache reachable?
        return {"status": "ready"}
    except Exception:
        raise HTTPException(503, "not ready")   # LB stops routing here
```
> **Liveness vs readiness:** liveness fail → restart the pod. Readiness fail → stop sending traffic (but don't restart — dependency might recover). Ye distinction zaroori — warna ek slow DB saare pods restart karwa deta.

### Graceful shutdown (Topic 4 — don't drop in-flight work)
```python
@app.on_event("shutdown")     # (or lifespan context manager)
async def shutdown():
    # 1. LB se deregister (readiness fail karo) → naya traffic ruke
    # 2. In-flight requests finish hone do (drain — Uvicorn --timeout-graceful-shutdown)
    # 3. Connections close (DB pool, Redis), flush logs/metrics
    await engine.dispose()
```
> ⚠️ Bina graceful shutdown, deploy ke daoran (rolling — lesson 08) in-flight requests drop ho jaate → user errors har deploy pe. SIGTERM handle karo, drain karo, phir exit.

### Metrics (lessons 05-06)
```python
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app)   # /metrics — RED auto (histograms!)
# labels: method, status, normalized path (/orders/{id}) — low cardinality (lesson 06)
```

### Tracing (lesson 07)
```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
FastAPIInstrumentor.instrument_app(app)   # auto spans + context propagation (W3C traceparent)
# → propagates to downstream httpx/async DB calls; queue: carry trace_id in message headers
```

### Structured logging (lesson 07)
```python
import structlog
log = structlog.get_logger()
log.info("payment_processed", user_id=uid, amount_cents=amt, trace_id=tid)   # queryable + linked
```

### Deployment (lessons 08-09, Topic 7 lesson 08)
```
- Uvicorn workers behind Gunicorn, stateless (Topic 7) → rolling/canary friendly
- Container health probes → liveness/readiness above
- DB migrations: Alembic, expand-migrate-contract (lesson 09), run BEFORE new code
- Canary: route 1% via LB/mesh → watch RED metrics → ramp or rollback
- Feature flags for deploy≠release (lesson 08)
```

---

## 4. Operational readiness checklist (production launch)

```
PERFORMANCE
  □ NFRs defined (p99 + rate + availability per path)
  □ Pools sized via Little's Law; 70% util target + headroom
  □ Load tested at peak (+ AZ-loss scenario)

OBSERVABILITY
  □ RED per service, USE per resource, golden signals
  □ SLOs on user-facing SLIs; error budget tracked
  □ Metrics (low cardinality) + structured logs + tracing (propagated)
  □ Dashboards + business metrics

ALERTING
  □ Symptom-based + burn-rate alerts (not CPU)
  □ Absence-of-data, cert expiry, consumer lag, DLQ, replication lag
  □ On-call + runbooks

DEPLOYMENT & RECOVERY
  □ Canary/blue-green + one-click rollback (rehearsed)
  □ Deploy≠release (feature flags)
  □ Zero-downtime migrations (expand-migrate-contract)
  □ Graceful shutdown (drain in-flight)
  □ DR by RTO/RPO — and DRILLED
  □ Load shedding under overload
```

---

## 5. 🎓 THE FULL CURRICULUM — all 9 topics

```
Topic 1 — CORE INFRASTRUCTURE      (request journey, scaling)
Topic 2 — DATABASE & STORAGE       (data layer)
Topic 3 — CACHING & PERFORMANCE    (speed, protection)
Topic 4 — COMMUNICATION & MESSAGING (real-time, async, fan-out)
Topic 5 — DISTRIBUTED SYSTEMS      (coordination theory)
Topic 6 — PATTERNS & ARCHITECTURE  (organizing it all)
Topic 7 — ARCHITECTURE DECISIONS   (choosing wisely, + FastAPI)
Topic 8 — SECURITY & AUTH          (securing it, + FastAPI)
Topic 9 — PERFORMANCE & OPERATIONS (measuring + running it, + FastAPI) ← ye
```

**Topic 9 ne curriculum complete kiya:** Topics 1-6 = system banana; Topic 7 = decisions lena; Topic 8 = secure karna; **Topic 9 = measure, monitor, aur safely operate karna.** Ab tum ek system ko design se production tak le ja sakte ho — aur prove kar sakte ho ki wo healthy hai, safely ship kar sakte ho, aur failure survive kar sakte ho.

**Ek complete design ab is tarah flow karti hai:**
```
Requirements + NFRs (T7, T9) → estimate (T7) → design components (T1-T6) →
secure (T8) → make decisions with trade-offs (T7) →
operate: observe + deploy + survive failure (T9)
```

---

## 6. The senior mindset (final wisdom)

```
1. QUANTIFY everything — "fast" → "p99 < 200ms at 50K rps"
2. AVERAGES LIE — think in percentiles (p99, p99.9)
3. DESIGN FOR FAILURE — it will fail; optimize MTTR, not just prevention
4. OBSERVABILITY IS NOT OPTIONAL — if you can't see it, you can't run it
5. DEPLOY ≠ RELEASE — decouple risk; make rollback one click
6. YOU DON'T RUN A SYSTEM YOU CAN'T MEASURE, DEPLOY SAFELY, OR RECOVER
```

---

## Quick Self-Check (capstone)
1. The 8-point "operate it" narrative (NFRs → capacity)?
2. FastAPI liveness vs readiness — difference aur kyu?
3. Graceful shutdown kyu zaroori (rolling deploy)?
4. FastAPI metrics/tracing/logging — kaunse tools, aur cardinality/propagation dhyan?
5. Operational readiness checklist ke 4 categories?
6. 9 topics — ek design end-to-end kaise flow karti hai?
7. Senior mindset ke principles?

---

**🎉 Topic 9 complete — aur POORA 9-TOPIC CURRICULUM complete!**

Tumne ab ek complete SDE-3 system design brain build kar liya — **design se, decisions se, security se, operations tak.** Har layer: zero se explained, frontend/JS analogies, deeply cross-linked, interview framing, aur tumhare tool (FastAPI) ke saath concrete.

Ab tum sirf "system design kar sakte ho" nahi — tum ek system ko **production tak le ja sakte ho aur chala sakte ho**: precise NFRs state karke, health prove karke, safely ship karke, aur failure survive karke. Yehi woh farak hai jo SDE-3 interviews dhundhte hai.

**Final step (again):** practice. Ab har classic design (Twitter, WhatsApp, Uber, payment) ke end me wo last 15 min bhi add karo — "how would you operate this?" — aur ye 9-topic brain use karke pura answer do. Requirements → design → secure → operate. Tum ready ho. 🚀

All the best, Farzi Philosopher! 💪