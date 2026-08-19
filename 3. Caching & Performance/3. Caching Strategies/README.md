# 03 — Caching Strategies (Cache + DB Coordination)

> **Is lesson me:** 4 strategies jinse cache aur database coordinate karte hai — cache-aside (most common), write-through, write-behind, read-through. Har ek ka read/write flow, pros/cons, aur kab kaunsa. Ye lesson decide karta hai tumhara data kitna fresh vs kitna fast hoga.

Prerequisites: `02-Caching-Fundamentals-And-Layers.md`

---

## Setup: sawaal ye hai

Cache aur database dono me data hai. **Reads kahan se, writes kahan, aur consistency kaise?** In sawaalon ke alag jawaab = alag strategies. Chalo 4 dekhte hai.

---

## 1. Cache-Aside (Lazy Loading) — most common ⭐

Application khud cache ko explicitly manage karta hai. Cache ek **alag system** hai jise app "side me" use karta hai.

```
READ:
  1. Cache check karo → HIT? → return
  2. MISS? → database query karo
  3. Result cache me daalo (with TTL)
  4. Return

WRITE:
  1. Database update karo
  2. Cache key ko INVALIDATE (delete) karo
     (agli read MISS hogi → fresh data fetch → re-cache)
```

Code (yaad hai DB lesson 14 me dekha tha — ye wahi, full):
```java
User getUser(String userId) {
    User cached = redis.get("user:" + userId);
    if (cached != null) return cached;             // HIT

    User user = db.query("SELECT * FROM users WHERE id=?", userId);  // MISS
    redis.set("user:" + userId, user, Duration.ofMinutes(5));        // re-cache
    return user;
}

void updateUser(User user) {
    db.update(user);
    redis.delete("user:" + user.getId());          // invalidate
}
```

**Pros:** sirf actually-requested data cache hota hai (memory efficient). Simple. Cache down ho toh system chalta rehta hai (DB fallback).
**Cons:** har key ki **pehli request slow** (guaranteed miss). Agar koi doosra service DB update kare aur cache invalidate na kare → **stale**.

**Kab:** read-heavy workloads. **Interview me default yahi bolo.**

**Frontend analogy:** React Query / SWR **exactly** yahi karta hai — cached data serve, miss pe fetch, mutation pe `invalidateQueries`. Redis cache-aside = "React Query for your backend".

---

## 2. Write-Through

Har write **cache aur database dono** me jaata hai, synchronously.

```
WRITE:
  1. Cache me likho
  2. Cache database me likhe (synchronously)
  3. Success return

READ:
  1. Hamesha cache se (wo hamesha up-to-date hai)
```

**Pros:** cache kabhi stale nahi. Reads hamesha fast + fresh.
**Cons:** write latency badh jaata hai (do writes). Cache aisa data bhi store karta hai jo shayad kabhi read na ho (memory waste).

**Kab:** jab stale reads tolerate nahi kar sakte — financial data, user sessions.

---

## 3. Write-Behind (Write-Back)

Cache me turant likho, database me **asynchronously** (batches me) baad me flush karo.

```
WRITE:
  1. Cache me likho → TURANT success return
  2. Background worker har N seconds me DB me batch-flush kare

READ:
  1. Hamesha cache se
```

**Pros:** write latency minimal (sirf cache write). High write-throughput ke liye great.
**Cons:** **data loss risk** — agar cache crash ho DB me flush hone se pehle, wo writes gaye. Complex.

**Kab:** high write throughput jaha thoda data loss acceptable ho — view counts, analytics events, non-critical session data.

> **DB lesson 10 (WAL) connection:** Write-behind ka data-loss risk exactly wahi problem hai jo WAL solve karta hai databases me. Agar tum write-behind use karo aur durability chahiye, tumhe cache pe bhi ek append-only log (jaise Redis AOF) chahiye. Warna "cache crashed before flush = lost writes".

---

## 4. Read-Through

Cache **khud** database se fetch karta hai on miss — application ko pata bhi nahi (application DB se directly baat hi nahi karta reads ke liye).

```
READ:
  1. App cache.get(key) call kare
  2. Cache check → MISS → CACHE khud DB query kare
  3. Cache result store karke app ko return kare
```

**Cache-aside vs read-through — key difference:**
- **Cache-aside:** *application* miss handle karta hai (DB query app ke code me). Cache "dumb" store hai.
- **Read-through:** *cache library* miss handle karta hai (DB loader cache me built-in). App ko sirf cache dikhta hai.

**Kab:** jab ek caching library use kar rahe ho jo ye support kare (e.g., Caffeine with a loader function).

---

## 5. Comparison table (yaad rakhne layak)

| Strategy | Read latency | Write latency | Freshness | Complexity |
|---|---|---|---|---|
| **Cache-Aside** | Miss: slow, Hit: fast | Normal (DB only) | May be stale (TTL-bound) | Low |
| **Write-Through** | Always fast | Slow (2 writes) | Always fresh | Medium |
| **Write-Behind** | Always fast | Very fast (cache only) | Fresh in cache, DB lags | High |
| **Read-Through** | Miss: slow, Hit: fast | Normal | TTL-bound | Low |

**Common production combo:** **Cache-aside for reads + write-through (ya explicit invalidation) for writes.** Ye balance deta hai — reads fast, writes reasonably fresh.

---

## 6. Decision guide

```
Read-heavy, thoda staleness ok?        → Cache-Aside (default, most systems)
Stale bilkul nahi chahiye (money)?     → Write-Through
Write-heavy, thoda loss ok (counts)?   → Write-Behind
Library-managed clean abstraction?     → Read-Through
```

**Interview line:** *"Main cache-aside default rakhunga read-heavy paths ke liye — simple hai aur cache failure pe DB fallback deta hai. Writes pe cache key explicitly invalidate karunga. Agar koi path stale bilkul tolerate na kare (balance display), wahan write-through. Analytics/counters ke liye write-behind, kyunki wahan throughput > perfect durability."*

---

## Quick Self-Check
1. Cache-aside ka read flow aur write flow — dono steps me?
2. Cache-aside kis frontend library jaisa hai, aur kyu?
3. Write-through vs write-behind — latency aur data-loss ke terms me difference?
4. Write-behind ka data-loss risk kis DB concept (lesson 10) se relate karta hai?
5. Cache-aside aur read-through me core difference (kaun miss handle karta hai)?
6. Money-related data ke liye kaunsi strategy, aur view-counts ke liye kaunsi?

---
**Next:** `04-Eviction-And-Invalidation.md` — cache full ho toh kya hataao, aur stale kaise rokо.