# 16 — Putting It All Together: The Database Design Playbook

> **Is lesson me:** Poori series ka nichoD — ek repeatable **playbook** jisse tum kisi bhi system ka database layer end-to-end design kar sako. Phir 3 complete worked examples (URL shortener, Instagram-style feed, ticket booking) jinme har lesson ka concept apply hota hai. Ye tumhara interview + real-work reference hai.

Prerequisites: **saare pichle lessons (01-15).** Ye sabko tie karta hai.

---

## 1. The Playbook — 8 steps (kisi bhi system ke liye)

Jab koi bole "design the database for X", ye order follow karo:

```
STEP 1 — Requirements & scale samjho
  • Kitna data? (GB/TB/PB) Kitne users? Reads:writes ratio?
  • Latency needs? Consistency needs (money vs likes)?

STEP 2 — Access patterns list karo  ⭐ (sabse important)
  • "Kaise query hoga?" — har read/write pattern likho
  • Ye decide karta hai SB kuch (store, schema, indexes)

STEP 3 — Data model + store(s) choose karo
  • Access patterns → SQL/NoSQL/hybrid (lesson 14 framework)
  • Polyglot: alag data ke liye alag store

STEP 4 — Schema design karo
  • Tables/collections, keys, relationships (lesson 02)
  • Normalize by default; denormalize for read-heavy (lesson 02/13)

STEP 5 — Indexes add karo
  • Har hot query ke WHERE/ORDER BY dekho (lesson 06)
  • Composite: equality-first, range-last

STEP 6 — Transactions & consistency decide karo
  • Money/inventory → atomic updates + constraints (lesson 08/09)
  • Isolation level + technique per invariant

STEP 7 — Scale plan
  • Reads high → replicas (lesson 12)
  • Writes/storage high → shard (lesson 13), shard key choose
  • Hot data → cache (lesson 14)

STEP 8 — Durability, files, edge cases
  • WAL/backups already there (lesson 10)
  • Files/media → object storage (lesson 15)
  • Failover, monitoring, lag alerts
```

**Golden principle (poori series ka):** *Access pattern pehle, store baad me. "How will I query this?" — ye ek sawaal 80% decisions drive karta hai.*

---

## 2. Worked Example A — URL Shortener (bit.ly)

**Requirements:** short code → long URL. Massive reads (redirects), fewer writes (new links). Low latency critical.

**Access patterns:**
```
1. WRITE: long URL → short code generate + store   (low volume)
2. READ:  short code → long URL lookup             (MASSIVE volume)
3. (optional) analytics: click counts per code
```

**Store choice:** Read-heavy, simple key-value lookup (`code → url`) — **perfect for key-value.**
- Primary: **DynamoDB** (`PK = short_code`) ya Postgres with index — O(1)/O(log N) lookup
- **Redis cache** in front — popular links (viral) cache se serve, DB tak pahunche hi na

**Schema:**
```
Table: urls
  short_code (PK)  |  long_url  |  created_at  |  click_count
  "aB3xK"          |  "https..."|  ...         |  0
```

**Short code generation:** counter → base62 encode (0-9,a-z,A-Z), ya hash(long_url) ka prefix. Collision check.

**Scale:**
- Reads: **Redis cache** (99% hit for popular) + **read replicas** (lesson 12) for the rest
- Writes: low, single primary kaafi. Agar huge → **shard by short_code hash** (lesson 11/13)
- Analytics: click_count ko atomic `INCR` (Redis) ya async event (Kafka) → batch to DB (hot-row contention avoid)

**Consistency:** Eventual is fine — naya link 1 second late dikhe toh koi disaster nahi (AP-leaning, lesson 14). Redirect stale nahi hota (codes immutable).

**Concept map:** key-value store (07,14) + caching (14) + replication (12) + hashing (11) + atomic counter (09).

---

## 3. Worked Example B — Instagram-style Photo Feed

**Requirements:** users photos upload karein, followers ke feed me dikhe. Read-heavy, media-heavy, huge scale.

**Access patterns:**
```
1. WRITE: user uploads photo (image bytes + metadata)
2. READ:  user ka feed (followed users ke recent photos)
3. READ:  ek photo + uske likes/comments
4. WRITE: like/comment (very high volume)
```

**Stores (polyglot — lesson 14):**
```
Photo bytes         → S3 + CDN (lesson 15) — presigned URL upload
Photo metadata      → Postgres/DynamoDB ({id, user_id, s3_key, caption, created_at})
Feed (timeline)     → Cassandra or Redis (write-heavy, AP)
Likes/comments      → Cassandra (massive writes) or Postgres (if moderate)
Cache (hot photos)  → Redis
Search (users/tags) → Elasticsearch
```

**Photo upload flow (lesson 15):**
```
1. Client → backend: "upload photo"
2. Backend: presigned S3 PUT URL + metadata row (status: uploading)
3. Client → S3 directly (multipart if large)
4. S3 event → processing (resize, thumbnails) → metadata status: ready
5. CDN serves the image globally
```

**Feed generation (classic design choice):**
- **Fan-out on write (push):** post karte hi followers ke feed lists me daal do (Redis lists). Read fast, write expensive (celebrity = millions of inserts — hot partition, lesson 13).
- **Fan-out on read (pull):** feed read pe followed users ke recent posts fetch + merge. Write cheap, read expensive.
- **Hybrid (real Instagram):** normal users push, celebrities pull. Best of both.

**Schema (metadata):**
```
photos:  id (PK) | user_id | s3_key | caption | created_at
follows: follower_id | followee_id      (many-to-many, lesson 02)
likes:   photo_id | user_id | created_at
```

**Indexes (lesson 06):** `photos (user_id, created_at DESC)` for "user's photos newest-first"; `follows (follower_id)` for "who I follow".

**Scale:** Shard photos by `user_id` (lesson 13) → ek user ka data ek shard. Replicate each shard (lesson 12). Cache hot photos + feeds in Redis. Media entirely on S3/CDN (DB stays small).

**Consistency:** Feed eventual (lesson 14) — naya photo few seconds late dikhe, ok. Like counts eventual (approximate ok). This is AP-leaning by design.

**Concept map:** object storage (15) + CDN + polyglot (14) + fan-out + sharding (13) + replication (12) + caching (14) + indexes (06) + many-to-many (02).

---

## 4. Worked Example C — Ticket/Seat Booking (correctness-critical)

**Requirements:** events ke seats book karein. **Double-booking NEVER.** Correctness > availability here.

**Access patterns:**
```
1. READ:  event ke available seats dikhao
2. WRITE: ek seat book karo (MUST be exactly-once, no double-book)
3. READ:  user ki bookings
```

**Store choice:** Correctness + transactions critical → **PostgreSQL (ACID, CP-leaning — lesson 08/14).** No eventual consistency here.

**Schema:**
```
seats:    id (PK) | event_id | seat_no | status (available/held/booked) | booked_by
bookings: id (PK) | user_id  | seat_id | created_at
```

**The core problem — write skew (lesson 09!):** 2 users last seat ek saath book kare. Naive "check available then book" = both succeed = double-book.

**Solution (lesson 09 techniques):**
```sql
-- Pessimistic lock on the specific seat (contention naturally low, per-seat)
BEGIN;
  SELECT * FROM seats WHERE id = 42 AND status = 'available' FOR UPDATE;
  -- agar row mili → lock mil gaya, koi aur yaha block
  UPDATE seats SET status = 'held', booked_by = :user WHERE id = 42;
COMMIT;
-- Phir short TTL hold (payment ke liye), timeout pe auto-release
```
Ya atomic conditional update (lesson 09):
```sql
UPDATE seats SET status='booked', booked_by=:user
WHERE id = 42 AND status = 'available';
-- 0 rows → already taken. Race impossible.
```

**Payment flow:** seat ko "held" karo (short TTL), payment process, success → "booked", timeout/fail → release. **Payment ke across DB lock mat hold karo** (lesson 05/08) — hold-with-TTL use karo.

**Scale:** Ticket booking usually single-DB-able (ek event ke seats limited). Agar massive (huge concurrent events) → shard by `event_id` (lesson 13) — ek event ek shard, transactions single-shard rehte hai (cross-shard txn avoid — lesson 13).

**Consistency:** Strong/CP — better to show "sorry, unavailable" (fail) than double-book (wrong data). Exactly CAP's CP choice (lesson 14).

**Concept map:** ACID (08) + write-skew + pessimistic lock (09) + CP (14) + sharding by event (13) + TTL holds.

---

## 5. The mental checklist (interview me bolne layak)

Jab bhi DB design karo, ye 6 sawaal khud se poocho:

```
1. ACCESS PATTERNS?    → "How will I query this?" (drives everything)
2. SQL or NoSQL?       → transactions/JOINs → SQL; scale/simple-access → NoSQL (14)
3. CONSISTENCY?        → money → strong/CP; social → eventual/AP (08,14)
4. SCALE READS?        → replicas + cache (12,14)
5. SCALE WRITES/DATA?  → shard, choose shard key (11,13)
6. FILES/MEDIA?        → object storage + CDN (15)
```

---

## 6. Full series concept map (ek nazar me)

```
FOUNDATIONS         What/why DB (01) → RDBMS/keys (02) → SQL (03) → query journey (04)
PRACTICAL STACK     drivers/ORM/pooling/envs (05)
FAST QUERIES        indexing (06) → complexity/Big-O (07)
CORRECTNESS         transactions/ACID (08) → isolation/MVCC/locks (09) → WAL/durability (10)
SCALE OUT           hashing (11) → replication (12) → sharding (13) → CAP/caching/choosing (14)
BEYOND RELATIONAL   object storage/S3 (15)
SYNTHESIS           this playbook (16)
```

**Har design decision inhi me se kisi principle pe based hoga.** Agar tum ye 16 lessons internalize kar lo, tum genuinely **kisi bhi system ka database layer design kar sakte ho** — chhote CRUD app se leke Instagram-scale tak.

---

## Quick Self-Check (capstone)
1. 8-step playbook bina dekhe list kar sakte ho? Step 1 aur 2 sabse pehle kyu?
2. URL shortener me caching + replication kyu, sharding kab?
3. Instagram feed me photo bytes kaha, metadata kaha — aur kyu (lesson 15 pattern)?
4. Fan-out on write vs read — trade-off, aur hybrid kyu?
5. Ticket booking me double-booking ka root cause (kaunsa anomaly)? Fix?
6. Ticket booking CP kyu, Instagram feed AP kyu — CAP terms me?
7. 6-sawaal mental checklist bolo. Kaunsa sawaal "everything drive karta hai"?

---

**🎉 Series complete!** Ab tum databases ke har layer samajhte ho — storage se scale tak, correctness se consistency tak. Real design karte waqt is playbook (16) + decision framework (14) ko reference ki tarah use karo.

Agla natural step: in concepts ko actual **High-Level Design (HLD)** problems pe apply karna (Key-Value Store, Chat System, Rate Limiter, News Feed) — jahan ye DB knowledge poore system architecture ka ek hissa banti hai. Wahan connection pooling (05) bhi naturally full-system context me use hoga.