# 02 — Caching Fundamentals & Cache Layers

> **Is lesson me:** Cache kya/kyu (full depth), cache hit/miss + hit ratio, aur wo **5 layers** jaha caches rehte hai (browser → CDN → app → distributed → database), CDN ke mechanics ke saath. DB lesson 14 me basics dekhe the — ye uska complete version hai.

Prerequisites: `01-Performance-Fundamentals.md`

---

## 1. Caching kya hai (recap + depth)

**Caching** = frequently-accessed data ko ek **faster storage layer** me store karna, taaki slower layer (database/disk) ko har baar na chhuna paDe.

**Analogy:** Apni sabse zyada use hone wali books desk pe rakho (cache) — library (database) tak baar-baar chalne ke bajaye. Desk chhota hai (limited space) but instant. Library huge hai but slow.

```
Without cache:
  Client → Server → Database (50ms)   ×3 requests = 150ms

With cache:
  Client → Server → Database (50ms) → cache me store
  Client → Server → Cache (1ms) ← HIT
  Client → Server → Cache (1ms) ← HIT
  = 52ms for 3 requests (~3x faster)
```

Kyu itna farak? Kyunki cache **RAM** me hai aur database **disk** pe — aur RAM ~1000x fast hai (lesson 01). Caching basically "disk ko RAM se bypass karna" hai.

---

## 2. Cache hit, miss, aur hit ratio

```
Request aati hai → data cache me hai?
   HIT  → cache se return (fast) ✅
   MISS → slow source (DB) se fetch → cache me daalo → return
```

- **Cache hit** = data cache me mil gaya (fast path)
- **Cache miss** = nahi mila, DB tak jaana pada (slow path)
- **Hit ratio** = hits / (hits + misses) — **the key metric**

```
Hit ratio 90% → 100 requests me 90 cache se (fast), sirf 10 DB tak
Hit ratio 99% → sirf 1 request DB tak → database ka load 100x kam!
```

**Target:** read-heavy systems me **90-99%**. Agar **80% se neeche** → ya toh TTL bahut chhota hai (data jaldi expire ho raha), ya access pattern bahut random hai (har request unique, caching help nahi karti). Tab caching reconsider karo.

**Frontend analogy:** Ye bilkul browser image cache jaisa — pehli baar network se (miss, slow), phir cache se (hit, instant). Hit ratio jitna high, network utna kam.

---

## 3. Caching ka fundamental trade-off

Cache free nahi — ek core tension hai:

```
FRESHNESS  ◄──────────────────► SPEED/LOAD
(hamesha latest)              (DB ko kam chhuo)
```

- Jitni der data cache me rakhoge → utna DB load kam (good) — but utna **stale** (purana) hone ka risk (bad).
- Jitni jaldi refresh karoge → utna fresh — but DB load zyada.

Ye tension poore caching module ka dil hai. Strategies (lesson 03), eviction/invalidation (lesson 04), aur failure modes (lesson 05) — sab isi trade-off ko manage karne ke tarike hai.

---

## 4. Cache kaha rehte hai — the 5 layers

Ek request client se database tak jaate waqt **5 jagah** cache ho sakti hai. Har layer pichli se aage ka kaam bachati hai:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: BROWSER CACHE          latency: 0ms                 │
│   Cache-Control, ETag, Last-Modified headers                 │
│   (user ke apne device pe — network hi nahi lagta)           │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: CDN CACHE              latency: 5-20ms              │
│   Cloudflare, CloudFront, Fastly (edge servers globally)     │
│   Static assets, images, videos — user ke paas               │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: APPLICATION CACHE      latency: <1ms                │
│   In-process HashMap / Guava / Caffeine (app ki apni RAM)    │
│   Har server ki local memory — no network                    │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: DISTRIBUTED CACHE      latency: 1-5ms  ⭐           │
│   Redis / Memcached (shared across all servers)              │
│   → HLD interviews me MOSTLY yahi discuss hota hai           │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: DATABASE CACHE         latency: varies              │
│   Query cache, buffer pool (DB ka apna RAM — lesson 04)      │
└─────────────────────────────────────────────────────────────┘
                          ▼
                     Database (disk)
```

**Interview me focus: Layer 4 (Redis/Memcached).** Kyunki wahi tum control karte ho aur wahi shared/scalable hai.

### Layer 3 vs Layer 4 — important distinction
- **Layer 3 (local/in-process):** har server ki apni RAM me. Super fast (no network), **but** har server ka apna alag copy → inconsistent ho sakte hai, aur server restart pe gone. Hot keys ke liye great (lesson 05).
- **Layer 4 (distributed):** ek shared Redis jise saare servers use karte hai. Consistent (ek source), survives server restart, **but** network hop lagta hai (1-5ms).

Real systems dono use karte hai: L3 for ultra-hot tiny data (short TTL), L4 for shared cache.

---

## 5. CDN — Layer 2 ko thoda samjho

**CDN (Content Delivery Network)** = duniya bhar me phaile **edge servers** ka network jo content ko **user ke geographically paas** cache karta hai.

**Problem:** Tumhara server US me hai, user India me. Har image request ko US tak jaana = ~200ms+ round trip (lesson 01 — cross-country network slow). CDN India me ek edge server pe wo image cache kar deta hai → user ko ~10ms me milti hai.

```
Bina CDN:  India user ──────────(200ms)──────────▶ US server
Ke saath:  India user ──(10ms)──▶ Mumbai edge (cached)
                                     │ (miss hua toh hi US jaata hai)
                                     ▼
                                  US origin server
```

**CDN kya cache karta hai:** static, rarely-changing content — images, videos, CSS/JS bundles, fonts. (Dynamic per-user data usually nahi.)

**Kaise control hota hai — HTTP headers:**
- `Cache-Control: max-age=3600` → "ise 1 ghante cache karo"
- `ETag: "abc123"` → content ka version-hash; browser/CDN poochta hai "kya ETag badla?" — nahi toh `304 Not Modified` (dobara download nahi)
- `Last-Modified` → kab last change hua

**Frontend analogy:** Ye tumne dekha hoga — Network tab me `304 Not Modified`, ya assets jo instantly load hote hai repeat visit pe. Wo Cache-Control/ETag hi hai. CDN same mechanism ko global scale pe le jaata hai.

> **Note:** CDN aksar apne alag topic hota hai (networking/delivery). Yahan bas itna samajh lo ki ye caching ka Layer 2 hai — same principle (data user ke paas rakho), bigger scale.

---

## 6. Cache kya store karta hai (practical)

Usually **key → value** (Redis string ya structure):
```
Key                          Value
─────────────────────────────────────────────
"user:123"                   {id:123, name:"Bob", ...}   (serialized JSON)
"product:456:price"          "₹1299"
"feed:user:123"              [post_ids...]  (Redis List)
"leaderboard:game7"          {...}          (Redis Sorted Set)
"session:abc"                {userId, expiry}
```

Key design tips:
- **Namespaced keys** — `user:123`, `product:456` (colon convention, readable + groupable)
- **Include version if needed** — `user:123:v2` (schema change pe purana cache auto-ignore)
- Value ideally chhota rakho (bade values network + memory eat karte hai)

---

## Quick Self-Check
1. Caching itna fast kyu hai — kaunsi hardware property (lesson 01 se)?
2. Cache hit / miss / hit-ratio — teeno define karo. Target hit ratio kya, aur 80% se neeche kya matlab?
3. Caching ka fundamental trade-off (freshness vs kya)?
4. 5 cache layers naam + latency? Interview me kaunsa focus hota hai?
5. Layer 3 (local) vs Layer 4 (distributed) — 2 differences?
6. CDN kya problem solve karta hai, aur ETag/`304` ka kya role?
7. Cache keys ka namespacing convention kya hota hai?

---
**Next:** `03-Caching-Strategies.md` — cache ko database ke saath kaise coordinate karo.