# 10 — The Messaging Playbook (Putting It All Together)

> **Is lesson me:** Poore module ka synthesis — ek decision playbook (kab real-time, kab queue, kaunsa guarantee), phir 3 complete worked designs (chat system, news feed, notification system) jinme real-time + queues + fan-out + reliability sab ek saath kaam karte hai. Ye tumhara interview + real-work reference hai.

Prerequisites: **saare pichle lessons (01-09).**

---

## 1. The Communication Playbook — decision order

Jab koi system design karo, communication ke liye ye poocho:

```
STEP 1 — SYNC ya ASYNC? (lesson 01)
  User result ka wait kar raha? → sync (request-response)
  Background me ho sakta? → async (queue)

STEP 2 — CLIENT ko real-time updates chahiye? (lesson 02-03)
  One-way server push? → SSE
  Bidirectional high-rate? → WebSocket
  Infrequent? → polling
  Scale? → pub/sub backplane + connection registry + sticky sessions

STEP 3 — SERVICES ke beech messaging? (lesson 04-06)
  High throughput / replay / multiple consumers? → Kafka
  Simple managed tasks? → SQS
  Complex routing? → RabbitMQ
  Pattern: point-to-point (work split) ya pub-sub (broadcast)?

STEP 4 — RELIABILITY? (lesson 05, 07, 08)
  Delivery: at-least-once + idempotency (default)
  Ordering: partition key per entity
  Failures: retry + backoff + jitter → DLQ
  Producer too fast: back-pressure
  DB + event atomic: outbox pattern

STEP 5 — DISTRIBUTION at scale? (lesson 09)
  Feed/timeline? → hybrid fan-out (write for regular, read for celebrity)
  Notifications? → fan-out on write
```

**Golden principle:** *Decouple sender aur receiver. User jo wait kare wo sync; baaki sab async through queues. Reliability = at-least-once + idempotency, hamesha.*

---

## 2. Worked Design A — Chat System (WhatsApp/Slack)

**Requirements:** users real-time messages bhejein/receive karein, typing indicators, delivery/read receipts, offline delivery.

**Design:**
```
1. REAL-TIME (lesson 02-03):
   WebSocket — bidirectional (messages + typing + receipts). Client app khulne pe WS connect.

2. SCALING (lesson 03):
   Multiple WS servers behind LB (sticky sessions).
   Redis Pub/Sub backplane — User A (Server 1) → User B (Server 2) delivery.
   Connection registry (userId → serverId) in Redis.
   Heartbeats for dead-connection cleanup.

3. MESSAGE FLOW:
   User A message bheje (WS) → server DB me save (Topic 2) →
   Redis pub/sub se User B ke server ko → User B ki WS pe push.

4. OFFLINE DELIVERY (lesson 05):
   User B offline? → message queue/DB me store →
   B reconnect kare → last message ID se missed messages replay →
   at-least-once + client-side dedup (idempotency).

5. RELIABILITY:
   Message persistence in DB (durable). Delivery receipts via WS.
   Group chat = fan-out (lesson 09) to group members.
```
**Concepts:** WebSocket (02) + pub/sub scaling (03) + idempotency (05) + fan-out for groups (09) + DB persistence (Topic 2).

---

## 3. Worked Design B — News Feed (Twitter/Instagram)

**Requirements:** users post karein, followers ke feed me dikhe, huge scale, celebrities.

**Design:**
```
1. WRITE PATH (lesson 04, 09):
   User post → Post Service → Post DB (Topic 2) → Kafka "post_events"
   (Kafka write ko fan-out se decouple karta hai — async, load leveling)

2. FAN-OUT (lesson 09 — hybrid):
   Fan-out workers Kafka se consume karein:
     Regular user (<10K followers) → push to follower timelines (Redis sorted sets)
     Celebrity (>10K) → skip, store in own post list

3. READ PATH (lesson 09):
   Feed Service → pre-computed timeline (cache) + celebrity posts (pull) → merge + rank → top 50

4. REAL-TIME updates (lesson 02):
   New posts ka live indicator → SSE (one-way server push, simple)

5. RELIABILITY (lesson 05, 07):
   Kafka at-least-once + idempotent fan-out workers (duplicate post na dikhe).
   Failed fan-out → retry + DLQ.
```
**Concepts:** Kafka (06) + consumer groups (04) + hybrid fan-out (09) + idempotency (05) + DLQ (07) + Redis/caching (Topic 3) + sharding (Topic 2).

---

## 4. Worked Design C — Notification System

**Requirements:** events (new follower, like, comment, order shipped) → user ko email/SMS/push, reliably.

**Design:**
```
1. EVENT INGESTION (lesson 04, 08):
   Koi bhi service event produce kare → Kafka "notification_events"
   (outbox pattern se — DB write + event atomic, lesson 08)

2. FAN-OUT (lesson 04 — pub-sub):
   Ek event → multiple channels (consumer groups):
     Email Service, SMS Service, Push Service — har ek independently consume

3. DELIVERY (lesson 09 — fan-out on write):
   In-app notification inbox → push to recipient (pre-computed, instant load)

4. RELIABILITY (lesson 05, 07):
   at-least-once + idempotency (user ko duplicate notification na jaaye!)
   Failed sends → retry + backoff → DLQ → ops review

5. BACK-PRESSURE (lesson 08):
   Email provider rate-limited? → back-pressure / rate limit producers
   Spike (viral post = millions of likes)? → queue absorbs (load leveling)
```
**Concepts:** Kafka pub-sub (04) + consumer groups (04) + outbox (08) + idempotency (05) + retry/DLQ (07) + back-pressure (08) + fan-out on write (09).

---

## 5. The component cheat-sheet

| Need | Tool | Lesson |
|---|---|---|
| User waiting for result | Synchronous request-response | 01 |
| One-way live updates | SSE | 02 |
| Bidirectional real-time | WebSocket | 02-03 |
| Cross-server WS delivery | Redis Pub/Sub backplane | 03 |
| Decouple services | Message queue | 04 |
| Broadcast event | Pub-sub / consumer groups | 04 |
| High throughput + replay | Kafka | 06 |
| Simple managed queue | SQS | 06 |
| No message loss | At-least-once | 05 |
| Duplicate-safe | Idempotency | 05 |
| Ordering per entity | Partition key | 05 |
| Transient failure | Retry + backoff + jitter | 07 |
| Permanent failure | DLQ | 07 |
| Producer too fast | Back-pressure | 08 |
| DB + event atomic | Outbox pattern | 08 |
| DB changes as events | CDC | 08 |
| Feed distribution | Hybrid fan-out | 09 |

---

## 6. Full module concept map

```
MENTAL MODEL       sync vs async (01)
CLIENT REAL-TIME   polling/SSE/WebSocket (02) → scaling WS (03)
SERVICE MESSAGING  message queues (04) → guarantees/idempotency (05) →
                   Kafka/SQS/RabbitMQ (06) → retries/DLQ (07) → reliability (08)
DISTRIBUTION       fan-out patterns (09)
SYNTHESIS          this playbook (10)
```

**Aur saare 4 topics milke:**
```
Topic 1 (Core Infra)      → request ka safar: network, LB, proxy, CDN, gateway, pool
Topic 2 (Database)        → data: RDBMS, indexing, transactions, replication, sharding
Topic 3 (Caching/Perf)    → speed: caching, rate limiting, bloom filters
Topic 4 (Comm/Messaging)  → communication: real-time, queues, fan-out, reliability
```
Ek complete system: Topic 1 ke infra pe requests aati hai → Topic 4 ke messaging se services decouple hoti hai → Topic 3 ke cache se fast serve hoti hai → Topic 2 ke database me data rehta hai. **Chaaron ek saath ek production system banate hai.**

---

## Quick Self-Check (capstone)
1. Communication playbook ke 5 steps — sync/async se distribution tak?
2. Chat system me WebSocket + pub/sub + idempotency kaise fit hote hai?
3. News feed me Kafka write ko fan-out se kyu decouple karta hai?
4. Notification system me outbox + consumer groups + idempotency ka role?
5. Cheat-sheet se: "duplicate-safe", "DB+event atomic", "feed distribution" — kaunse tools?
6. Chaaron topics ek request/system me kaise flow karte hai?

---

**🎉 Topic 4 complete!** Ab tum system ke components ke beech communication design kar sakte ho — client real-time (chat, live updates) se leke service-to-service async messaging (queues, fan-out) tak, reliability patterns (idempotency, retry, DLQ, outbox) ke saath.

**Chaaron topics ab complete hai:**
- **Topic 1 — Core Infrastructure** → request ka safar + scaling
- **Topic 2 — Database & Storage** → data layer
- **Topic 3 — Caching & Performance** → speed + protection
- **Topic 4 — Communication & Messaging** → ye — real-time + async + fan-out

Ab tumhare paas ek **complete system design toolkit** hai — infrastructure, data, performance, aur communication. Agla natural step: in sabko actual **end-to-end HLD problems** (Design WhatsApp, Design Twitter, Design Uber, Design a Rate Limiter) pe apply karna, jahan chaaron topics ke pieces ek complete architecture me fit hote hai. Wahan tum ek blank whiteboard se pura system nikaal paoge.