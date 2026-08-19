# 09 — Connection Pooling

> **Is lesson me:** Connections banana mehenga kyu hai (breakdown), connection pool kaise kaam karta hai, **pool sizing** (aur "zyada connections = better" kyu galat hai), pool config parameters, application-side vs proxy-side pooling (PgBouncer), connection leaks, aur serverless ka special problem. Topic 2 me isko chhua tha — ye full depth hai, aur infra→database ka bridge.

Prerequisites: `01-Networking-I` (TCP handshake, RTT). Helpful: Topic 2 (Database).

---

## 1. Connection pooling kya hai

**Connection pooling** = pehle se bani hui (pre-established) database connections ka ek set maintain karna jo requests ke across **reuse** hota hai — har query ke liye nayi connection banane aur band karne ke bajaye.

**Analogy:** Ek busy office building with a security gate. Bina pooling, har employee poora security check (ID, bag scan, badge print) har baar karta hai — chahe wo sirf coffee ke liye bahar gaya ho. Pooling ke saath, employees apna badge (connection) poore din active rakhte hai — enter karne ke liye bas tap. Badges (connections) limited hai, toh sab busy ho toh naye employees line me wait karte hai.

---

## 2. Connections mehenge kyu hai

Ek nayi database connection banane me:
| Step | Action | Time |
|---|---|---|
| 1 | DNS resolution | 1-5 ms |
| 2 | TCP handshake (SYN/ACK — lesson 01) | 1-3 ms |
| 3 | TLS handshake (if SSL — lesson 03) | 5-30 ms |
| 4 | Authentication (username/password) | 2-10 ms |
| 5 | Server session ke liye memory allocate | 5-10 MB per connection |
| 6 | Connection ready | **Total: 10-50 ms** |

Ek web request jo 2ms ki query kare, uspe 30ms connection setup = **15x overhead**. 1000 RPS pe = 1000 connections/sec banao-todo = massive waste.

```
Without pool (naya per request):        With pool (reuse):
Request 1 → create conn (30ms) →        Request 1 → borrow (0.1ms) →
  query (2ms) → close                     query (2ms) → return to pool
Request 2 → create conn (30ms) →        Request 2 → borrow SAME conn (0.1ms) →
  query (2ms) → close                     query (2ms) → return
```

> **Yaad hai lesson 01** — TCP + TLS handshake ka RTT tax? Connection pooling exactly wahi tax bachata hai, database connections ke liye — pehle se khuli connections reuse karke.

---

## 3. Pool kaise kaam karta hai

```
App request → pool me idle connection hai?
   HAAN → borrow karo
   NAHI → pool max size pe hai?
            NAHI → nayi conn banao, pool me add
            HAAN → queue me wait karo
                     → timeout exceed? → connection timeout error
→ query execute → connection wapas pool me (idle, reuse ready)
```

**Pool lifecycle:**
1. **Initialization** — startup pe `minIdle` connections banao
2. **Borrow** — app maange → pool ek de
3. **Use** — queries chalao
4. **Return** — app wapas kare (band nahi karta)
5. **Validation** — pool check kare connection abhi zinda hai
6. **Eviction** — `minIdle` se zyada idle connections `idleTimeout` ke baad band

---

## 4. Pool Sizing — "zyada = better" GALAT hai ⭐

Ye counterintuitive hai but important: **bahut zyada connections actually performance HURT karti hai.**

**Formula (PostgreSQL wiki / HikariCP):**
```
Pool Size = (CPU cores × 2) + disk spindles
SSD-backed ke liye: Pool Size = CPU cores × 2 + 1
```

**Kyu chhote pools better:**
| Pool Size | Behavior |
|---|---|
| Too small (5) | Requests queue up, high latency under load |
| **Optimal (10-20)** | Max throughput, CPU + disk fully utilized |
| Too large (200) | **Context switching overhead, lock contention, WORSE throughput** |

Ek 4-core PostgreSQL server usually **9-10 connections** pe best perform karta hai, **200 pe nahi**.

> **Kyu 200 connections 20 se slow (interview Q):** database ke paas limited CPU cores hai (say 8). 200 concurrent connections sab queries chala rahi ho → DB actual query execute karne se zyada time **context-switching** me lagata hai. Lock contention badhti hai, CPU cache misses badhte hai, disk I/O random ho jaata hai. 20 connections DB ko full CPU se 20 queries pe focus karne dete hai, baaki briefly pool queue me wait — higher overall throughput, lower average latency.

**Frontend analogy:** ek chef (CPU core) ek time pe 2-3 dishes achhe se bana sakta hai. Usse 200 orders ek saath do → wo har dish ke beech switch karta rahega, kuch achha nahi banega. 20 orders = focused, fast. Yehi context-switching overhead hai.

---

## 5. Pool config parameters

| Parameter | Kya karta hai | Recommended |
|---|---|---|
| `maxPoolSize` | Max connections | 10-30 (DB cores pe depend) |
| `minIdle` | Idle pe kitni alive rakho | 5-10 |
| `connectionTimeout` | Connection ke liye max wait | 3-10 sec |
| `idleTimeout` | Idle conn kab band | 10-30 min |
| `maxLifetime` | Force-close (stale rok) | 30-60 min |
| `validationQuery` | Conn alive test | `SELECT 1` |
| `leakDetectionThreshold` | Leak warn if not returned | 30-60 sec |

---

## 6. Application-side vs Proxy-side pooling

**Application-side (HikariCP):** pool har app instance ke **andar**.
```
App Instance 1 (pool: 10) ─┐
App Instance 2 (pool: 10) ─┼─▶ Database
App Instance 3 (pool: 10) ─┘
```
**Problem at scale:** 50 instances × 20 pool = **1000 DB connections** — most databases efficiently handle nahi kar sakte.

**Proxy-side (PgBouncer):** ek alag proxy app aur DB ke beech, jo **connection multiplexing** karta hai.
```
50 app instances (1000 app conns) → PgBouncer → 50-100 actual DB conns
```

| Aspect | Application-side (HikariCP) | Proxy-side (PgBouncer) |
|---|---|---|
| Kaha | Har app instance ke andar | Alag process, app aur DB ke beech |
| Total DB conns | N instances × pool size | Shared, fewer |
| Use case | Fixed number of instances | Many instances / serverless |
| Complexity | None (library) | Extra infra |

> **Connection multiplexing kya hai (PgBouncer):** PgBouncer hazaaron app connections accept karta hai but bahut kam actual PostgreSQL connections maintain karta hai. Jab ek app connection query chalati hai, PgBouncer use ek real DB connection assign karta hai query ke duration ke liye, phir wapas shared pool me. Isse 1000 app connections 50 DB connections efficiently share kar lete hai.

---

## 7. Connection Leaks

**Connection leak** = app code connection borrow karta hai but kabhi wapas nahi karta. Pool dheere-dheere khali ho jaata hai jab tak saare requests block.

**Common causes:** exception `close()` se pehle throw ho gaya (finally block missing), try-with-resources na use kiya, long-running transaction connection hold kiye, async code me close bhool gaye.

**Detection + prevention:**
- `leakDetectionThreshold` set karo (HikariCP stack-trace ke saath warn karta hai)
- Pool metrics monitor karo (active vs idle)
- Try-with-resources (Java) / context managers (Python `with`)
- `maxLifetime` set karo (stale connections force-reclaim)

> **Frontend analogy:** ye ek memory leak jaisa hai — resource acquire kiya but release nahi kiya (jaise `addEventListener` bina `removeEventListener`, ya subscription bina unsubscribe). Dheere-dheere resources khatam.

---

## 8. Serverless ka special problem

Serverless (AWS Lambda) me: har invocation nayi connection khol sakti hai, aur hazaaron concurrent invocations DB ki connection limit **exhaust** kar dete hai.

| Solution | Kaise |
|---|---|
| **RDS Proxy** | Lambda aur DB ke beech, connections multiplex |
| **PgBouncer** | External pooler saare Lambdas ke across shared |
| Connection reuse | Connection handler ke bahar init karo, warm invocations me reuse |
| **DynamoDB** | Problem hi avoid karo (HTTP-based, no persistent connections — Topic 2) |

> **Interesting:** DynamoDB (Topic 2 lesson 07) HTTP-based hai — persistent connections nahi, isliye connection pooling ki problem hi nahi. Ye ek design trade-off hai (HTTP overhead vs no connection management).

---

## Quick Self-Check
1. Connection pooling kya hai — office badge analogy?
2. Connection banane ke 6 steps (kya mehenga)? Ye lesson 01 ke kis concept se relate?
3. Pool kaise kaam karta hai — borrow/return/wait flow?
4. Pool sizing formula? "200 connections 20 se slow" kyu (context switching)?
5. Application-side vs proxy-side pooling — 100 instances pe kya problem, PgBouncer kaise fix?
6. Connection multiplexing kya hai?
7. Connection leak kya hai + 2 prevention (frontend memory-leak analogy)?
8. Serverless me connection problem aur DynamoDB use kaise avoid karta hai?

---
**Next:** `10-Life-Of-A-Request-Playbook.md` — sab kuch ek request ke safar me tie karo.