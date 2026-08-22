# 09 — Fan-Out Patterns (Feed Distribution at Scale)

> **Is lesson me:** Fan-out kya hai, fan-out on write (push) vs read (pull) vs hybrid, celebrity problem, aur news feed ka complete architecture. Topic 2 (Instagram) + Topic 3 (news feed) me chhua tha; ye full depth hai with storage math aur real-world approaches.

Prerequisites: `04-Message-Queues.md`, Topic 3 (caching, Redis sorted sets)

---

## 1. Fan-out kya hai

**Fan-out** = ek source se **many recipients** tak data distribute karna. Jab ek user content post kare, wo uske followers ke feeds me kaise dikhe? Wo distribution mechanism "fan-out" hai.

**Analogy — newspaper publisher:**
- **Fan-out on write (push):** publisher har subscriber ke liye ek copy print karke unke mailbox me pahuncha deta hai har subah. Publish hote hi delivery. But 10M subscribers = printing/delivery me time + resources.
- **Fan-out on read (pull):** koi delivery nahi. Har reader newsstand pe jaake apni copy uthata hai jab padhna ho. Publisher upfront kuch nahi karta, but har reader har read pe kaam karta hai.

```
User (1000 followers) ek tweet post kare:
  Fan-out on write: 1000 follower timelines me turant likho
  Fan-out on read:  abhi kuch mat karo, jab har follower app khole tab compute karo
```

---

## 2. Fan-out on Write (Push Model)

Jab user content publish kare, **turant har follower ke feed me likho.**
```
User A (1000 followers) "Hello World" post kare:
  1. Fan-out worker follower list fetch kare: [B, C, D, ... 1000]
  2. Har follower ke pre-computed timeline me push kare:
     Redis: ZADD timeline:B <timestamp> <tweet_id>
     Redis: ZADD timeline:C <timestamp> <tweet_id>
     ... (1000 writes)
  3. Follower B app khole:
     Redis: ZREVRANGE timeline:B 0 49 → instant! Already computed.
```

> **Recap (Topic 3):** timeline ek **Redis Sorted Set** hai (score = timestamp, value = tweet_id). Sorted set O(log N) insert + O(log N + K) range deta hai — leaderboards/feeds ke liye perfect. Last ~800 tweets rakho, baaki trim.

**Pros:** read **extremely fast** (pre-computed, bas cache fetch), consistent UX (instant feed), simple read path.
**Cons:** **write amplification** (1 post → N writes), inactive users ke liye wasted work, **celebrity problem** (50M followers = 50M writes!), higher storage (har follower ko copy).

---

## 3. Fan-out on Read (Pull Model)

Write time pe kuch mat karo. Jab user feed khole, tab uske saare followed users ke posts fetch + merge karo.
```
User B feed khole:
  1. B ki following list fetch: [A, X, Y, ... 500 users]
  2. Har followed user ke recent posts fetch (500 queries!)
  3. Sab merge, timestamp se sort
  4. Top 50 return

  Latency: 500 users × 1ms = 500ms (too slow!)
           Parallel fetches se optimize: ~50-100ms
```

**Pros:** write fast (bas ek baar store), no wasted work (sirf jab read ho), **celebrity naturally handled** (no fan-out), less storage.
**Cons:** read **slow** (real-time me many sources aggregate), inconsistent latency (following count pe depend), complex merge (dedup, ranking).

---

## 4. Hybrid (jo Twitter/Instagram actually use karte hai) ⭐

Dono combine karo: regular users ke liye **fan-out on write**, celebrities ke liye **fan-out on read**.
```
User post kare:
  Celebrity? (> 10K followers)
    NAHI → Fan-out on WRITE: sab follower timelines me push
    HAAN → Store in celebrity's OWN post list only (NO fan-out)

User feed read kare:
  1. Pre-computed timeline fetch (cache se — regular users ke posts)
  2. Followed celebrities ke recent posts fetch (pull)
  3. Merge + rank + return top N
```

**Example:**
```
User B follows: [Alice (200 followers), Bob (500), Taylor (50M celebrity)]
  Alice posts → fan-out on write → B ke timeline me push (200 writes, cheap)
  Taylor posts → NO fan-out → Taylor ki post list me store
  B feed khole → pre-computed timeline (Alice, Bob) + Taylor ki recent posts (pull) → merge
```

Best of both: regular users ke liye fast reads (pre-computed), celebrities ke liye no write-explosion.

---

## 5. The Celebrity Problem (hot partition) ⭐

Ye woh challenge hai jo fan-out ko interesting banata hai:
```
Regular user (1000 followers):
  Post → 1000 timeline writes → ~1 second, cheap ✅

Celebrity (50M followers):
  Post → 50M timeline writes → MINUTES lagte hai!
  Problem 1: massive write amplification
  Problem 2: followers tweet minutes late dekhte hai
  Problem 3: ek tweet enormous system resources kha jaata hai 💥
```

> **Recap (Topic 2 lesson 13 — sharding):** ye "hot partition" problem hai — ek key (celebrity) disproportionate load leta hai. Wahan solutions the salting, dedicated shards; yahan solution hai **hybrid model** (celebrity ko fan-out se skip karo).

**Solutions:**
| Approach | Kaise | Trade-off |
|---|---|---|
| **Hybrid model** | Celebrity fan-out skip, read time merge | Thoda slower reads |
| Tiered fan-out | Active followers pehle, inactive baad me | Complexity |
| Selective fan-out | Sirf online/recently-active users ko | Kuch users miss |
| Async with priority | Celebrity posts ko fan-out queue me low priority | Eventual delivery |

---

## 6. News feed architecture (complete)

Yahan message queue (Kafka), workers, aur cache sab aate hai:
```
WRITE PATH:
  Client → API GW → Post Service → Post DB
                        │
                        ▼
                    Kafka ("post_events")   ← decouples write from fan-out (lesson 04)
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       Fan-Out Worker 1    Fan-Out Worker 2   ← parallel workers (regular users)
              │                   │
              └─────────┬─────────┘
                        ▼
                 Timeline Cache (Redis Sorted Sets)

READ PATH:
  Client → API GW → Feed Service
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       Timeline Cache       Celebrity Posts (pull)
       (pre-computed)             │
              └─────────┬─────────┘
                        ▼
                 Merge + Rank + Return top 50
```

**Kafka ka role:** write ko fan-out se decouple karta hai (lesson 04) — post save turant, fan-out background workers me. Spike absorb (load leveling).

---

## 7. Storage math (interview me poocha jaata hai)

```
Sirf tweet IDs store karo (full tweets nahi):
  Har entry: 8 bytes (tweet_id) + 8 bytes (timestamp) = 16 bytes
  Last 800 tweets per user
  200M DAU × 800 × 16B = ~2.5 TB in Redis
  (expensive but feasible with Redis Cluster — Topic 3 lesson 06)
```
Full tweet content DB/cache me alag rehta hai; timeline me sirf **references** (IDs). Ye woh "metadata vs blob" pattern hai (Topic 2 lesson 15 — S3).

---

## 8. Comparison + when to use

| Feature | Fan-Out Write | Fan-Out Read | Hybrid |
|---|---|---|---|
| Write speed | Slow (N writes) | Fast (1 write) | Medium |
| Read speed | Very fast | Slow | Fast |
| Storage | High | Low | Medium |
| Celebrity | Terrible (50M writes) | Good | Good |
| Best for | Most social feeds | Celebrity-heavy | Production systems |

**Real-world:** Twitter (hybrid — write for <10K, pull for celebrities), Instagram (hybrid + ML ranking at read), Facebook (mostly read — complex ranking), LinkedIn (write — smaller network), TikTok (read — algorithm-driven, not follow-based).

> **Notifications ke liye:** fan-out on **write** (push). Notifications time-sensitive + infrequent per user. Event trigger (new follower/like/comment) → turant recipient ke notification inbox me likho → user jab khole instant load.

---

## Quick Self-Check
1. Fan-out kya hai — newspaper push vs pull analogy?
2. Fan-out on write ke pros/cons? "Write amplification" kya?
3. Fan-out on read ke pros/cons? Celebrity naturally kyu handle hota hai?
4. Hybrid model kaise kaam karta hai (regular vs celebrity)?
5. Celebrity problem kya hai, aur ye Topic 2 ke kis concept (hot partition) se relate?
6. Timeline Redis me kaise store hoti hai (sorted set, IDs only)? Storage math?
7. News feed me Kafka ka role kya (decoupling)?
8. Notification system ke liye push ya pull, aur kyu?

---
**Next:** `10-Messaging-Playbook.md` — sab combine, ek chat + feed + notification system.