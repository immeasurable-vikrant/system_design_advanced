# 08 — Reliability Patterns: Back-Pressure & Outbox

> **Is lesson me:** Do critical reliability patterns — **back-pressure** (jab producer consumer se tez ho) aur **outbox pattern** (dual-write problem — DB aur queue dono me consistently likhna), plus CDC. Ye woh patterns hai jo messaging ko production-grade banate hai. Outbox ek gap tha jo Topic 2 ne point kiya tha ("publish event with DB write → outbox pattern").

Prerequisites: `07-Retries-Backoff-And-DLQ.md`, Topic 2 lesson 08-09 (transactions)

---

## 1. Back-Pressure — jab producer consumer se tez ho

**Problem:** Producer messages consumer se **tezi se** generate karta hai. Queue unbounded grow karti hai → memory/disk khatam → crash.
```
Producer: 10,000 msg/sec  ─────▶  [Queue growing...]  ─────▶  Consumer: 1,000 msg/sec
                                    (9,000/sec jama ho rahe → infinite growth 💥)
```

**Back-pressure** = system ka wo mechanism jo producer ko "dheere!" signal deta hai jab consumer keep up na kar paaye.

**Solutions:**
| Strategy | Kaise |
|---|---|
| **Bounded queue + reject** | Queue ka max size. Full → producer ko error (back-pressure signal) |
| **Rate limit producers** | Producer sirf N msg/sec publish kar sake (Topic 3 rate limiting!) |
| **Auto-scale consumers** | Queue depth badhe → aur consumer instances add (Topic 1 auto-scaling) |
| **Alert on queue depth** | Threshold cross → alarm → manual intervention |

**Frontend analogy:** ye TCP flow control (Topic 1 lesson 01) jaisa hai — receiver sender ko batata hai "mera buffer bhar raha, dheere bhejo". Ya React me — agar tum events consumer se tez fire karo, tumhe debounce/throttle chahiye. Back-pressure wahi backend-scale pe.

> **Connect:** back-pressure + auto-scaling + rate limiting teeno milke queue ko healthy rakhte hai — Topic 1 (scaling), Topic 3 (rate limiting), aur ye module ek saath.

---

## 2. The Dual-Write Problem (kyu outbox chahiye)

Ye ek subtle-but-critical bug hai jo har messaging system me aata hai. Scenario: ek order place hone pe tumhe **DB me order save** karna hai **aur** Kafka pe "order_placed" event **publish** karna hai.

```python
# NAIVE (buggy!) approach:
def place_order(order):
    db.save(order)              # Step 1: DB me save
    kafka.publish("order_placed", order)   # Step 2: event publish
```

**Problem — ye do alag systems hai, ek transaction me nahi:**
```
Case 1: db.save() success, phir CRASH before kafka.publish()
  → Order DB me hai, but event kabhi nahi gaya
  → Inventory/email/analytics ko pata hi nahi chala! (lost event)

Case 2: db.save() success, kafka.publish() success, but DB transaction rollback
  → Event chala gaya, but order actually save nahi hua! (phantom event)
```

Do alag systems (DB + Kafka) ko atomically update karna **impossible** hai without a pattern — ye distributed systems ki classic "dual-write problem" hai.

> **Recap (Topic 2 lesson 08-09):** ek single DB me transaction atomic hoti hai (ACID). But DB + Kafka **do alag systems** hai — koi shared transaction nahi. Isliye ek special pattern chahiye.

---

## 3. Outbox Pattern — the solution ⭐

Idea: event ko bhi **usi database transaction** me likho jisme business data likhte ho (ek "outbox" table me). Phir ek alag process outbox se events padh ke Kafka pe publish kare.

```
┌─ SINGLE DB TRANSACTION (atomic — Topic 2) ─┐
│  1. orders table me order INSERT             │
│  2. outbox table me "order_placed" event INSERT │
│  COMMIT (dono saath, ya dono nahi)           │
└──────────────────────────────────────────────┘
                    │
                    ▼ (baad me, alag process)
        Outbox Relay: outbox table poll/read kare
                    │
                    ▼
             Kafka pe publish kare → outbox row ko "sent" mark kare
```

**Kyu ye kaam karta hai:**
- Order aur event **ek hi DB transaction** me likhe → dono save honge ya dono nahi (atomic — no lost/phantom events).
- Relay process **at-least-once** publish karega (crash ho toh unsent rows dobara publish — duplicate possible, but idempotency se safe — lesson 05).

```
Table: outbox
  id | event_type    | payload        | status | created_at
  1  | order_placed  | {order data}   | sent   | ...
  2  | order_placed  | {order data}   | pending| ...   ← relay ise publish karega
```

**Frontend analogy:** socho tum ek action aur uska "side-effect intent" ek saath local state me commit karte ho (atomic), phir ek separate effect uss intent ko process karta hai. Redux me action + a queued side-effect. Outbox wahi DB-level pe — intent ko data ke saath durably store karo, phir alag se dispatch.

> **Interview line:** *"DB write ke saath event publish karne ke liye main 2PC nahi, outbox pattern use karunga — event ko usi transaction me ek outbox table me likho, phir ek relay (ya CDC) usse Kafka pe publish kare. Ye dual-write problem ko atomically solve karta hai, at-least-once semantics ke saath."* (Topic 2 lesson 09 me humne yahi bola tha — "outbox pattern, not XA/2PC".)

---

## 4. CDC (Change Data Capture) — outbox ka auto version

Outbox relay ko manually poll karne ke bajaye, **CDC** database ke changes ko automatically stream kar deta hai.

**CDC** = database ke **changes** (inserts/updates/deletes) ko ek event stream ki tarah capture karna — usually database ke **WAL/replication log** ko read karke.

```
DB me koi change → CDC tool (Debezium) WAL padhta hai → Kafka pe event publish
```

> **Recap (Topic 2 lesson 10 — WAL):** yaad hai WAL har change ka ordered log hai, aur replication WAL ko replicas tak stream karti hai? CDC wahi WAL ko read karke changes ko Kafka events me convert karta hai. **Outbox + CDC combo:** outbox table me event likho, CDC us table ke inserts ko automatically Kafka pe stream kare — no manual relay polling.

**CDC use cases:**
- **Cache invalidation** (Topic 3 lesson 04!) — DB change → CDC event → cache update/delete
- **Search index sync** — DB change → Elasticsearch update
- **Data replication** to analytics/data lake
- **Event-driven microservices** — ek service ka DB change doosri service ka trigger

**Tools:** Debezium (most popular, Kafka ke saath), AWS DMS, Maxwell.

---

## 5. Reliability patterns — the full toolkit

```
Message lose na ho          → at-least-once delivery (lesson 05)
Duplicate safe              → idempotency (lesson 05)
Transient failure           → retry + backoff + jitter (lesson 07)
Permanent failure           → DLQ (lesson 07)
Producer too fast           → back-pressure (ye lesson)
DB + event atomically       → outbox pattern (ye lesson)
DB changes as events        → CDC (ye lesson)
```
Ye saare patterns milke ek messaging system ko **production-grade reliable** banate hai. Interview me inme se relevant ones mention karna = maturity dikhata hai.

---

## Quick Self-Check
1. Back-pressure kya problem solve karta hai (producer > consumer)?
2. Back-pressure ke 4 solutions? Ye kis Topic-1/Topic-3 concept se relate karte hai?
3. Dual-write problem kya hai — DB save + Kafka publish me kya galat ho sakta (2 cases)?
4. Ye Topic 2 ke transactions se kaise connect (do alag systems, no shared transaction)?
5. **Outbox pattern** kaise dual-write solve karta hai (single transaction)?
6. Outbox at-least-once kyu hai, aur duplicate kaise safe (idempotency)?
7. CDC kya hai, aur WAL (Topic 2) se kaise relate karta hai?
8. CDC ka ek use case jo Topic 3 (caching) se connect karta hai?

---
**Next:** `09-Fan-Out-Patterns.md` — ek event ko millions tak pahunchana (feeds).