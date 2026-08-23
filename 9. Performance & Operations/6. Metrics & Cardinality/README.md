# 06 — Metrics & Cardinality

> **Is lesson me:** Metric types (counter/gauge/histogram/summary), aur cardinality explosion — the #1 real-world observability incident. Kyu histograms > summaries (percentile aggregation — lesson 02 se connect), aur kya metric labels me kabhi nahi daalna.

Prerequisites: `05-Observability-Metrics-Logs-Traces.md`, `02-Percentiles-And-Tail-Latency.md`

---

## 1. Metric types ⭐

| Type | Meaning | Example | Gotcha |
|---|---|---|---|
| **Counter** | Monotonically increasing | `requests_total` | Hamesha RATE graph karo, raw value nahi; restart pe reset handle |
| **Gauge** | Point-in-time value (up/down) | `queue_depth`, `memory_bytes` | Sampled — scrapes ke beech spikes miss |
| **Histogram** | Bucketed distribution | `request_duration_seconds` | Buckets definition pe fixed; bad buckets = no useful p99 |
| **Summary** | Client-computed quantiles | p99 per instance | **Not aggregatable across instances** — isliye histograms win |

> ⚠️ **Prefer histograms over summaries.** Tum per-instance p99s ko fleet p99 me merge nahi kar sakte (lesson 02 — you can't average percentiles). **Histograms merge; quantiles don't.** Ye do lessons directly connected hai — histogram = mergeable structure jo lesson 02 me chahiye tha.

**Counter gotcha explained:** counter hamesha badhta hai (total requests ever). Raw value useless (bas badhta dikhega). Tum uska **rate** graph karte ho (`rate(requests_total[5m])` = requests/sec). Restart pe counter 0 ho jaata — monitoring system ise handle karta (reset detection).

---

## 2. Cardinality explosion ⚠️⭐ (the #1 real incident)

Har unique combination of label values = ek alag time series.
```
http_requests_total{method, status, endpoint}
  5 methods × 10 statuses × 50 endpoints = 2,500 series          ← fine

...ab user_id add karo:
  2,500 × 1,000,000 users = 2,500,000,000 series                 ← metrics
                                                                    system DEAD
```
Ye **sabse common tarika hai jisse teams apna hi monitoring gira dete hai** — aur exactly tab jab unhe uski zarurat hoti hai (during an incident).

**Metric label me KABHI ye nahi:**
```
❌ user ID, request ID, trace ID, email, session ID
❌ full URL path with IDs (/orders/12345)
❌ raw error message, timestamp
```

**High-cardinality data kahan belong karta:** **logs aur traces** (wo iske liye indexed hai; metrics nahi).
```
❌ metric label: endpoint="/orders/12345"
✅ metric label: endpoint="/orders/:id"        ← normalize the template
✅ trace attribute / log field: order_id=12345  ← high cardinality yahan
```

> **Recap (lesson 05):** metrics = low cardinality (the hard constraint); logs/traces = high cardinality fine. Ye woh division hai — per-request identifiers (user/order/trace ID) traces/logs me, metric labels normalized templates + low-cardinality dimensions (region, status class).

**Say this:** *"Main metric labels low-cardinality rakhunga — normalized route templates, status class, region — aur per-request identifiers ko traces/logs me push karunga. User ID ko label me daalke cardinality explosion sabse common tarika hai jisse teams apna monitoring girate hai, thik us waqt jab unhe chahiye."*

**Frontend analogy:** ye analytics events me high-cardinality properties add karne jaisa hai — agar tum har event pe unique user_id ko ek "dimension" bana do, tumhara analytics tool choke kar jaata. Aggregate dimensions (country, device type) rakhо; unique IDs event properties me (queryable, not grouped).

---

## 3. The most important cache metric (interview favorite)

> **Recap (Topic 3):** cache metrics — sabse important **hit ratio** (directly batata cache apni cost earn kar raha ya nahi).
```
Hit ratio falling + high eviction rate → cache too small
Hit ratio falling + low evictions      → access pattern changed / TTLs too short
```
Paired signals se root cause. (Ye interview me aksar poocha jaata — "most important cache metric?" → hit ratio + eviction rate.)

---

## 4. Instrumentation strategy (practical)

```
1. RED metrics per endpoint (rate/errors/duration) — histograms for duration
2. USE per resource (utilization/saturation/errors) — gauges for saturation
3. Normalize labels — /orders/:id not /orders/12345
4. Low-cardinality dimensions only — region, status_class, endpoint_template
5. High cardinality → logs/traces (with trace_id linking)
6. Business metrics — orders/min, payment success (lesson 05)
```

**Interview line:** *"Metrics ke liye: duration histograms (mergeable — fleet p99 nikaal saku), counters for rates, gauges for saturation. Labels strictly low-cardinality — normalized route templates, status class, region — kabhi user/request/trace IDs (cardinality explosion monitoring gira deta). High-cardinality identifiers traces/logs me jaate hai, trace_id se linked. Cache ke liye hit ratio + eviction rate (paired root cause)."*

---

## Quick Self-Check
1. Counter vs gauge vs histogram vs summary — char types?
2. Counter ko raw kyu nahi graph karte (rate)?
3. Histogram vs summary — kaunsa mergeable, aur ye lesson 02 se kaise connect?
4. Cardinality explosion kya hai — user_id label ka example?
5. Metric label me kya kabhi nahi (3 cheezein)? High-cardinality kahan (logs/traces)?
6. `/orders/12345` vs `/orders/:id` — kaunsa label me?
7. Most important cache metric + iska pair (hit ratio + eviction)?

---
**Next:** `07-Logging-Tracing-And-Alerting.md` — logs, traces, aur good alerts.