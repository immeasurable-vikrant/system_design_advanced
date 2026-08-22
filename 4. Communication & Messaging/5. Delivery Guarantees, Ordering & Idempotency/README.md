# 05 — Delivery Guarantees, Ordering & Idempotency

> **Is lesson me:** Messages kitni baar deliver ho sakte hai (at-most/at-least/exactly-once), ordering kaise guarantee karo (partition key), aur **idempotency** — woh critical concept jo duplicate messages ko safe banata hai. Idempotency poore distributed systems ki reliability ka core hai (Topic 2 me hint tha, ab properly).

Prerequisites: `04-Message-Queues.md`

---

## 1. Delivery guarantees — 3 levels

Jab producer message bheje aur consumer padhe, network/crashes ke beech kitni baar deliver hoga?

| Guarantee | Matlab | Trade-off |
|---|---|---|
| **At-most-once** | Message kho sakta hai, but **kabhi 2 baar nahi** | Fastest, lossy |
| **At-least-once** | Message **guaranteed deliver**, but **duplicate ho sakta** | Most common (SQS, Kafka default) |
| **Exactly-once** | Bilkul **ek baar** | Hardest, expensive |

**At-least-once most systems ka standard hai.** Duplicates ko tum consumer side pe **idempotency** se handle karte ho (aage section 4).

### At-least-once me duplicate kyu hota hai
```
Producer message bheje → broker acknowledge kare
Consumer message padhe → process kare → acknowledge kare (commit offset)

Agar consumer PROCESS ke BAAD but ACKNOWLEDGE se PEHLE crash ho jaye:
  → broker ko laga process nahi hua → message DOBARA deliver karta hai
  → consumer wahi message PHIR dekhta hai (DUPLICATE!)
```
Isliye at-least-once me duplicates **inevitable** hai — aur isiliye idempotency zaroori hai.

> **Exactly-once itna mushkil kyu:** "process + acknowledge" ko truly atomic banana across network + crashes bahut hard hai (distributed systems ki fundamental problem). Kafka "exactly-once semantics" deta hai transactions se, but overhead ke saath. Practically: **at-least-once + idempotent consumer = effectively exactly-once**, cheaper.

---

## 2. Message Ordering

Kya messages usi order me process hote hai jis order me bheje? Depends.

### Kafka: partition ke andar ordered, across partitions nahi
```
Topic "orders" (3 partitions):
  Partition 0: [order_1, order_4, order_7]  ← is partition me ordered
  Partition 1: [order_2, order_5, order_8]  ← is partition me ordered
  Partition 2: [order_3, order_6, order_9]  ← is partition me ordered

But order_2 (P1) order_1 (P0) se pehle process ho sakta hai (alag partitions)
```

> **Partition kya hai (detail lesson 06):** ek Kafka topic ko multiple **partitions** me baanta jaata hai — har ek ek independent ordered log. Zyada partitions = zyada parallelism, but ordering sirf ek partition ke andar guaranteed.

### Ek specific entity ki ordering kaise guarantee karo — partition key ⭐
```
Us entity ki ID ko PARTITION KEY banao:
  hash("user_123") → hamesha Partition 1 → user_123 ke saare events ordered

Matlab: ek user/order/account ke saare messages same partition pe → order guaranteed
```

> **Recap (Topic 2 lesson 11):** ye wahi `hash(key) % N` consistent-hashing idea hai — same key hamesha same partition. Yahan ordering guarantee ke liye use ho raha hai.

**Frontend analogy:** socho tum events ko `userId` se group kar rahe ho taaki ek user ke actions sequence me rahe, chahe alag users parallel process ho. Same idea.

**SQS:** Standard = no ordering; FIFO = ordered within a "message group" (Kafka partition jaisa).

---

## 3. Idempotency — the critical concept ⭐⭐

At-least-once delivery duplicates deta hai. Toh consumer ko **duplicate-safe** banana padta hai. Iska tarika: **idempotency.**

**Idempotency** = ek operation ko **ek baar ya kai baar** karne ka **same result** ho. Duplicate execution se koi extra effect nahi.

**Examples:**
```
IDEMPOTENT:
  "Set balance = 500"        → 3 baar chalao, balance 500 hi (same result)
  "Mark order #123 as paid"  → 3 baar chalao, order paid hi (no double effect)
  DELETE user 5              → 3 baar chalao, user 5 gone (same)

NOT idempotent:
  "balance = balance - 100"  → 3 baar chalao, ₹300 kat gaye! (WRONG — duplicate ne damage kiya)
  "INSERT new order"         → 3 baar chalao, 3 duplicate orders!
```

**Frontend analogy (tumhe ye pata hai):** HTTP methods! **GET, PUT, DELETE idempotent hai** (baar-baar karo, same result). **POST idempotent NAHI** (baar-baar = multiple creates). Yehi wajah hai ki retry-safe APIs PUT prefer karti hai, aur POST ke liye special handling chahiye.

### Idempotency kaise achieve karo — idempotency key
Har message/request ko ek **unique ID** do. Consumer process karne se pehle check kare "ye ID already process ki?":
```
Consumer ko message mila (messageId = "abc-123"):
  1. Check: "abc-123" already processed? (Redis/DB me dekho)
     HAAN → skip (duplicate, kuch mat karo) ✅
     NAHI → process karo, phir "abc-123" ko "processed" mark karo
```
```python
def handle_message(msg):
    if redis.exists(f"processed:{msg.id}"):
        return  # duplicate, skip
    process(msg)                              # actual kaam
    redis.set(f"processed:{msg.id}", 1, ex=86400)  # mark done (TTL)
```

> **Recap (Topic 2 lesson 09):** double-spend rokne ke liye humne atomic `UPDATE ... WHERE balance >= x` aur compare-and-set dekha tha. Idempotency wahi philosophy hai — operation ko is tarike design karo ki repeat safe ho. Payment systems me idempotency key + ledger dono use hote hai.

**Real example (Stripe):** har payment request ke saath ek `Idempotency-Key` header. Agar network glitch se same request 2 baar aaye, Stripe pehle wala result return karta hai — customer ko **double charge nahi** hota. Ye idempotency ka textbook use hai.

---

## 4. Sab milke — the reliability recipe

```
At-least-once delivery (message kabhi lose nahi)
        +
Idempotent consumer (duplicate safe)
        =
Effectively exactly-once (reliable, without the cost of true exactly-once)
```
Ye **most production systems ka standard** hai. Interview me yahi bolo — "at-least-once + idempotency" — na ki "main exactly-once use karunga" (jo mushkil + mehenga hai).

---

## Quick Self-Check
1. 3 delivery guarantees — at-most/least/exactly-once? Kaunsa most common?
2. At-least-once me duplicate kyu inevitable hai (crash timing)?
3. Kafka me ordering kaha guaranteed hai (partition)? Ek entity ki ordering kaise (partition key)?
4. Partition key kis Topic-2 concept se relate karta hai?
5. **Idempotency** ki definition? Idempotent vs non-idempotent operation ka example?
6. HTTP methods me kaunse idempotent hai (GET/PUT/DELETE vs POST)?
7. Idempotency key se duplicate handling kaise hoti hai (consumer flow)?
8. "Effectively exactly-once" recipe kya hai?

---
**Next:** `06-Kafka-vs-SQS-vs-RabbitMQ.md` — kaunsa message broker kab.