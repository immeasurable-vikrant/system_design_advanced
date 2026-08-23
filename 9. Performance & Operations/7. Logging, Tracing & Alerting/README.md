# 07 — Logging, Tracing & Alerting

> **Is lesson me:** Structured logging (aur uske hidden costs), distributed tracing (spans, context propagation, sampling), aur alerting done right — symptoms not causes, burn-rate alerting, aur alert fatigue. Ye observability ko actionable banata hai.

Prerequisites: `05-Observability-Metrics-Logs-Traces.md`, `06-Metrics-And-Cardinality.md`

---

## 1. Structured logging ⭐ — always

```
❌ log.info("User " + userId + " failed payment of " + amt + " reason " + r)
   → unqueryable; 3am pe regex likhoge

✅ log.info("payment_failed", {user_id: userId, amount_cents: amt,
                               reason: r, trace_id: traceId})
   → queryable: reason:insufficient_funds AND amount_cents:>10000
```
Structured (JSON/key-value) logs **queryable** hai. String-concatenated logs sirf grep-able (painful at scale).

**Levels, used consistently:**
```
ERROR → kuch fail hua jo eventually human chahiye   (alert on RATE, not per-event)
WARN  → unexpected but handled (retry succeeded, fallback used)  (trend it, don't alert)
INFO  → notable business events (order placed, user registered)
DEBUG → developer detail  (off/sampled in prod)
```
> ⚠️ **If everything is ERROR, nothing is.** Log-level discipline hi "error rate" ko ek usable signal banati hai.

**The costs people forget:**
```
1. VOLUME dominates the bill. 10K rps × 2KB/log = 20MB/s = ~1.7TB/day.
   Prod me DEBUG logging = five-figure monthly line item.
2. Logging synchronous hai jab tak async na banao. Blocking write to slow disk/pipeline
   HAR request me latency add karta — logging ne real outages kiye hai.
3. PII in logs = compliance problem (Topic 8 lesson 09). Emitter pe redact karo,
   pipeline me nahi (store hone ke baad deletion obligation).
```
**Sampling:** all errors log karo, successes sample (1%), dynamic log levels support karo (ek service/user ke liye DEBUG on without redeploy).

**Frontend analogy:** tumne `console.log("user " + x)` vs structured logging seen hai. Backend at scale: structured JSON + trace_id, taaki tum ek trace se uske logs pe jump kar sako.

---

## 2. Distributed tracing ⭐

```
trace_id: abc123  (ek user request, har hop se propagated)

[gateway ──────────────────────────────────────── 450ms]
  [auth ── 20ms]
  [orders ─────────────────────────────── 400ms]
     [db query ── 30ms]
     [inventory ────── 90ms]
     [payments ──────────────────── 260ms]     ← there it is
        [fraud-check ──────── 200ms]           ← actually, there
```

| Term | Meaning |
|---|---|
| **Trace** | Ek end-to-end request |
| **Span** | Ek unit of work within it (service call, DB query) |
| **Parent span** | Jisne ise cause kiya — tree banata |
| **Trace context** | Headers propagated between services (W3C `traceparent`) |
| **Baggage** | Key-value data poore trace ke saath (tenant ID, experiment group) |

> ⚠️ **Propagation is the whole game.** Agar koi service trace header drop kare, trace wahan toot jaata aur downstream sab unrelated trace jaisa dikhta. **Ek shared HTTP/gRPC client me instrument karo** (ya service mesh se free — Topic 1 lesson 06), per-endpoint nahi. Aur **queues:** trace context message headers me bhi carry karo, warna async work invisible ho jaata (Topic 4).

> **Recap (Topic 6 lesson 01):** microservices ka biggest operational cost = observability. Ek request 5-10 services touch karti hai. Distributed tracing hi wo full path deta hai jo monolith me ek stack trace deta tha.

**Sampling:**
```
HEAD-based: entry point pe decide (e.g. keep 1%). Cheap, but jo slow request
            tum chahte ho wo probably sample nahi hua.
TAIL-based: poora trace buffer karo, finish ke baad decide. Keeps all errors +
            slow traces; needs a collector holding spans in memory.
```
**Say this:** *"Tail-based sampling agar platform support kare — 100% errors + slow traces, 1% boring ones. Head-based 1% cheap hai but statistically most rare failures miss karta jo tum debug kar rahe ho."*

---

## 3. Alerting — symptoms, not causes ⭐

```
❌ "CPU > 80%"          → aksar fine, aur user pain nahi matlab
❌ "a pod restarted"    → wo system self-healing hai
❌ "disk 70% full"      → not urgent unless growing fast

✅ "checkout error rate > 1% for 5 min"    → users failing
✅ "p99 latency > 1s for 10 min"           → users waiting
✅ "queue depth growing for 15 min"        → users will fail soon
✅ "error budget burn rate 14× normal"     → SLO blow in 2 days
```

> **The test for a good alert:** agar fire ho aur kisi ko kuch nahi karna, toh fire nahi hona chahiye tha. Har non-actionable page on-call ko alerts ignore karna sikhata — wo **alert fatigue** hai, aur real incidents aise miss hote hai.

**Page on symptoms (user-visible). Route causes to dashboards/tickets.** Ek symptom alert + ek good dashboard > bees cause alerts (jo sab ek saath fire hote incident me aur signal bury kar dete).

---

## 4. Burn-rate alerting (the SRE approach) ⭐

"Error rate > 1%" ke bajaye, error budget kitni tezi se consume ho raha us pe alert:
```
SLO: 99.9% over 30 days → error budget = 43 minutes

Burn rate 1×  → budget exactly 30 din me use → fine
Burn rate 14× → budget ~2 din me gone       → PAGE now
Burn rate 6×  → budget ~5 din me gone       → ticket, not page
```
**Two windows** (fast one for spikes, slow one for slow bleeds) = high precision + recall — fixed threshold se kahin behtar (jo ya har blip pe page kare ya slow degradation miss kare).

> **Recap (lesson 04):** error budget = 100% − SLO. Burn-rate isse operational banata — velocity govern (budget bacha → ship; exhausted → freeze).

---

## 5. Alerts people forget ⚠️

```
□ ABSENCE of data — metric silent = exporter died; silent system healthy DIKHTA hai!
□ Certificate expiry at 30 days (Topic 1 lesson 03 — classic outage)
□ Consumer lag growing (Topic 4 — earliest signal async falling behind)
□ DLQ depth > 0 (Topic 4 — messages permanently dropped)
□ Replication lag > read-your-writes assumption (Topic 2, 5)
□ Cron/batch job didn't run (nothing failing = nothing alerting)
```

**Say this:** *"Main symptoms pe page karunga (user-visible — error rate, p99, queue depth growing), causes ko dashboards/tickets pe route. Burn-rate multi-window alerts against error budget, CPU pe nahi. Aur absence-of-data, cert expiry, consumer lag, DLQ depth — jo silent breaks hai, un pe bhi."*

---

## Quick Self-Check
1. Structured vs string logs — kyu structured (queryable)?
2. "Everything is ERROR" ka problem? Log levels ka use?
3. Logging ke 3 hidden costs (volume/sync/PII)?
4. Trace vs span vs context propagation? Propagation "whole game" kyu?
5. Async work trace me invisible kyu ho jaata (queue headers)?
6. Head vs tail sampling — kaunsa errors/slow traces keep karta?
7. Alert on symptoms not causes — 2 bad vs 2 good examples? Alert fatigue?
8. Burn-rate alerting kya hai (14× vs 6×)?

---
**Next:** `08-Deployment-Strategies.md` — bina downtime ship karna.