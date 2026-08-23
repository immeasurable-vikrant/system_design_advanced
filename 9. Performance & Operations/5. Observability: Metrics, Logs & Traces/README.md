# 05 — Observability: Metrics, Logs & Traces

> **Is lesson me:** Monitoring vs observability, the three pillars (metrics/logs/traces), the debug workflow jo unhe tie karta hai, the four golden signals, aur RED vs USE. Ye senior interview ke last 5 min ka answer hai — "how would you know this system is healthy?"

Prerequisites: `04-Availability-And-Reliability-Math.md`, Topic 3 (caching metrics), Topic 4 (queue metrics)

---

## 1. Monitoring vs Observability

| | Monitoring | Observability |
|---|---|---|
| Answers | Known questions ("is CPU high?") | Unknown questions ("why are Android users in Mumbai slow since 3pm?") |
| Built from | Predefined dashboards + alerts | High-dimensional data you can slice arbitrarily |
| Failure mode | Sirf wo dikhta jo tumne measure socha | Cost + cardinality |

**Say this:** *"Monitoring batata hai ki kuch wrong hai; observability batata hai KYU, bina naya code ship kiye. Practical test: kya main ek aisa question answer kar sakta hu jo maine anticipate nahi kiya? Agar debug karne ko metric add karke redeploy karna pade, toh mere paas monitoring hai, observability nahi."*

---

## 2. The three pillars ⭐

| | Metrics | Logs | Traces |
|---|---|---|---|
| What | Numeric aggregates over time | Discrete timestamped events | Ek request ka path across services |
| Answers | "Is it broken? How much?" | "What exactly happened here?" | "Where is the time going?" |
| Cost | Cheap (pre-aggregated) | Expensive at volume | Expensive (sampled) |
| Cardinality | **Low — the hard constraint** | High is fine | High is fine |
| Retention | Long (months-years, downsampled) | Short (days-weeks) | Short, sampled |
| Tools | Prometheus, CloudWatch, Datadog | ELK, Loki, CloudWatch Logs | Jaeger, Tempo, X-Ray, Honeycomb |
| Use for | Alert + trend | Debug a specific failure | Find the slow hop |

---

## 3. The debug workflow ⭐ (ties them together)

Ye "how would you debug a latency regression?" ka answer hai — tools list karne se zyada valuable:
```
1. METRIC alert fires: "checkout p99 > 1s"
     ↓
2. DASHBOARD: which service? → payments
     ↓
3. TRACE: where in the request? → 800ms in the fraud-check call
     ↓
4. LOGS for that trace ID → "fraud model cache miss, falling back to sync scoring"
     ↑ root cause, in ~90 seconds
```
**Metrics to DETECT, traces to LOCALIZE, logs to EXPLAIN.** Ye ek workflow hai, alternatives nahi. Is sequence ko naam dena tools list karne se worth zyada hai.

**Frontend analogy:** ye bilkul frontend debugging jaisa hai — performance graph spike dekha (metric) → Network/Performance tab me kaunsa resource slow (trace-ish) → console/error logs me kyu (logs). Same mental model, distributed scale pe.

---

## 4. The four golden signals ⭐ (Google SRE)

Har service ke liye ye 4 watch karo:
```
LATENCY:     How slow?   → p50/p99 histogram, split successful vs failed
TRAFFIC:     How much?   → rps, qps, concurrent connections
ERRORS:      How often wrong? → 5xx rate, exception rate, + silent failures
SATURATION:  How full?   → CPU, memory, queue depth, connection pool utilization
```

> ⚠️ **Split latency by success and failure.** Ek fast-failing 500 latency improvement jaisa dikhta hai agar sab ek saath bucket karo — errors 2ms me return hoke tumhara p99 pull down karenge jab system burn ho raha hai. **Always separate.**

---

## 5. RED vs USE ⭐ (two lenses)

| | RED | USE |
|---|---|---|
| Applies to | **Services** (request-driven) | **Resources** (CPU, disk, pool, queue) |
| Measures | **R**ate, **E**rrors, **D**uration | **U**tilization, **S**aturation, **E**rrors |
| Perspective | User's | Machine's |

**Say this:** *"RED har service ke liye — rate, errors, duration — kyunki wo user experience karta hai. USE har resource ke liye, kyunki saturation ek leading indicator hai — queue depth aur pool utilization latency se PEHLE rise karte hai, jo unhe autoscaling aur early paging ke liye useful banata hai (Little's Law + queueing wall — lessons 03)."*

> **Recap (lesson 03):** saturation (queue depth, pool utilization) leading indicator hai — latency ke bad dikhta hai, but saturation pehle. Isliye scale/alert saturation pe, CPU pe nahi.

---

## 6. What to instrument (walk the request path)

"How would you monitor this?" pe, request path walk karo (Topics 1-4 se connect):

| Layer | Watch (which prior topic) |
|---|---|
| **Edge / CDN** (Topic 1) | Cache hit ratio, origin request rate, 4xx/5xx split, TLS errors |
| **Load balancer** (Topic 1) | Per-target health, surge queue length, rejected connections |
| **Service** (Topic 6) | RED per endpoint, saturation (pools), GC pause time |
| **Cache** (Topic 3) | Hit ratio, eviction rate, memory used/max, hot-key skew |
| **Database** (Topic 2) | p99 query time, slow query log, connections vs max, replication lag, lock waits |
| **Queue** (Topic 4) | Consumer lag, oldest message age, DLQ depth, processing vs arrival rate |
| **Business** | Orders/min, signups/min, payment success — fastest detector of subtle breaks |

> ⚠️ **Business metrics catch what technical metrics miss.** Ek deploy jo checkout button invisible kar de → perfect technical dashboard (200s everywhere, low latency) → but revenue zero. "Orders/min vs same time last week" ise minutes me pakad leta. Ye mention karna strong senior signal.

**Interview line:** *"Health ke liye RED per service + USE per resource, SLO user-facing indicator pe. Debug ke liye: metrics detect → traces localize → logs explain. Request path walk karke har layer instrument — edge (cache hit ratio), LB (health), service (RED), cache (hit ratio/evictions), DB (p99 query, replication lag), queue (consumer lag, DLQ). Plus business metrics (orders/min) subtle breaks ke liye jo technical metrics miss karte hai."*

---

## Quick Self-Check
1. Monitoring vs observability — known vs unknown questions? Practical test?
2. Three pillars — metrics/logs/traces, har ek kya answer karta?
3. The debug workflow — metrics/traces/logs ka role (detect/localize/explain)?
4. Four golden signals? Latency ko success/failure se split kyu?
5. RED vs USE — kaunsa services, kaunsa resources? Saturation leading kyu?
6. Request path pe har layer kya instrument (cache/DB/queue)?
7. Business metrics kyu critical (invisible checkout button example)?

---
**Next:** `06-Metrics-And-Cardinality.md` — metrics deep + the #1 real-world incident.