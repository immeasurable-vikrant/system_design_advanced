# 10 — Life of a Request (Capstone Playbook)

> **Is lesson me:** Poore module ka synthesis — ek request ka **poora safar** jab tum URL type karke Enter dabate ho (DNS se database tak), sab components ek saath. Phir ek "scale a system from scratch" playbook, aur ye Topic 1 kaise Topics 2 & 3 me flow hota hai. Ye tumhara master reference hai.

Prerequisites: **saare pichle lessons (01-09).**

---

## 1. "URL type karke Enter dabaya" — the full journey ⭐

Ye THE classic interview question hai. Ab tumhare paas poora answer hai. Chalo step-by-step (har step ka lesson bracket me):

```
User types "example.com/feed" and hits Enter:

┌─ STEP 1: DNS RESOLUTION (lesson 03) ────────────────────────────┐
│  browser cache → OS cache → resolver → authoritative NS         │
│  "example.com" → 93.184.216.34 (TTL honored)                    │
│  CDN me hai toh ANYCAST IP milta hai → nearest edge route       │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─ STEP 2: TCP + TLS HANDSHAKE (lesson 01, 03) ───────────────────┐
│  TCP 3-way handshake (1 RTT) + TLS handshake (1-2 RTT)          │
│  CDN edge pe terminate hota hai → RTT chhota (user ke paas)      │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─ STEP 3: CDN EDGE (lesson 07) ──────────────────────────────────┐
│  Static assets (CSS/JS/images)? → edge cache se serve (~10ms)   │
│  Dynamic /feed? → origin tak jaana padega (edge pass-through)    │
└─────────────────────────────────────────────────────────────────┘
                              ▼ (dynamic request, warm HTTP/2 conn)
┌─ STEP 4: LOAD BALANCER / API GATEWAY (lesson 05, 08) ───────────┐
│  L7 LB / gateway: path dekho → /feed → Feed Service              │
│  Auth (JWT validate), rate limit check (429 if over — Topic 3)  │
│  Healthy instance choose (least connections)                    │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─ STEP 5: APP SERVER (stateless — lesson 04) ────────────────────┐
│  Feed Service instance request handle kare                      │
│  Session? → Redis se (stateless server)                         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─ STEP 6: CACHE CHECK (Topic 3) ─────────────────────────────────┐
│  Feed cache me hai (Redis)? → HIT → return (fast path) ✅        │
│  MISS → database tak jaana padega                               │
└─────────────────────────────────────────────────────────────────┘
                              ▼ (cache miss)
┌─ STEP 7: CONNECTION POOL → DATABASE (lesson 09, Topic 2) ───────┐
│  Pool se DB connection borrow (no handshake — reuse)            │
│  Query (indexed — Topic 2 lesson 06) → rows                     │
│  Read replica se (read-heavy — Topic 2 lesson 12)               │
│  Connection wapas pool me                                       │
└─────────────────────────────────────────────────────────────────┘
                              ▼
        Response → cache me daalo → gateway → CDN → user
        (aur agli same request STEP 6 pe hi ruk jaayegi — cached)
```

**Ye poora flow bolna = you understand core infrastructure.** Har box ek lesson hai, aur teeno topics (infra + DB + caching) yahan milte hai.

---

## 2. Pre-first-byte cost (kyu optimizations matter)

Us journey me, **actual kaam se pehle** kitna time gaya:
```
DNS lookup:      ~20-100ms (cache miss pe)
TCP handshake:   1 RTT
TLS handshake:   1-2 RTT
─────────────────────────────
= 2-3 RTT + DNS before your request even arrives
```
Isliye ye sab exist karta hai: **connection reuse** (lesson 01), **CDN edge termination** (lesson 03/07), **HTTP/3** (lesson 02), **caching** (Topic 3). Har ek in RTTs ko kam karta hai.

---

## 3. Playbook — "scale a system from 100 to 10M users"

Ek unified progression (Topic 1 + 2 + 3 milke). Har step **ek specific bottleneck** pe trigger, preemptively nahi (lesson 04):

```
STAGE 1 (100 users):   Single server (app + DB same machine)
STAGE 2 (1K):          App aur DB alag machines
STAGE 3 (10K):         + Load Balancer + multiple app servers (stateless)
                         [lesson 04, 05]
STAGE 4 (50K):         + Redis cache (90% reads absorb) [Topic 3]
                       + CDN for static/media [lesson 07]
STAGE 5 (200K):        + Read replicas (read scaling) [Topic 2 lesson 12]
                       + Connection pooling tuned [lesson 09]
STAGE 6 (1M):          + API Gateway (microservices) [lesson 08]
                       + Rate limiting [Topic 3]
                       + Message queues (async work)
STAGE 7 (10M):         + Database sharding (write scaling) [Topic 2 lesson 13]
                       + Multi-region (geo latency) [lesson 03 anycast]
                       + Service mesh (internal mTLS) [lesson 06]
```

**Golden principle:** *har step ek bottleneck ka jawaab hai. Measure first, phir sahi cheez add karo (Topic 3 lesson 01).*

---

## 4. The component cheat-sheet (kaun kya karta hai)

Interview me ye distinctions clear hone chahiye:

| Component | Ek line role | Layer |
|---|---|---|
| **DNS** | Naam → IP | — (lesson 03) |
| **CDN** | Static content user ke paas cache | Edge (lesson 07) |
| **Load Balancer** | Requests ko instances me distribute | L4/L7 (lesson 05) |
| **Reverse Proxy** | LB + SSL + cache + security (superset) | L7 (lesson 06) |
| **API Gateway** | Routing + auth + rate-limit (smart entry) | L7 (lesson 08) |
| **Service Mesh** | Service↔service mTLS/retries (internal) | L7 sidecar (lesson 06) |
| **Cache (Redis)** | Hot data, DB se pehle | (Topic 3) |
| **Connection Pool** | Reusable DB connections | (lesson 09) |
| **Database** | Source of truth | (Topic 2) |

**Common confusions resolved:**
- **Reverse proxy vs LB:** LB ek function; reverse proxy uska superset (lesson 06).
- **Gateway vs LB:** gateway "kaunsi service" (+ auth/rate-limit); LB "us service ka kaunsa instance" (lesson 08).
- **Gateway vs service mesh:** gateway north-south (external); mesh east-west (internal) (lesson 08).
- **CDN vs cache vs replica:** CDN edge bytes; Redis app-level hot data; replica DB-level queries (lesson 07).

---

## 5. Interview one-liners (ready to say)

- **"Request kaise reach karti hai?"** → *"DNS anycast CDN IP pe resolve; edge TLS 1.3 terminate karke static cache se serve karta hai. Dynamic requests warm HTTP/2 connection pe origin ALB tak jaate hai, jo L7 routing karta hai path pe, phir keep-alive ke saath service tak. Internal service-to-service gRPC over HTTP/2 with mTLS."*
- **"Mobile app web se slow kyu?"** → *"Higher RTT + packet loss cellular pe, plus radio wake-up. Connection aggressively reuse karo, HTTP/3 pe jao (per-stream recovery + 0-RTT), requests batch karo, CDN edge user ke paas rakho."*
- **"1M connected clients ko update kaise push?"** → *"SSE agar server→client only. Connections L4 LB ke peeche fleet me spread, long idle timeouts. Har server apne users ke Redis/Kafka channels subscribe kare, taaki publish sahi server tak pahunche bina sticky routing ke. ~30KB/connection budget → 1M = ~30GB → instance count decide."*

---

## 6. Full module concept map

```
NETWORK FOUNDATION   layers/transport (01) → HTTP/real-time (02) → DNS/TLS (03)
SCALE & DISTRIBUTE   scalability (04) → load balancing (05)
INTERMEDIARIES       proxies (06) → CDN (07) → API gateway (08)
LAST MILE            connection pooling (09)
SYNTHESIS            life of a request (10)
```

**Aur teeno topics milke:**
```
Topic 1 (Core Infra)  → request ka safar: network, LB, proxy, CDN, gateway, pool
Topic 2 (Database)    → data: RDBMS, indexing, transactions, replication, sharding
Topic 3 (Caching/Perf)→ speed: caching, rate limiting, bloom filters
```
Ek request Topic 1 ke infra se guzarti hai → Topic 3 ke cache se hoke → Topic 2 ke database tak. **Teeno ek continuous system hai.**

---

## Quick Self-Check (capstone)
1. "URL type karke Enter" — 7 steps bina dekhe bol sakte ho (DNS se DB tak)?
2. Pre-first-byte me kitne RTT lagte hai, aur kaunse optimizations use kam karte hai?
3. Scale 100→10M ka progression — har stage ka trigger (bottleneck)?
4. Reverse proxy vs LB vs gateway vs service mesh — chaaron ko ek-ek line me distinguish karo.
5. CDN vs Redis cache vs read replica — teeno kaunse level pe cache karte hai?
6. Ek request Topic 1 → Topic 3 → Topic 2 me kaise flow karti hai?

---

**🎉 Topic 1 complete!** Ab tum ek request ka poora infrastructure safar samajhte ho — DNS/TLS se leke load balancers, proxies, CDN, gateways, aur connection pooling tak. Ye woh foundation hai jispe Database (Topic 2) aur Caching & Performance (Topic 3) khade hai.

**Teeno topics ab complete hai:**
- **Topic 1 — Core Infrastructure** (ye) → request ka safar + scaling
- **Topic 2 — Database & Storage** → data layer
- **Topic 3 — Caching & Performance** → speed + protection

Ab tum ek poora system design kar sakte ho — client se leke database tak, har layer par sahi decisions ke saath. Agla natural step: in teeno ko actual **HLD problems** (URL Shortener, Chat System, News Feed, Rate Limiter, Instagram) pe apply karna, jahan ye saare pieces ek complete architecture me fit hote hai.