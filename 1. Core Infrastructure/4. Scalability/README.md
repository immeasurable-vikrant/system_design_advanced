# 04 — Scalability

> **Is lesson me:** Scalability kya hai, vertical vs horizontal scaling (scale up vs scale out), **stateless services** (horizontal scaling ki neenv), auto-scaling, aur database ko scale karne ki progression. Ye woh "load badhne pe kya karein" ka framework hai jo baaki poore module (LB, cache, CDN) ko motivate karta hai.

Prerequisites: `01-Networking-I` (basics). Helpful: Topic 2 (Database) aur Topic 3 (Caching) — but yahan self-contained.

---

## 1. Scalability kya hai

**Scalability** = ek system ki capability ki wo **badhte hue load** ko handle kar sake **resources add karke**, bina architecture ko fundamentally badle. Ek scalable system apni performance (latency, throughput) maintain karta hai jaise demand badhti hai.

**Analogy:** Ek pizza restaurant. **Vertical scaling** = ek bada oven khareedna (ek oven zyada kaam kare). **Horizontal scaling** = aur branches kholna (kai ovens parallel kaam karein). Bada oven eventually max out ho jaata hai. Aur branches almost infinitely scale karti hai — but tumhe customers ko sahi branch route karna, aur menu consistent rakhna padta hai.

---

## 2. Vertical vs Horizontal Scaling (the core fork)

```
                 Load badh gaya
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   VERTICAL (Scale Up)       HORIZONTAL (Scale Out)
   Bada machine               Aur machines
   (more CPU/RAM/SSD)         (add instances)
   Limit: hardware max        Limit: almost none
   Single point of failure    LB + stateless design chahiye
```

| Aspect | Vertical (Scale Up) | Horizontal (Scale Out) |
|---|---|---|
| Kaise | Bada hardware (CPU/RAM/disk) | Aur machines |
| Limit | Hardware max (~96 cores, ~24TB RAM) | Practically unlimited |
| Downtime | Usually restart chahiye | Zero (rolling) |
| Cost curve | Exponential (2x power ≠ 2x cost) | Linear (2x machines ≈ 2x cost) |
| Complexity | Low (same code, bada box) | High (distributed systems challenges) |
| Failure | Single point of failure | Redundant by design |
| Best for | Databases, early-stage | Stateless services, web/app tier |

**Vertical** — simple (same code, bada box), but ek limit + single point of failure. **Horizontal** — practically unlimited + redundant, but distributed-systems complexity (LB chahiye, state manage karo).

> **Frontend analogy:** Vertical = tumhare laptop ko upgrade karna (zyada RAM). Horizontal = kaam ko kai laptops me baant dena. Pehla ek point tak, doosra hamesha.

---

## 3. Stateless Services — horizontal scaling ki neenv ⭐

Horizontal scaling ki **buniyaad** hai statelessness — koi bhi instance koi bhi request handle kar sakta hai, kyunki **koi request-specific state server pe nahi rehti.**

**Problem (stateful):** User Server A pe login karta hai (session Server A ki memory me). Agli request Server B pe jaati hai → session nahi milta → user logged out! Ab tum kisi bhi server pe route nahi kar sakte.

**Solution (stateless):** state ko server se bahar nikaalo:

| Stateful (scale karna mushkil) | Stateless (scale karna easy) |
|---|---|
| Session server ki memory me | Session Redis me, ya JWT me encode |
| File uploads locally stored | Files S3 / object storage me (Topic 2 lesson 15) |
| In-process cache per server | Shared cache (Redis cluster — Topic 3) |
| WebSocket ek server se bound | WebSocket + pub-sub backplane (lesson 02) |

Jab **koi request kisi specific server pe depend nahi karti**, tab tum load balancer ke peeche freely horizontal scale kar sakte ho.

> **JWT kya hai (quick):** JSON Web Token — ek signed token jisme user ki session info **khud encode** hoti hai. Server ko store karne ki zarurat nahi — token khud proof hai. Isliye stateless. (Trade-off: revoke karna mushkil, token size.) Alternative: session Redis me rakho (shared, revocable).

**Interview line:** *"Service ko stateless banane ke liye: sessions Redis/JWT me, files S3 me, cache shared Redis me, WebSockets pub-sub backplane pe. Jab koi request kisi ek server pe hit hone pe depend na kare, main LB ke peeche horizontal scale kar sakta hu."*

---

## 4. Auto-Scaling

**Auto-scaling** = instances ki number ko **automatically** adjust karna, metrics ke basis pe:

| Metric | Scale-out (add) | Scale-in (remove) |
|---|---|---|
| CPU utilization | > 70% for 3 min | < 30% for 10 min |
| Request count | > 1000 RPS/instance | < 200 RPS/instance |
| Queue depth | > 1000 pending | < 100 pending |
| Response time (P95) | > 500ms | < 100ms |

**Key parameters:**
- **Cooldown period** — scaling events ke beech 5-10 min wait, taaki "thrashing" (baar-baar up-down) na ho
- **Min/max instances** — bounds set karo (min=2 redundancy ke liye, max=50 cost ke liye)
- **Predictive scaling** — known spikes se pehle pre-scale (Black Friday, sale)

> **P95 kya hai:** "95th percentile" — 95% requests isse fast hai. P95=500ms matlab 95% users ≤500ms, top 5% slowest 500ms+. Average se better metric hai (average outliers chhupa deta hai). P99 aur bhi strict.

---

## 5. Database ko scale karna (the hardest part)

Database usually **sabse mushkil** component hai scale karne me. Progression (har step ek specific bottleneck pe trigger hota hai, pehle se nahi):

| Stage | Handles | Technique |
|---|---|---|
| Single DB | < 1K QPS | Ek powerful machine |
| Vertical scale | 1-5K QPS | Bada machine (more RAM, faster SSD) |
| **Read replicas** | 10-50K read QPS | Reads replicas pe, writes primary pe (Topic 2 lesson 12) |
| **Cache layer** | 100K+ read QPS | Redis/Memcached 90%+ reads absorb kare (Topic 3) |
| **Sharding** | 50K+ write QPS | Data multiple DB instances me split (Topic 2 lesson 13) |
| Multi-region | Global scale | Regions me replicate, low latency |

> **QPS = Queries Per Second.** Ye poori progression tumne Topic 2 (replication, sharding) aur Topic 3 (caching) me detail me dekhi — yahan ye **ek unified ladder** hai. Key insight: **har step ek bottleneck pe trigger hota hai, preemptively nahi.**

---

## 6. Scaling strategies comparison + mistakes

| Strategy | Complexity | Cost | Best for |
|---|---|---|---|
| Vertical scaling | Low | High/unit | Quick fix, databases |
| Horizontal scaling | Medium | Linear | Stateless app tier |
| Caching | Low-Med | Low | Read-heavy (Topic 3) |
| Read replicas | Medium | Medium | Read-heavy DBs |
| Sharding | High | Medium | Write-heavy DBs |
| CDN | Low | Low | Static content, global (lesson 07) |
| Async processing | Medium | Low | Spiky workloads |

> ⚠️ **Common mistakes:**
> - **Premature sharding** — read replicas + caching try kiye bina shard karna (sharding complex hai — last resort).
> - **Stateful services ko scale-out** karna state nikaale bina (pehle stateless banao).
> - **Auto-scaling cooldown na set karna** → thrashing.
> - **App tier scale karna jabki DB bottleneck hai** — galat jagah optimize (Topic 3 lesson 01: "optimize the bottleneck").

**Interview: "100 users se 10M users tak kaise scale?"** — Progressive: (1) single server + DB → (2) app aur DB alag machines → (3) LB + multiple app servers → (4) Redis cache → (5) read replicas → (6) CDN → (7) shard DB (jab writes bottleneck) → (8) message queues async → (9) multi-region. **Har step ek specific bottleneck pe, preemptively nahi.**

---

## Quick Self-Check
1. Scalability ki definition — pizza analogy se vertical vs horizontal?
2. Vertical aur horizontal scaling — 3 differences (limit, cost, failure)?
3. Stateless service kyu horizontal scaling ki neenv hai? Stateful ka session problem?
4. Service ko stateless kaise banate ho (4 cheezein kaha move karo)?
5. Auto-scaling me cooldown period aur thrashing kya hai?
6. Database scaling progression ke 6 stages (single DB se multi-region)?
7. "Premature sharding" mistake kya hai — pehle kya try karna chahiye?

---
**Next:** `05-Load-Balancing.md` — multiple servers me traffic distribute karna.