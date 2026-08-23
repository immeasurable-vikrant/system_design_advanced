# 01 — Latency, Throughput & Response Time

> **Is lesson me:** Performance ki precise vocabulary — latency vs response time vs service time (jo log galat interchangeably use karte hai), aur latency vs throughput (jo opposites nahi hai). Ye woh language hai jisse tum non-functional requirements state + defend karte ho. "Fast aur scalable" bolna junior signal hai; "p99 < 200ms at 50K rps" bolna senior signal hai.

Prerequisites: Topic 1 lesson 01 (network basics), Topic 7 lesson 02-03 (NFRs, estimation)

---

## 1. Kyu ye vocabulary matter karti hai

Har system design interview non-functional requirements se khulta hai. Candidate bolta hai "it should be fast and scalable." Strong candidate bolta hai *"p99 read latency under 200ms at 50K rps, 99.9% availability, aur thread pools main Little's Law se size karunga."*

Same design, **completely different signal.** Ye poora module wahi vocabulary hai.

---

## 2. Latency vs Response time vs Service time ⭐

Ye teen interchangeably use hote hai — **aur teeno alag hai:**
```
        ┌──────────── response time (jo user feel karta) ─────────────┐
client →│ network out │ queue wait │ service time │ serialize │ net back │→ client
        │             └──── latency (waiting, not working) ────┘         │
```

| Term | Definition | Kahan dikhta |
|---|---|---|
| **Service time** | Actual processing me laga time | Tumhara app timer, server-side histogram |
| **Latency** | Strictly: waiting time (queue + network). Loosely: response time ka synonym | Load balancer metrics |
| **Response time** | Total client-observed duration | Client-side timing, RUM, synthetic checks |

**Say this:** *"Response time jo user feel karta hai; service time jo mera server log karta hai. Gap = queueing — aur load pe queueing dominate karta hai, isliye p99 CPU ke 100% pahunchne se pehle hi explode ho jaata hai."*

> ⚠️ **The classic gotcha:** tumhara server-side p99 40ms hai but users complain karte hai. Wo 40ms **exclude** karta hai LB queue wait, TLS handshakes, DNS, mobile radio wake-up. **Hamesha clear raho tumne kahan measure kiya.** (Isliye client-side RUM chahiye, sirf server-side nahi — lesson 07.)

**Frontend analogy:** tumne DevTools Network tab me dekha hoga — "Waiting (TTFB)", "Content Download", "Stalled/Queueing". Wo breakdown exactly ye hai — total (response time) me se actual server work (service time) alag hai, baaki waiting (latency) hai.

---

## 3. Latency vs Throughput ⭐

Ye **opposites nahi hai**, aur ek improve karna aksar doosre ki keemat pe aata hai:
```
Latency    = time per operation     (ms/op)   → lower better
Throughput = operations per second  (ops/s)   → higher better
```

| Technique | Throughput | Latency |
|---|---|---|
| **Batching** (batch bhar ne ka wait) | ↑↑ | ↑ (wait pay karte ho) |
| **Pipelining** (overlap work) | ↑ | ~ |
| **Replicas add** | ↑ | ~ (jab tak coordination cost na aaye) |
| **Compression** | ↑ (kam bytes) | ↑ CPU, ↓ transfer — WAN pe net win, LAN pe loss |
| **Caching** | ↑ | ↓↓ (**the rare win-win**) |

**The highway analogy (jo actually kaam karti hai):** latency = tumhari car ko road travel karne me kitna time. Throughput = kitni cars per minute guzarti hai. **Lanes add karna throughput badhata hai bina tumhare trip ko fast kiye.** Speed limit badhana latency kam karta hai. Aur enough traffic pe, latency degrade hoti hai chahe throughput maximum ho — wo **congestion** hai, aur exactly wahi ek saturated service ke saath hota hai (lesson 03 — queueing wall).

> **Key insight:** sirf ek cheez dono help karti hai — **kaam kam karna** (fewer hops, fewer bytes, fewer round trips). Aur **caching** (dono win). Baaki sab trade-off hai.

---

## 4. Throughput vocabulary (precise)

| Term | Precise meaning |
|---|---|
| **QPS / RPS** | Queries/requests per second — ek rate. Latency figure ke bina capacity ke liye meaningless |
| **TPS** | Transactions per second. Ek transaction kai queries ho sakti — units mat mix karo |
| **Concurrency** | Requests in flight simultaneously (= λ × W, lesson 03). **Ye resources exhaust karta hai, rate nahi** |
| **Bandwidth** | Link ki theoretical max capacity |
| **Throughput** | Jo tum actually achieve karte ho |
| **Goodput** | Useful payload delivered (retransmits/headers/overhead ke bina). **Retry storm me throughput high but goodput collapse** — ek great phrase |
| **IOPS** | I/O operations per second — disk metric (4KB random reads conventional unit) |
| **Peak vs Average** | Consumer apps me peak-to-average ~2-5×. Provision for peak (+ headroom), bill on average. **Hamesha batao kaunsa number** |

> ⚠️ **"QPS alone is meaningless."** "50K rps" capacity nahi batata jab tak latency target na ho. "50K rps at p99 < 200ms" = actual capacity. Interview me hamesha rate + latency dono do.

---

## 5. Latency numbers to memorize ⭐

> **Recap (Topic 1 lesson 01):** ye numbers estimation + design decisions drive karte hai:
```
L1 cache:                1 ns       (1×)
Main memory (RAM):       100 ns     (100×)
SSD random read:         16 µs      (16,000×)
Datacenter round trip:   500 µs
HDD seek:                2 ms       (2,000,000×)
India ↔ US East RTT:     ~200 ms    (physics — can't fix)
```

**The 3 ratios that matter in an interview:**
```
1. Memory ~100× faster than SSD; SSD ~100× faster than disk seek
   → isliye CACHING works (Topic 3)
2. Sequential ~100× faster than random on any storage
   → isliye LSM trees, WALs, Kafka APPEND (Topic 2 lesson 10, Topic 4)
3. Cross-continent RTT ~200ms, physics won't fix it
   → isliye CDNs + regional replicas (Topic 1 lesson 07, Topic 5) — speed of light ko beat nahi kar sakte
```

---

## 6. Stating NFRs like a senior (interview application)

```
"Reads: p99 < 200ms at 50K rps.
 Writes: p99 < 500ms at 5K rps.
 Availability: 99.9% reads, 99.99% payment path.
 Consistency: strong for balances, eventual (<2s) for feeds.
 Peak-to-average 3×, so I'll provision 150K rps read capacity with 30% headroom."
```
Ye ek sentence tumhe baaki candidates se alag kar deta hai — precise, quantified, defensible.

**Interview line:** *"Main NFRs precisely state karunga — p99 latency + rate together (rate alone meaningless), success rate, availability per-path, consistency per-component. 'Fast' ke bajaye 'p99 < 200ms at the edge'. Aur clear rahunga main kahan measure kar raha hu — edge/client, not just server-side (jahan queue wait + network miss ho jaata)."*

---

## Quick Self-Check
1. Service time vs latency vs response time — teeno kya, kahan measure?
2. "Server p99 40ms but users complain" — kya miss ho raha (gotcha)?
3. Latency vs throughput — opposites kyu nahi? Highway analogy?
4. Kaunsi 2 cheezein dono (latency + throughput) help karti hai?
5. QPS alone kyu meaningless — kya add karo?
6. Goodput vs throughput — retry storm me kya hota?
7. The 3 memorized ratios (memory/SSD, sequential/random, cross-continent) — kaunse design decisions drive karte?

---
**Next:** `02-Percentiles-And-Tail-Latency.md` — kyu averages jhooth bolte hai.