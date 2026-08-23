# 05 — Event-Driven Architecture

> **Is lesson me:** Events ko system ka backbone banana — event-driven architecture kya hai, commands vs events, choreography vs orchestration, aur trade-offs. Topic 4 me messaging tools (Kafka, pub-sub) dekhe; yahan wo ek **architecture style** ban jaate hai jo services ko deeply decouple karta hai.

Prerequisites: `01-Monolith-vs-Microservices.md`, Topic 4 (message queues, pub-sub)

---

## 1. Event-driven architecture kya hai

**Event-Driven Architecture (EDA)** = services **events** produce aur consume karke communicate karte hai, directly ek dusre ko call karne ke bajaye. Ek service kuch hone pe event emit karti hai; interested services react karti hai — bina producer ko jaane ki kaun sun raha hai.

```
REQUEST-DRIVEN (services directly call):
  Order Service → calls → Payment Service
                → calls → Inventory Service
                → calls → Email Service
  (Order Service sabko jaanti hai, sabka wait karti — tight coupling)

EVENT-DRIVEN (services react to events):
  Order Service → emits "OrderPlaced" event → [Event Bus / Kafka]
                                                    │
              ┌─────────────────┬───────────────────┤
              ▼                 ▼                   ▼
        Payment Service   Inventory Service   Email Service
        (independently react, Order Service inhe jaanti bhi nahi)
```

> **Recap (Topic 4 lesson 01, 04):** ye async, loosely-coupled messaging ka architectural application hai. Order Service bas "OrderPlaced" event drop karti hai — kaun consume karta hai, kitne consumers hai, wo react karne me kitna time lete hai — Order Service ko koi farak nahi. Ultimate decoupling.

---

## 2. Commands vs Events (crucial distinction) ⭐

- **Command** = ek specific service ko "ye karo" ka instruction. Ek recipient, expects action. Imperative.
  ```
  "ProcessPayment" → Payment Service ko (ye karo)
  ```
- **Event** = "ye ho gaya" ka announcement. Kai listeners, past-tense fact. Koi specific recipient nahi.
  ```
  "OrderPlaced" → jo bhi interested ho (ho chuka, react karo)
  ```

```
Command: directed, imperative, one handler   → "CreateOrder", "ChargeCard"
Event:   broadcast, past-tense, many listeners → "OrderCreated", "CardCharged"
```

**Frontend analogy:** command = ek function call (`processPayment()` — ek specific cheez karo). Event = a DOM event / EventEmitter (`emit('orderPlaced')` — jo bhi listen kar raha react kare). EDA me tum events prefer karte ho for decoupling.

---

## 3. Choreography vs Orchestration ⭐ (the big EDA decision)

Jab ek business process me multiple services involved ho (order → payment → inventory → shipping), unhe coordinate 2 tarike se kar sakte ho:

### Choreography (decentralized) — "har koi apna part jaanta hai"
Koi central coordinator nahi. Har service events sunti hai aur react karti hai, apne events emit karti hai. Process **emerge** hota hai.
```
Order Service:     "OrderPlaced" emit
Payment Service:   "OrderPlaced" sune → charge → "PaymentCompleted" emit
Inventory Service: "PaymentCompleted" sune → reserve → "InventoryReserved" emit
Shipping Service:  "InventoryReserved" sune → ship → "OrderShipped" emit
(koi ek service poora flow control nahi karti — chain of reactions)
```
**Pros:** loosely coupled, no single point of failure/bottleneck, easy naye service add (bas relevant event subscribe karo).
**Cons:** poora flow **kahi likha nahi** — samajhna mushkil ("kaun sun raha kaun?"), debugging hard (distributed), cyclic dependencies ka risk.

### Orchestration (centralized) — "ek conductor sabko direct karta"
Ek central **orchestrator** poore workflow ko control karta hai, har service ko explicitly call/command karta hai.
```
Orchestrator:
  1. Payment Service → "charge" → wait for result
  2. Inventory Service → "reserve" → wait
  3. Shipping Service → "ship" → wait
  4. Order complete
(orchestrator poora flow jaanta + control karta hai)
```
**Pros:** flow **ek jagah visible** (easy samajhna/debug), explicit error handling, clear ownership.
**Cons:** orchestrator ek coupling point (sabko jaanta hai), ek potential bottleneck/SPOF, orchestrator complex ho sakta.

### Comparison
| | Choreography | Orchestration |
|---|---|---|
| Control | Decentralized (events) | Centralized (orchestrator) |
| Coupling | Loose | Orchestrator tightly coupled |
| Visibility | Hard (flow implicit) | Easy (flow explicit) |
| Flexibility | Easy add services | Orchestrator update karo |
| Debugging | Hard | Easier |
| Best for | Simple flows, high decoupling | Complex flows, clear control |

**Interview:** *"Simple flows with few steps aur high decoupling chahiye → choreography (events). Complex multi-step workflows jahan visibility + error handling + compensation chahiye → orchestration (saga orchestrator — lesson 08). Zyadatar real systems dono mix karte hai."*

---

## 4. EDA ke benefits & challenges

**Benefits:** extreme decoupling (services independent), scalability (async, buffer spikes — Topic 4 load leveling), extensibility (naye consumers bina producer change), resilience (consumer down → events queue me wait), audit trail (events ka log).

**Challenges:**
- **Eventual consistency** (Topic 5 lesson 03) — events async process hote hai, saara system ek saath consistent nahi
- **Debugging** — ek business flow multiple services + events me faila (distributed tracing chahiye)
- **Event schema evolution** — event format badle toh consumers toot sakte hai (versioning, schema registry)
- **Duplicate/ordering** (Topic 4 lesson 05) — at-least-once + idempotency + partition-key ordering
- **Dual-write problem** (Topic 4 lesson 08) — DB + event atomically → **outbox pattern**

> **Recap (Topic 4 lesson 08):** EDA me "DB update + event publish" ka dual-write problem aata hai — outbox pattern se solve. Aur CDC se DB changes ko events me convert kar sakte ho. Ye EDA ke core reliability tools hai.

---

## 5. Kab EDA use karo

**✅ Use:** multiple services ko ek action pe react karna (fan-out — Topic 4 lesson 09), loose coupling critical, async processing acceptable, audit trail chahiye, spiky workloads (buffer).
**❌ Avoid:** simple request-response kaafi ho, strong immediate consistency chahiye (EDA eventual hai), synchronous result chahiye (user wait kar raha), team distributed-systems ke liye ready nahi.

---

## Quick Self-Check
1. Event-driven vs request-driven — coupling ke terms me difference?
2. Command vs event — directed/imperative vs broadcast/past-tense? Frontend analogy?
3. Choreography kya hai — har service kaise react karti (no coordinator)?
4. Orchestration kya hai — orchestrator ka role?
5. Choreography vs orchestration — visibility, coupling, kab kaunsa?
6. EDA ke 3 benefits + 3 challenges?
7. EDA me dual-write problem kaise solve (Topic 4 outbox recap)?

---
**Next:** `06-CQRS.md` — reads aur writes ko alag karna.