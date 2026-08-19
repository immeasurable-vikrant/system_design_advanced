# 05 — Cache Failure Modes (When Caching Backfires)

> **Is lesson me:** Caching kab ULTA problem banati hai — cache stampede (thundering herd), hot keys, cache penetration, aur cache down hone pe kya. Ye woh "gotchas" hai jo interviews me tumhe senior dikhaate hai, aur production me system girate hai.

Prerequisites: `04-Eviction-And-Invalidation.md`

---

## 1. Cache Stampede (Thundering Herd) — the classic

**Problem:** Ek **popular** cache key expire hoti hai. Us exact moment pe 1000 concurrent requests aati hai — sab **miss** hoti hai (key gayab) — sab **ek saath database** pe jaati hai → DB overwhelmed → crash.

```
TTL expires for "popular_product_123"
Thread 1:  cache miss → query DB ─┐
Thread 2:  cache miss → query DB ─┤
Thread 3:  cache miss → query DB ─┼──▶ DB pe 1000 identical queries
...                               │     EK SAATH → DB crash 💥
Thread 1000: cache miss → query DB┘
```

Irony: caching **DB ko bachane** ke liye thi, but expiry ke ek moment pe usne DB ko **maar** diya. Ek hi data ke liye 1000 redundant queries.

### Solutions

**(a) Lock / Mutex** — sirf pehla thread DB jaaye, baaki wait kare:
```java
Product getProduct(String id) {
    Product cached = redis.get("product:" + id);
    if (cached != null) return cached;

    // Lock acquire karne ki koshish (Redis SET NX = "set if not exists")
    boolean gotLock = redis.set("lock:product:" + id, "1", "NX", "EX", 5);
    if (gotLock) {
        Product p = db.get(id);                          // sirf main DB jaata hu
        redis.set("product:" + id, p, Duration.ofMinutes(5));
        redis.delete("lock:product:" + id);
        return p;
    } else {
        Thread.sleep(100);        // koi aur fetch kar raha, wait
        return getProduct(id);    // retry → ab cache hit hoga
    }
}
```
Pehla thread lock jeetta hai, DB se fresh laata hai, cache bharta hai. Baaki 999 wait karke **refreshed cache** se padhte hai. DB pe sirf **1 query**.

**(b) Early Recompute** — expiry se pehle refresh. Background job TTL ke 80% pe cache refresh kar deta hai, taaki wo expire hi na ho jab traffic aaye.

**(c) Stale-While-Revalidate** — expire ho gaya toh bhi **purana (stale) data turant serve** karo, aur background me fresh fetch karo. User ko thoda purana milta hai but fast, aur DB stampede nahi.

> **Frontend analogy:** `stale-while-revalidate` HTTP cache directive + SWR library ka naam isi se aaya! SWR = "stale data dikhao, chupke se revalidate karo". Wahi pattern backend cache pe.

---

## 2. Hot Keys — ek key pe puri duniya

**Problem:** Ek single cached item pe **millions of requests** — celebrity profile, viral post, flash-sale product. Redis single-threaded hai (DB lesson 07), toh ek key pe itna load us Redis instance ko bottleneck bana deta hai (chahe baaki keys idle ho).

**Solutions:**
| Solution | Kaise |
|---|---|
| **Local cache (Layer 3)** | Har app server apni RAM me hot key rakhe, very short TTL (~10s). Redis load massively kam (lesson 02 — L3 in-process). |
| **Replicate the key** | Same value ko multiple keys pe copy — `product:123:copy1..N` — reads randomly distribute |
| **Client-side cache** | Redis 6+ "client-side caching" — client apni copy rakhe, Redis invalidate push kare |

**Interview line (yaad hai DB caching notes se):** *"Hot key ke liye main har server pe ek local in-memory cache (Layer 3) rakhunga 10-second TTL ke saath. Isse millions of reads Redis tak pahunche hi nahi — har server apni RAM se serve karta hai. Thoda staleness (10s) acceptable hai celebrity profile ke liye."*

---

## 3. Cache Penetration — non-existent data ka attack

**Problem:** Requests aisi keys maangti hai jo **exist hi nahi karti** (na cache me, na DB me). Har aisi request: cache miss → DB query → DB me bhi nahi → kuch cache nahi hota → **agli same request phir DB tak**. Cache bypass ho jaata hai. Malicious actor jaan-boojh ke random non-existent IDs maang ke DB overwhelm kar sakta hai.

```
Request "user:99999999" (exist nahi karta)
  → cache miss → DB query → DB: "nahi hai" → cache me kuch nahi daala
Agli same request → PHIR cache miss → PHIR DB → ... har baar DB tak
```

**Solutions:**
1. **Cache the negative result** — "nahi hai" ko bhi cache karo (short TTL): `SET user:99999999 "NULL" EX 60`. Agli baar cache hit (empty), DB nahi jaata.
2. **Bloom Filter** ⭐ — ek bloom filter me saari **valid** keys rakho. Request aaye → pehle bloom filter poochо "ye key exist bhi karti hai?" → agar bloom bole "definitely nahi" → turant reject, DB/cache bypass. **Ye bloom filters ka #1 caching use case hai** (lesson 09 me poora).

> **Yahi bridge hai bloom filters tak (lesson 09):** cache penetration ko elegantly solve karne ke liye bloom filter perfect hai — "definitely not present" ko RAM me O(1) me answer karta hai, DB ko bekaar lookups se bachata hai.

---

## 4. Cache down ho gaya toh? (graceful degradation)

**Sawaal (common interview):** "Agar Redis crash ho jaye, kya hoga?"

**Sahi jawaab:** saari requests **DB tak fall through** karti hai. System **slow** ho jaata hai but **kaam karta rehta hai** (kyunki cache-aside me DB source-of-truth hai — lesson 03). Design principle: **cache ko single point of failure mat banao** — cache optional speed layer ho, mandatory nahi.

**Danger:** agar Redis down + traffic high → saara traffic ek saath DB pe (stampede jaisa) → DB bhi gir sakta hai. Mitigations:
- **Circuit breaker** — DB overload detect ho toh kuch requests fail-fast (poora system girne se better)
- **Rate limiting** (lesson 07!) — DB tak jaane wale requests ko cap karo
- **Multiple Redis replicas** (DB lesson 12 — replication) — ek gire toh doosra serve kare

**"Fail open vs fail closed":**
- **Fail open** — cache down → sab DB tak jaane do (availability first, DB risk)
- **Fail closed** — cache down → kuch reject karo (DB bachao, kuch users unhappy)
Usually **fail open** for cache (better to be slow than down), but rate-limit + circuit-breaker ke saath.

---

## 5. Summary — caching ke 4 gotchas

| Failure mode | Ek line | Fix |
|---|---|---|
| Stampede | Popular key expire → 1000 miss → DB flood | Lock, early recompute, stale-while-revalidate |
| Hot key | Ek key pe millions → Redis bottleneck | Local cache (L3), replicate key |
| Penetration | Non-existent keys → har baar DB | Cache negatives, **bloom filter** |
| Cache down | Redis crash → DB flood | Graceful degradation, circuit breaker, replicas |

---

## Quick Self-Check
1. Cache stampede kya hai — root cause aur wo irony kya hai?
2. Lock-based stampede fix me pehla thread kya karta hai, baaki 999 kya?
3. `stale-while-revalidate` kis frontend library ke naam se relate karta hai?
4. Hot key problem kyu hoti hai (Redis ki kaunsi property)? Best fix?
5. Cache penetration kya hai, aur bloom filter ise kaise rokta hai?
6. Redis down hone pe system ko kaisa behave karna chahiye (graceful degradation)?
7. Fail open vs fail closed — cache ke liye usually kaunsa?

---
**Next:** `06-Redis-vs-Memcached.md` — cache engine choose karna.