# 04 — Eviction & Invalidation

> **Is lesson me:** Do alag-but-related problems — **Eviction** (cache RAM full ho gayi, kya hataao? LRU/LFU/FIFO/TTL) aur **Invalidation** (DB badla, cache purana, stale kaise rokо? TTL/explicit/event-based). Invalidation ko CS ke "do sabse mushkil problems" me se ek kaha jaata hai.

Prerequisites: `03-Caching-Strategies.md`

---

## Pehle: Eviction ≠ Invalidation (confusion clear karo)

Ye do alag cheezein hai, log mix karte hai:

- **Eviction** = cache **bhar gaya** (RAM full), naye data ke liye jagah chahiye → **kaunsa purana item hataao?** (space problem)
- **Invalidation** = data **DB me badal gaya**, cache me purana pada hai → **stale entry kaise remove/refresh karo?** (freshness problem)

Dono zaroori hai. Pehle eviction.

---

## PART A — Eviction Policies (cache full → kya hataao)

Cache ki RAM limited hai. Jab bhar jaye, ek item nikaalna padta hai naye ke liye. Kaunsa?

| Policy | Kaise | Best for |
|---|---|---|
| **LRU** (Least Recently Used) | Sabse der pehle access hua item hatao | General purpose (most common) |
| **LFU** (Least Frequently Used) | Sabse kam baar access hua item hatao | Hot content bacha rahe (trending feeds) |
| **FIFO** (First In First Out) | Sabse purana inserted item hatao | Simple, time-based data |
| **TTL** (Time To Live) | Fixed time baad hatao, usage se farak nahi | Har caching (har key pe TTL set karo) |
| **Random** | Koi bhi random item hatao | Jab access pattern uniform ho |

### LRU ko samjho (sabse important)
"Jise sabse der se nahi chhua, wo shayad aage bhi nahi chahiye" — is assumption pe LRU chalta hai.
```
Cache (capacity 3), access order: A, B, C, phir A, phir D aata hai
  State: [C, B, A]  (A most-recently-used, C least)
  D aaya, jagah nahi → LRU (C) hatao → [D, A, B]
```
**Frontend analogy:** browser tabs — jo tab tumne sabse der se nahi dekha, wo memory se suspend ho jaata hai. Same LRU idea.

### LFU vs LRU
- **LRU** *recency* dekhta hai (kab last chhua). Ek viral post jo abhi-abhi thanda hua, LRU jaldi hata dega.
- **LFU** *frequency* dekhta hai (kitni baar chhua). Ek consistently-popular item bacha rehta hai even if thoda time se nahi chhua.

**Interview me bolo: "LRU + TTL".** LRU space-limit handle karta hai, TTL staleness handle karta hai. Ye combo Redis ka `allkeys-lru` + per-key `EX` deta hai.

> **Redis eviction config (practical):** Redis me `maxmemory` + `maxmemory-policy` set karte ho — `allkeys-lru`, `allkeys-lfu`, `volatile-ttl`, etc. Production me usually `allkeys-lru`. Ye tumhe manually manage nahi karna, Redis khud karta hai.

---

## PART B — Cache Invalidation (stale kaise rokо)

> *"Computer Science me sirf do hard cheezein hai: cache invalidation aur naming things."* — Phil Karlton

**Problem:** Database update ho gaya, but cache me abhi purana data pada hai. User ko **stale** (basi) information dikhti hai.

3 solutions, simple se robust tak:

### 1. TTL (Time-To-Live) — simplest
Har cache entry pe expiry set karo:
```
SET user:123 {...} EX 300     # 5 min baad auto-delete
```
5 min baad entry khud gayab → agli read miss → fresh data DB se.
**Trade-off:** data **up to 5 min stale** ho sakta hai. Chhota TTL = kam stale but kam hit-ratio; bada TTL = zyada hit-ratio but zyada stale. (Lesson 02 ka freshness-vs-speed trade-off, exactly.)

### 2. Explicit Invalidation — accurate
Write/update ke time, cache key **turant delete** karo:
```java
void updateUser(User user) {
    db.update(user);
    redis.delete("user:" + user.getId());   // invalidate NOW
}
```
Agli read miss → fresh fetch → re-cache. **Accurate** (stale window ~0), **but** har write-path pe yaad rakhna padta hai — ek jagah bhool gaye toh stale bug.

### 3. Event-Based Invalidation (CDC) — for complex systems
```
DB change → event publish → cache subscriber key delete/update kare
```
**CDC (Change Data Capture)** = database ke changes ko ek event-stream ki tarah capture karna (jaise Postgres WAL ko read karke — DB lesson 10 — ya Debezium jaisa tool). Koi bhi service DB change kare, event aata hai, cache automatically invalidate.

**Kab:** jab **multiple services** same DB me likhte hai, aur koi bhi ek cache ko stale kar sakta hai. Explicit invalidation har service me nahi laga sakte — CDC central solution deta hai.

**Frontend analogy:** CDC = WebSocket/subscription jaisa — DB "push" karta hai "ye row badli", cache react karta hai. Explicit invalidation = manually har mutation ke baad refetch. CDC = reactive, automatic.

---

## 3. The stale-cache reality (interview wisdom)

Perfect consistency + caching **saath me mushkil** hai (yaad karo DB lesson 14 — CAP-style trade-off). Practical approach:
- **Non-critical data** (profile, product description) → TTL kaafi (thoda stale ok)
- **Semi-critical** (inventory count display) → explicit invalidation on write
- **Critical + multi-writer** (pricing across services) → CDC/event-based
- **Never-stale-allowed** (account balance) → shayad **cache hi mat karo**, ya write-through (lesson 03)

**Golden line:** *"Choose your staleness budget per data type. Not everything needs to be fresh, and pretending it does kills your hit ratio."*

---

## Quick Self-Check
1. Eviction aur invalidation me core difference (space vs freshness)?
2. LRU kaise decide karta hai kya hataana? LFU se kaise alag?
3. Interview me eviction ke liye kya bolna chahiye aur kyu (LRU + TTL)?
4. TTL invalidation ka trade-off — chhota vs bada TTL?
5. Explicit invalidation accurate hai but iska risk kya (multi-writer)?
6. CDC kya hai, aur kab TTL/explicit se better hai?
7. "Staleness budget per data type" — account balance vs product description ke liye alag approach kyu?

---
**Next:** `05-Cache-Failure-Modes.md` — jab caching ULTA problem create kar de (stampede, hot keys).