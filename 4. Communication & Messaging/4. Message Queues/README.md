# 04 — Message Queues (Async Service Messaging)

> **Is lesson me:** Message queue kya hai, producer/consumer model, kyu use karte hai (decoupling, async, load leveling, reliability, fan-out), aur 3 messaging patterns — point-to-point, pub-sub, consumer groups. Ye service-to-service async communication ka core hai (lesson 01 ka async idea, ab concrete).

Prerequisites: `01-Sync-vs-Async-Communication.md`

---

## 1. Message queue kya hai

**Message queue** = ek buffer jo **producer** (sender) aur **consumer** (receiver) ke beech baithta hai. Producer messages queue me daalta hai. Consumer unhe **apni pace** pe padhta hai.

**Analogy:** Ek mailbox. Tum (producer) letter daal dete ho. Recipient (consumer) jab free ho tab uthata hai. Tumhe unke ghar hone ka wait nahi karna. Mailbox (queue) sender ko receiver se **decouple** karti hai.

```
Producer ──▶ [ Message Queue ] ──▶ Consumer
  (bhejta)      (buffer/store)      (padhta, apni pace)
```

> **Recap (lesson 01):** ye woh asynchronous, loosely-coupled communication hai. Producer message drop karke aage badh jaata hai (non-blocking); consumer ready hone pe process karta hai.

---

## 2. Kyu use karte hai

| Benefit | Example |
|---|---|
| **Decoupling** | Payment Service ko Email/SMS/Push Service ke baare me jaanne ki zarurat nahi |
| **Async processing** | User video upload → response turant → encoding background me |
| **Load leveling** | 10K orders ka burst → queue absorb kare → workers steady 1K/sec process kare |
| **Reliability** | Consumer crash ho jaye → message queue me safe → no data loss |
| **Fan-out** | Ek event → kai services independently consume kare (lesson 09) |

### Load leveling ko samjho (super important)
```
Bina queue: 100K requests/sec spike → DB directly hit → DB crash 💥
Queue ke saath: 100K spike → queue buffer kare → workers steady 10K/sec process
              → DB kabhi overwhelmed nahi
```
Queue ek **shock absorber** hai — spikes ko smooth karta hai. (Ye Topic 3 ke rate limiting se related hai — dono system ko overload se bachate hai, alag tarike se.)

**Frontend analogy:** ye ek debounce/throttle + task queue jaisa hai. Bahut saare events aaye toh unhe queue karke steady rate pe process karo, sab ek saath fire karke system girane ke bajaye.

---

## 3. Pattern 1 — Point-to-Point (Queue)

Ek message **exactly ek** consumer consume karta hai. Process hone ke baad message hat jaata hai.
```
Producer → [Queue] → Consumer A (message 1 mila)
                   → Consumer B (message 2 mila)
                   → Consumer C (message 3 mila)

Har message exactly ek consumer process karta hai (kaam baant liya)
```
**Use case:** task processing, job scheduling, order processing. Kaam ko multiple workers me distribute karna.
**Example:** SQS, RabbitMQ (default mode).

**Analogy:** ek task list jise 3 workers share karte hai — har task ek hi worker uthata hai (do baar nahi hota).

---

## 4. Pattern 2 — Pub-Sub (Topic)

Ek message **saare** subscribers tak pahunchta hai. Har subscriber ko ek **copy** milti hai.
```
Publisher → [Topic] → Subscriber A (SAARE messages milte hai)
                    → Subscriber B (SAARE messages milte hai)
                    → Subscriber C (SAARE messages milte hai)
```
**Use case:** event broadcasting. "Order placed" → inventory, analytics, email — sab ko ek saath notify karo.
**Example:** Kafka topics, SNS, Redis Pub/Sub (jo lesson 03 me WebSocket scaling me use kiya).

**Analogy:** ek newsletter — har subscriber ko same edition ki apni copy milti hai.

**Point-to-point vs pub-sub ka core:**
```
Point-to-point: ek message → ek consumer   (kaam baantna)
Pub-sub:        ek message → sab consumers  (event broadcast)
```

---

## 5. Pattern 3 — Consumer Groups (best of both) ⭐

Messages ek **group ke andar** consumers me distribute hote hai (point-to-point WITHIN group), but **multiple groups** ko sab messages milte hai (pub-sub ACROSS groups).
```
Topic: "order_events"
  → Consumer Group "notification-service": messages 3 workers me split
  → Consumer Group "analytics-service": same messages 2 workers me split
  → Consumer Group "inventory-service": single worker sab leta hai

Har GROUP ko SAARE messages dikhte hai (pub-sub across groups),
but ek group ke ANDAR, har message ek hi worker ko jaata hai (point-to-point).
```

Ye **Kafka ka core model** hai — aur system design interviews me sabse common pattern.

**Kyu powerful:** har team (service) ek consumer group hai — sabko poora event stream dikhta hai (independence), but har team apne andar workers add karke parallelism badha sakti hai (scaling). Best of both worlds.

**Frontend analogy:** socho ek shared event stream jise 3 alag features (notification, analytics, inventory) subscribe karte hai — har feature ko saare events dikhte hai, but har feature apne andar kaam ko multiple handlers me baant sakta hai.

---

## 6. Common patterns (quick preview)

Message queues ke 4 classic uses (detail capstone lesson 10 me):
```
1. ASYNC PROCESSING:  User action → API turant respond → queue → worker background
2. EVENT FAN-OUT:     Order placed → topic → notification + inventory + analytics + fraud
3. WORK DISTRIBUTION: 10K emails → queue → 20 workers parallel
4. RATE SMOOTHING:    Flash sale spike → queue buffer → workers steady rate (DB bachao)
```

---

## Quick Self-Check
1. Message queue kya hai — mailbox analogy se producer/consumer?
2. Ye lesson 01 ke sync/async se kaise connect hota hai?
3. Message queue ke 5 benefits? "Load leveling" (shock absorber) kaise kaam karta hai?
4. Point-to-point vs pub-sub — core difference (ek message kitne consumers)?
5. Consumer groups "best of both" kaise hai (within vs across group)?
6. Consumer groups kis system (Kafka) ka core model hai, aur kyu powerful?
7. Message queues ke 4 common patterns kya hai?

---
**Next:** `05-Delivery-Guarantees-And-Idempotency.md` — messages kitni baar deliver hote hai, aur duplicates handle karna.