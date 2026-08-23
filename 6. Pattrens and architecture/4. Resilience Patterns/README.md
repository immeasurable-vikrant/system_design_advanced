# 04 — Resilience Patterns (Fault Tolerance)

> **Is lesson me:** Ek service fail ho toh poora system girne se kaise bachao — circuit breaker, bulkhead, timeout, retry, fallback/graceful degradation. Topic 1 (circuit breaker) + Topic 4 (retry/idempotency/DLQ) me ye touch huye; yahan consolidated as an architectural concern. Microservices me ye ESSENTIAL hai (kyunki har network call fail ho sakta hai).

Prerequisites: `01-Monolith-vs-Microservices.md`, Topic 4 lesson 07 (retries)

---

## 1. Kyu resilience patterns zaroori hai

Microservices me har service call ek **network call** hai (Topic 1 lesson 01) — aur network calls fail hote hai (timeout, service down, slow). Ek monolith me function call fail nahi hoti; microservices me **failure normal hai** (Topic 5 lesson 01).

**The killer scenario — cascading failure:**
```
Payment Service slow ho gayi (DB issue)
  → Order Service Payment ka wait karti hai (blocked threads)
  → Order Service ke saare threads Payment ke wait me atke
  → Order Service bhi unresponsive
  → Order ko call karne wali API Gateway bhi atki
  → POORA SYSTEM down — ek service ki wajah se! 💥
```
Ek slow service **poore system ko le doobti hai** agar resilience patterns na ho. Ye patterns failure ko **contain** karte hai.

---

## 2. Timeout — the foundation

Har network call ka ek **timeout** hona chahiye. Bina timeout, ek slow service tumhare threads ko **forever** hold kar sakti hai.
```
Bina timeout: Payment call → Payment hang → tumhara thread forever blocked
Timeout ke saath: Payment call → 2s me response nahi → fail fast, thread free
```
> ⚠️ **Sabse common mistake:** koi timeout na set karna (ya default infinite). Har external call pe aggressive timeout lagao. Ye baaki saare patterns ki foundation hai.

**Rule of thumb:** timeout downstream ke p99 latency se thoda zyada. Bahut chhota = false failures; bahut bada = slow failure detection.

---

## 3. Retry (with backoff + jitter) — recap

> **Recap (Topic 4 lesson 07):** transient failure pe retry karo, but **exponential backoff + jitter** ke saath (immediate retry = retry storm). Aur **sirf idempotent operations** retry karo (Topic 4 lesson 05) — warna duplicate side-effects (double charge!).

```
Attempt 1 fail → wait 1s (+jitter) → retry
Attempt 2 fail → wait 2s (+jitter) → retry
Attempt 3 fail → give up (circuit breaker / DLQ)
```

⚠️ **Retry ka danger:** agar service already overloaded ho, retries use aur maar dete hai. Isliye retry + circuit breaker saath me.

---

## 4. Circuit Breaker ⭐ — the key pattern

> **Recap (Topic 4 lesson 08):** circuit breaker ek electrical fuse jaisa hai — ek service baar-baar fail kare toh us pe calls **temporarily rok** do (fail fast), taaki ek slow service poore system ko na le doobe. Yahan full mechanism:

**3 states:**
```
CLOSED (normal):  calls jaane do. Failures count karo.
                  Failure threshold cross (e.g. 50% in 10s) → OPEN

OPEN (tripped):   calls TURANT fail karo (service ko call bhi mat karo).
                  Fail fast — no waiting, no thread blocking.
                  Ek timeout (e.g. 30s) ke baad → HALF-OPEN

HALF-OPEN (testing): kuch test calls jaane do.
                     Success → CLOSED (recovered)
                     Fail → wapas OPEN (abhi bhi down)
```

```
      failures > threshold
CLOSED ──────────────────▶ OPEN
  ▲                          │
  │ success            timeout elapsed
  │                          ▼
  └───────── HALF-OPEN ◀─────┘
     test calls succeed
```

**Kyu powerful:** OPEN state me, calls **turant fail** hoti hai (no waiting) → tumhare threads free rehte hai → cascading failure ruk jaati hai. Aur service ko "saans" milti hai recover karne ko (hammering band). Jab wo theek ho, half-open automatically detect karke wapas normal.

**Frontend analogy:** ye us UI pattern jaisa hai jaha ek API baar-baar fail kare toh tum retry button disable kar dete ho + "service unavailable" dikhate ho, kuch der baad automatically retry enable — bina har baar failing call kiye.

**Tools:** Resilience4j, Hystrix (legacy), Envoy/Istio (service mesh level — Topic 1 lesson 06).

---

## 5. Bulkhead — isolate resources

**Bulkhead** (ship ke watertight compartments se naam) = resources ko isolate karo taaki ek part ka failure baaki ko na doobaye.
```
Bina bulkhead: ek shared thread pool (100 threads)
  Payment slow → saare 100 threads Payment ke wait me atke
  → koi thread nahi bacha Users/Orders ke liye → sab down

Bulkhead ke saath: alag thread pools per dependency
  Payment pool (30), Users pool (30), Orders pool (40)
  Payment slow → sirf Payment ke 30 threads atke
  → Users/Orders ke pools unaffected → wo kaam karte rehte
```
Ek failing dependency ko uske apne "compartment" me contain karo. Ek leak poore ship ko nahi doobati.

**Frontend analogy:** ye connection pooling (Topic 1 lesson 09) ke resource-isolation jaisa hai — alag pools taaki ek exhaust ho toh baaki bache.

---

## 6. Fallback / Graceful Degradation

Jab ek dependency fail ho, poora fail hone ke bajaye ek **degraded but working** response do.
```
Recommendation Service down?
  → Fail nahi karo. Generic/cached recommendations dikhao (fallback)
  → User ko product page still dikhe (bas personalized recs missing)

Payment Service down?
  → "Order received, payment processing" dikhao, async retry (Topic 4)
  → Hard fail se better UX
```

> **Recap (Topic 3 lesson 05):** cache down pe DB fallback (graceful degradation). Same philosophy — degraded service > no service. Design me poocho: "is dependency ke bina main kya minimal serve kar sakta hu?"

---

## 7. Sab milke — the resilience stack

```
Har external call ke liye:
  1. TIMEOUT       → slow calls ko fail fast (foundation)
  2. RETRY         → transient failures (backoff + jitter, idempotent only)
  3. CIRCUIT BREAKER → repeated failures pe fail fast, service ko saans do
  4. BULKHEAD      → failing dependency ko isolate (separate pools)
  5. FALLBACK      → graceful degradation (degraded > down)
```

**Interview line:** *"Har inter-service call ko timeout + circuit breaker + retry (backoff/jitter, idempotent only) se wrap karunga. Critical dependencies ko bulkhead se isolate. Non-critical dependencies ke liye fallback (cached/default response). Goal: ek service ka failure poore system me cascade na kare."* Ye zyadatar service mesh (Istio — Topic 1 lesson 06) declaratively provide karta hai.

---

## Quick Self-Check
1. Cascading failure kya hai — ek slow service poore system ko kaise le doobti?
2. Timeout kyu foundation hai? Common mistake kya?
3. Retry me kya dhyan (backoff, jitter, idempotent — Topic 4 recap)? Danger?
4. Circuit breaker ke 3 states + transitions? OPEN state cascading failure kaise rokta?
5. Bulkhead kya isolate karta hai (thread pools)? Ship analogy?
6. Fallback / graceful degradation — recommendation service down pe kya (Topic 3 recap)?
7. Resilience stack ke 5 patterns (timeout se fallback)?

---
**Next:** `05-Event-Driven-Architecture.md` — events ko system ka backbone banana.