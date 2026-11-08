# 04 — Scalability

> **Is lesson me:** Scalability kya hai, vertical vs horizontal scaling (scale up vs scale out), **instance** kya hota hai, **stateful vs stateless** (history ke saath — sticky sessions se serverless tak), **REST** aur uska statelessness se rishta, auto-scaling (cooldown, thrashing, min/max, predictive), **P50/P95/P99** latency, **QPS** asal me kya hai, concurrency vs parallelism, aur database ko scale karne ki progression. Ye woh "load badhne pe kya karein" ka framework hai jo baaki poore module (LB, cache, CDN) ko motivate karta hai.

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

### "Instance" kya hota hai?
**Instance = tumhare app ki ek chalti hui copy.** Same code, alag-alag copies, har ek apna kaam kar rahi. Ye copy ho sakti hai:
- ek **VM** (AWS EC2 machine),
- ek **container** (Docker / Kubernetes pod),
- ya ek machine pe ek **process** (jaise `uvicorn` ke 4 workers).

Toh "instance" ≈ "server", bas thoda loose. Ek badi machine pe kai instances chal sakte hai, aur 10 instances ka matlab hamesha 10 physical machines nahi hota. **Horizontal scaling = instances ki ginti badhana.**

```
            Load Balancer
         ┌──────┼──────┐
         ▼      ▼      ▼
     instance instance instance     ← same FastAPI code, 3 copies
         └──────┼──────┘
         Redis / DB / S3 (shared)
```

---

## 3. Stateful vs Stateless — horizontal scaling ki neenv ⭐

**State** = woh data jo ek request se agli request tak yaad rakhna padta hai (logged-in user kaun hai, cart me kya hai, upload ki hui file).

- **Stateful service** = ye state **instance ki apni memory/disk** me rehti hai. Agli request usi instance pe aani chahiye.
- **Stateless service** = instance kuch yaad nahi rakhta. State **bahar** (Redis, DB, S3, ya client ke token me) rehti hai, toh **koi bhi instance koi bhi request** handle kar sakta hai.

> **Stateless ka matlab "state nahi hai" nahi hai.** State hai, bas app server ke **bahar shift** ho gayi hai. App tier stateless; Redis/DB stateful (aur unko scale karna isliye mushkil — section 7).

### Example — shopping cart

**Stateful (purana tarika):**
```
Request 1: "cart me iPhone daalo"  → LB → Instance A  (cart A ki RAM me)
Request 2: "mera cart dikhao"      → LB → Instance B  → "cart khaali hai" ❌
Instance A crash → saare users ke carts gayab ❌
```

**Stateless (aaj ka tarika):**
```
Request 1: cookie: session=abc → LB → Instance A → Redis: cart:abc = [iPhone]
Request 2: cookie: session=abc → LB → Instance B → Redis se cart:abc padha ✅
Instance A crash → koi farak nahi, data Redis me hai ✅
```

### History — kaise yahan tak pahunche

| Daur | Kaise chalta tha | Problem / kyu badla |
|---|---|---|
| **2000s — stateful monoliths** | PHP/Java apps, session server ki RAM me (`HttpSession`), uploads local disk pe. Scale ke liye **sticky sessions**: LB ek user ko hamesha same server pe bhejta | Server mara = users logged out; ek server pe load atak jaata; naya server add karne se purane users ka load nahi baant'ta |
| **~2010 — cloud + auto-scaling** | Servers ab "cattle, not pets" — kabhi bhi ban/mar sakte hai. Isliye state bahar: sessions Redis/DB me, files S3 me. "12-Factor App" guidelines ne isko standard bana diya | Ab koi bhi instance koi bhi request le sakta hai → freely scale |
| **~2014+ — serverless (AWS Lambda etc.)** | Tum server manage hi nahi karte; har request pe cloud ek function instance chalata hai, jo kabhi bhi band ho sakta hai | Yahan statelessness **forced** hai — function ki memory pe bharosa hi nahi kar sakte |

> **Stateless ≠ serverless.** Stateless = ek **design property** (instance kuch yaad nahi rakhta). Serverless = ek **deployment model** (server tum manage nahi karte). Serverless code ko stateless hona hi padta hai, but tumhara normal FastAPI on EC2 bhi fully stateless ho sakta hai.

### Kya kahan move karte hai

| Stateful (scale karna mushkil) | Stateless (scale karna easy) |
|---|---|
| Session server ki memory me | Session Redis me, ya JWT me encode |
| File uploads locally stored | Files S3 / object storage me (Topic 2 lesson 15) |
| In-process cache per server | Shared cache (Redis cluster — Topic 3) |
| WebSocket ek server se bound | WebSocket + pub-sub backplane (lesson 02) |

> **JWT kya hai (quick):** JSON Web Token — ek signed token jisme user ki session info **khud encode** hoti hai. Server ko store karne ki zarurat nahi — token khud proof hai. Isliye stateless. (Trade-off: revoke karna mushkil, token size.) Alternative: session Redis me rakho (shared, revocable).

**Interview line:** *"Service ko stateless banane ke liye: sessions Redis/JWT me, files S3 me, cache shared Redis me, WebSockets pub-sub backplane pe. Jab koi request kisi ek server pe hit hone pe depend na kare, main LB ke peeche horizontal scale kar sakta hu."*

---

## 4. REST — aur kya statelessness = REST?

**REST (REpresentational State Transfer)** = HTTP APIs design karne ki ek **style** (Roy Fielding, 2000). Protocol nahi — HTTP hi use hota hai, bas kuch rules ke saath:

| Rule | Matlab | Example |
|---|---|---|
| **Resources + URLs** | Har "cheez" ka ek URL, **noun** (verb nahi) | `/users/123`, `/orders/55` (not `/getUser?id=123`) |
| **HTTP methods = action** | Kya karna hai, woh method batata hai | `GET` padho, `POST` banao, `PUT/PATCH` update, `DELETE` hatao |
| **Representation** | Resource JSON (ya kisi format) me aata-jaata hai | `{ "id": 123, "name": "Bob" }` |
| **Status codes** | Result HTTP code se batao | `201 Created`, `404`, `409 Conflict` |
| **Stateless** | Har request **apne aap me complete** — saari zaroori info (auth token, params) usi me | Har call pe `Authorization: Bearer ...` |
| **Cacheable** | Response bata sake ki cache ho sakta hai ya nahi | `Cache-Control: max-age=60` |

**Toh kya statelessness = REST?** Nahi — **statelessness REST ka ek rule hai**, poora REST nahi. Do related cheezein:
- **REST statelessness** = server **pichli request yaad nahi rakhta**; har request khud bata deti hai "main kaun hu, kya chahiye".
- **Server/instance statelessness** (upar section 3) = app instance ki memory me koi user-specific data nahi.

Dono ka fayda same: koi bhi instance koi bhi request le sake → horizontal scaling. Ek REST API bhi stateful ho sakti hai agar session instance ki RAM me rakh do — isliye dono ko alag samjho. (Tumhari company ke REST APIs + JWT/Redis sessions = practically dono rules follow.)

---

## 5. Auto-Scaling

**Auto-scaling** = load ke hisaab se **instances ki ginti automatically badhana/ghatana**. Cloud metrics dekhta rehta hai, aur rule match hone pe naya instance chalu karta hai ya ek band karta hai. (AWS Auto Scaling Group, Kubernetes HPA.)

```
CPU 80% (3 min) → naya instance launch → boot + health check (1-5 min) → LB usko traffic dena shuru
CPU 20% (10 min) → ek instance ko LB se nikaalo → current requests khatam hone do → band
```

| Metric | Scale-out (add) | Scale-in (remove) |
|---|---|---|
| CPU utilization | > 70% for 3 min | < 30% for 10 min |
| Request count | > 1000 RPS/instance | < 200 RPS/instance |
| Queue depth | > 1000 pending | < 100 pending |
| Response time (P95) | > 500ms | < 100ms |

Notice: **add jaldi (3 min), remove dheere (10 min).** Load badhne pe users dukhi hote hai; extra instance thodi der zyada chale toh bas thoda paisa jaata hai.

### Key parameters

**1. Cooldown period — aur "thrashing" kya hai**

**Thrashing (ya "flapping")** = system baar-baar instance add-remove karta rehta hai, bina stable hue:
```
10:00  CPU 75% → +1 instance
10:01  naya instance abhi boot ho raha, CPU abhi bhi 75% → +1 aur  (ghabraahat me)
10:03  dono ready, CPU 30% → -1
10:04  CPU phir 70% → +1 ...   ← ye loop = thrashing
```
Har launch me paisa + boot time lagta hai, aur instance band hone pe uske requests disturb hote hai. **Cooldown** = ek scaling action ke baad **N minute (5-10) tak naya action mat lo**, taaki naya instance warm ho jaye aur metrics settle ho jaaye. Phir dekho ki zarurat hai ya nahi.

**2. Min/max instances**
- **Min (e.g. 2)** — raat ko load zero ho tab bhi kam se kam 2 chalu. Ek mare toh doosra sambhale (redundancy), aur achanak spike pe zero se boot karne ka wait na ho. Usually alag availability zones me.
- **Max (e.g. 50)** — upar ki chhat. Agar koi bug (infinite retry loop) ya DDoS attack CPU bhar de, auto-scaler 5000 instances na khol de aur bill ₹ lakhon me na chala jaye. Ye DB ko bhi bachata hai — 5000 instances × connection pool DB ko gira dega.

**3. Predictive / scheduled scaling**
Naye instance ko ready hone me **minutes** lagte hai. Agar spike **ek second me** aata hai (IPL match shuru, 12 baje sale, Tatkal booking 10 AM), toh reactive scaling late hogi — pehle 5 minute users ko errors milenge. Isliye spike se **pehle** scale karo:
- **Scheduled:** "har din 9:50 AM pe min=40 kar do" (Tatkal).
- **Predictive:** cloud pichle hafton ka pattern seekh ke khud pehle se scale karta hai (AWS Predictive Scaling).

---

## 6. Latency percentiles — P50, P95, P99 ⭐

**Problem:** "Average latency 263ms" sunke lagta hai sab theek hai. But ho sakta hai kuch users 2 second wait kar rahe ho. Average ye chhupa deta hai.

**Percentile kaise nikalta hai:** saari requests ka time **sort** karo (fast se slow), phir dekho list me X% pe kaunsa value hai.

**Example — 10 requests (ms):**
```
sorted: 50, 50, 60, 60, 70, 70, 80, 90, 100, 2000
```
- **Average** = 2630 / 10 = **263ms** → kisi bhi user ko 263ms nahi laga! Ek 2000ms wala outlier average ko kheench raha hai.
- **P50 (median)** = beech wala ≈ **70ms** → "aadhe users ko 70ms se kam laga". Typical experience.
- **P90** = 9th value = **100ms** → "90% users ko 100ms ya kam".
- **P99** (100 requests me 99th value) → sabse slow 1% ka experience. Yahan woh 2000ms dikhega.

**Ek line me:** **P95 = 500ms** matlab "100 me se 95 requests 500ms ya usse fast hui; sirf 5 isse slow".

**P99 itna important kyu (interview gold):**
1. **Ek page = kai API calls.** Agar page load 50 backend calls karta hai aur har call ka 1% chance slow hai, toh kam se kam ek slow call ka chance = 1 − 0.99⁵⁰ ≈ **40%**. Matlab tumhara "rare" P99 lagbhag har doosre page load pe user ko dikh raha hai.
2. **Slowest users aksar sabse important hote hai** — jinke cart me zyada items, zyada data → zyada slow queries. Wahi tumhare best customers hai.
3. **Tail latency** = P99/P99.9 wala hissa. Yahi timeouts, retries aur cascading failures ka source hota hai.

**Real usage:**
- **SLO** (target): *"P99 latency < 300ms"* — 99% requests 300ms ke andar.
- Dashboards (Datadog, Grafana) me P50, P95, P99 teen lines dikhte hai. P50 theek aur P99 kharaab = kuch specific requests/users me problem (bada payload, cold cache, GC pause, slow DB query).
- **Auto-scaling trigger** (upar table) me P95 isi liye use hota hai.

> **Yaad rakho:** P50 = typical user. P95/P99 = unlucky users. **Average ko kabhi latency metric mat bolna** interview me.

---

## 7. QPS — asal me kya hai, aur concurrency vs parallelism

### QPS kya hai?
**QPS (Queries Per Second)** = **ek second me kitni queries complete hui** — ek **rate/speed** hai, jaise km/h. (APIs ke liye isko **RPS — Requests Per Second** bolte hai.)

**100K QPS ka matlab ye NAHI ki 100,000 queries ek hi instant (t0) pe saath chal rahi hai.** Matlab hai ki 1 second ke andar, alag-alag milliseconds pe aate-jaate, total 100,000 queries poori hui.

**Ek instant pe kitni chal rahi hoti hai?** — **Little's Law** se:
```
In-flight (ek waqt pe chal rahi) = QPS × average query time

100,000 QPS × 2ms (0.002s) = 200 queries ek waqt pe "in flight"
```
Toh 100K QPS pe bhi DB ek moment pe sirf ~200 queries handle kar raha hota hai — kyunki har query 2ms me khatam hoke jagah khaali kar deti hai. Agar queries slow ho jaye (20ms), toh in-flight 2,000 ho jaata hai → connections khatam, queue lambi, sab slow. **Isliye slow queries scaling ka dushman hai.**

### App server se DB tak kaise flow hota hai
```
User request → App instance (FastAPI)
              → connection pool se ek khuli DB connection uthao (TCP socket, lesson 01)
              → SQL query us connection pe bhejo
              → DB ek worker (Postgres me har connection ka ek process) query chalata hai, CPU cores pe
              → result wapas → connection pool me wapas
```
- Ek DB connection pe ek time **ek hi query** (Postgres me). Toh fleet ke total connections = max in-flight queries. 10 app instances × 20 pool = 200 connections → upar wale ~200 in-flight ke liye just enough.
- DB ke paas maan lo 32 CPU cores → ek instant pe max ~32 queries **truly parallel** CPU pe; baaki disk/network ka wait kar rahi hoti hai ya apni baari ka.

### Kya numbers ki koi chhat hai? (100K+ pe ruk jaata hai?)
Nahi. Table ke numbers **typical points** hai jahan ek technique ki zarurat padti hai — **ceiling nahi**. Sharding roughly linear scale karta hai: ek shard ~10K writes/sec le, toh 100 shards ≈ 1M writes/sec. Cache bhi cluster me nodes add karke badhta hai (Redis cluster millions ops/sec). Bade players **millions to 100M+ requests/sec** handle karte hai — e.g. AWS ne bataya tha ki Prime Day pe DynamoDB peak 100M+ requests/sec gaya. Limit tech ki kam, **paisa aur complexity** ki zyada hoti hai.

### Sequential vs Concurrent vs Parallel vs Simultaneous vs Async

| Term | Matlab | Kitchen analogy |
|---|---|---|
| **Sequential (serial)** | Ek kaam poora, phir agla | 1 chef, 1 dish poori, phir agli |
| **Concurrent** | Kai kaam **ek hi time period me progress** kar rahe — beech-beech me switch karke. 1 core pe bhi possible | 1 chef: dal chulhe pe chadha di, jab tak pak rahi, roti bel li, phir sabzi chalai |
| **Parallel** | Kai kaam **literally usi instant** chal rahe — alag cores/machines pe | 3 chefs, 3 chulhe, teen dishes ek saath |
| **Simultaneous** | Roz-marra ka word = "ek hi waqt pe". Technical baat me isko **parallel** samjho | — |
| **Asynchronous** | Kaam shuru karke **uske khatam hone ka wait nahi karte** — done hone pe callback/await. Ye "waiting" ke baare me hai, execution ke nahi | Chef ne pressure cooker chadhaya, siti ka intezaar khade-khade nahi karta |

Ek line (Rob Pike): *"Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once."*

**Tumhari duniya me:**
- **JavaScript `Promise.all([...])`** = concurrent (ek thread, event loop) — parallel nahi. **Web Workers** = parallel.
- **Python `asyncio` / FastAPI `async def`** = concurrent (ek thread, I/O wait ke time doosri request). **Multiple uvicorn workers / multiprocessing** = parallel (alag cores).
- **DB pe 100K QPS** = bahut saari queries concurrent (overlapping), aur unme se jitne CPU cores hai utni ek instant pe parallel.

> Network-heavy backend (DB calls, API calls) ko zyada **concurrency** chahiye (wait ke time doosra kaam). CPU-heavy kaam (image processing, ML inference) ko **parallelism** chahiye (zyada cores).

---

## 8. Database ko scale karna (the hardest part)

Database usually **sabse mushkil** component hai scale karne me — kyunki ye **stateful** hai (data yahin rehta hai; app tier ki tarah "bas copy chala do" nahi kar sakte). Progression (har step ek specific bottleneck pe trigger hota hai, pehle se nahi):

| Stage | Roughly kab chahiye | Technique |
|---|---|---|
| Single DB | < 1K QPS | Ek powerful machine |
| Vertical scale | 1-5K QPS | Bada machine (more RAM, faster SSD) |
| **Read replicas** | 10-50K read QPS | Reads replicas pe, writes primary pe (Topic 2 lesson 12) |
| **Cache layer** | 100K+ read QPS | Redis/Memcached 90%+ reads absorb kare (Topic 3) |
| **Sharding** | 50K+ write QPS | Data multiple DB instances me split (Topic 2 lesson 13) — shards badha ke millions tak |
| Multi-region | Global scale | Regions me replicate, low latency |

> Ye poori progression tumne Topic 2 (replication, sharding) aur Topic 3 (caching) me detail me dekhi — yahan ye **ek unified ladder** hai. Numbers rough hai (query kitni heavy hai, uspe depend). Key insight: **har step ek bottleneck pe trigger hota hai, preemptively nahi.**

---

## 9. Scaling strategies comparison + mistakes

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
> - **Max instances na set karna** → bug/DDoS pe bill aur DB connections dono phat jaate hai.
> - **App tier scale karna jabki DB bottleneck hai** — galat jagah optimize (Topic 3 lesson 01: "optimize the bottleneck"). Zyada instances = zyada DB connections = DB aur zyada dukhi.
> - **Average latency dekhna** P95/P99 ki jagah.

**Interview: "100 users se 10M users tak kaise scale?"** — Progressive: (1) single server + DB → (2) app aur DB alag machines → (3) LB + multiple app servers (stateless!) → (4) Redis cache → (5) read replicas → (6) CDN → (7) shard DB (jab writes bottleneck) → (8) message queues async → (9) multi-region. **Har step ek specific bottleneck pe, preemptively nahi.**

---

## Quick Self-Check
1. Scalability ki definition — pizza analogy se vertical vs horizontal?
2. Vertical aur horizontal scaling — 3 differences (limit, cost, failure)?
3. "Instance" kya hai? 10 instances = 10 machines? 
4. Stateful vs stateless — shopping cart example se. Sticky sessions → cloud → serverless history?
5. Stateless vs serverless me farak? Service ko stateless kaise banate ho (4 cheezein kaha move karo)?
6. REST kya hai (5-6 rules)? Statelessness = REST?
7. Auto-scaling kaise kaam karta hai? Thrashing kya hai aur cooldown kaise rokta hai? Min/max kyu? Predictive scaling kab?
8. 10 latencies diye ho toh average, P50, P90 nikalo. Average kyu misleading hai? P99 kyu matter karta hai (50-calls example)?
9. 100K QPS ka matlab kya hai? Little's Law se in-flight queries nikalo (2ms query pe).
10. Sequential vs concurrent vs parallel vs async — kitchen analogy se. `Promise.all` aur `asyncio` kaunsa hai?
11. Database scaling progression ke 6 stages? Kya 100K QPS ceiling hai?
12. "Premature sharding" mistake kya hai — pehle kya try karna chahiye?

---

## Flow — load badhne pe ek request kaise handle hoti hai
1. User request (cookie/JWT ke saath) **LB** pe aati hai → LB use **kisi bhi** healthy instance pe bhej deta hai (stateless hai, toh koi bhi chalega).
2. Instance user ki state **bahar** se padhta hai (JWT verify / Redis session), data pehle **cache**, miss pe connection pool se **DB** (reads replica pe, writes primary pe).
3. Har instance pe bahut saari requests **concurrently** chalti hai (async I/O), aur alag instances/cores pe **parallel** — total throughput QPS/RPS kehlata hai.
4. Monitoring CPU / RPS / **P95 latency** dekhta hai. Threshold cross → auto-scaler naya instance boot karta hai → health check pass → LB traffic dene lagta hai. **Cooldown** thrashing rokta hai, **min/max** limits bound karti hai.
5. Agar app tier badhane ke baad bhi slow → bottleneck DB hai → ladder chadho: cache → read replicas → sharding.

---
**Next:** `05-Load-Balancing.md` — multiple servers me traffic distribute karna.