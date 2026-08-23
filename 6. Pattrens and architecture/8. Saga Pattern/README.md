# 08 — Saga Pattern (Distributed Transactions)

> **Is lesson me:** Microservices me ek transaction jo multiple services span kare (each apna DB) use kaise atomic-ish banaye — saga pattern, compensating transactions, choreography vs orchestration sagas, aur 2PC se kyu better. Topic 2 lesson 13 me saga vs 2PC chhua tha; yahan full architectural treatment.

Prerequisites: `05-Event-Driven-Architecture.md`, Topic 2 lesson 08-09 (transactions), Topic 2 lesson 13 (2PC/saga)

---

## 1. The problem — transaction across services

Monolith me ek order place karna ek **single ACID transaction** hai (Topic 2 lesson 08) — payment + inventory + shipping sab ek DB, sab ya commit ya rollback.

Microservices me har service apna DB rakhta hai (lesson 01). Ek order me:
```
Order Service (Order DB)     → order create
Payment Service (Payment DB) → charge card
Inventory Service (Inv DB)   → reserve stock
Shipping Service (Ship DB)   → schedule shipment
```
Ye **4 alag databases** hai. Ek ACID transaction inpe span nahi kar sakti (no shared transaction — Topic 2 lesson 13). Toh agar payment success but inventory fail ho — kya karo? Paisa kat gaya, saman reserve nahi hua. **Inconsistent!**

> **Recap (Topic 2 lesson 13):** cross-shard/cross-service transactions ke liye 2 options the — 2PC (blocking, fragile) ya saga (compensating). Yahan saga ko detail me.

---

## 2. Saga pattern kya hai

**Saga** = ek distributed transaction ko **local transactions ki sequence** me toDna, jaha har step apne service me commit hota hai, aur agar koi step fail ho toh **compensating transactions** (undo actions) pichle steps ko reverse karte hai.

```
Normal flow (sab success):
  T1: Order create      → T2: Payment charge  → T3: Inventory reserve → T4: Ship
  (har step apna local transaction commit karta hai)

Failure flow (T3 fail):
  T1: Order create ✓ → T2: Payment charge ✓ → T3: Inventory reserve ✗ FAIL
  Compensate (reverse order me undo):
  C2: Payment refund  → C1: Order cancel
  (har completed step ka "undo" chalao)
```

**Key idea:** koi global rollback nahi (no shared transaction). Instead, har step ka ek **compensating action** (undo) define karo. Fail hone pe, completed steps ko unke compensations se reverse karo.

**Frontend analogy:** ye ek multi-step wizard/checkout jaisa hai jaha agar step 4 fail ho, tum manually steps 1-3 ke effects undo karte ho (reserved cart release, applied coupon revert). Saga isko formalize karta hai — har forward action ka ek backward action.

---

## 3. Compensating transactions ⭐

Har saga step ke liye ek **compensating transaction** define karo — jo us step ka business effect undo kare:

| Forward action | Compensating action |
|---|---|
| Order create | Order cancel |
| Payment charge | Payment refund |
| Inventory reserve | Inventory release |
| Loyalty points deduct | Points restore |

> ⚠️ **Compensation ≠ rollback.** Ek DB rollback data ko pehle jaisa kar deta hai (jaise kuch hua hi nahi). Compensation ek **new action** hai jo effect ko counteract karti hai — but trace rehta hai. E.g., "charge" ke baad "refund" — dono transactions ledger me dikhte hai (charge + refund), na ki "charge kabhi hua hi nahi". **Semantically undo, not literally.**

**Important properties:**
- **Idempotent** (Topic 4 lesson 05) — compensation retry-safe honi chahiye (double-refund na ho)
- **Commutative-ish** — order me robust honi chahiye
- **Always succeed** (eventually) — compensation fail nahi honi chahiye (warna stuck) — isliye retry + eventually manual intervention

---

## 4. Choreography Saga (events)

> **Recap (lesson 05):** choreography = no central coordinator, services events se react karte hai.

```
Order Service:    order create → "OrderCreated" emit
Payment Service:  "OrderCreated" sune → charge → "PaymentCompleted" emit
Inventory Service:"PaymentCompleted" sune → reserve → "InventoryReserved" emit
Shipping Service: "InventoryReserved" sune → ship

Failure (Inventory fail):
Inventory Service: reserve fail → "InventoryFailed" emit
Payment Service:   "InventoryFailed" sune → refund → "PaymentRefunded" emit
Order Service:     "PaymentRefunded" sune → order cancel
```
**Pros:** decoupled, no central point. **Cons:** flow implicit (hard to see/debug), cyclic risk, har service ko compensation logic pata honi chahiye.
**Kab:** simple sagas (3-4 steps), high decoupling.

---

## 5. Orchestration Saga (coordinator) ⭐

> **Recap (lesson 05):** orchestration = ek central orchestrator poore flow ko control karta hai.

```
Saga Orchestrator:
  1. Order Service → create → OK
  2. Payment Service → charge → OK
  3. Inventory Service → reserve → FAIL!
  4. Orchestrator compensation shuru kare (reverse):
     → Payment Service → refund
     → Order Service → cancel
  5. Saga failed, cleanly compensated
```
Orchestrator ek **state machine** hai — jaanta hai kaunsa step chal raha, kya success/fail hua, kab compensate karna.

**Pros:** flow explicit + visible (easy debug), centralized error handling, clear compensation logic, complex flows manageable.
**Cons:** orchestrator ek component (SPOF risk — HA banao), thoda coupling.
**Kab:** complex sagas (many steps), visibility + control chahiye. **Most production systems ye prefer karte hai.**

**Tools:** Temporal, AWS Step Functions, Camunda, Netflix Conductor.

---

## 6. Saga vs 2PC (recap + depth)

> **Recap (Topic 2 lesson 13):**
| | 2PC (Two-Phase Commit) | Saga |
|---|---|---|
| Mechanism | Coordinator prepare→commit, locks held | Local transactions + compensations |
| Consistency | Strong (atomic) | Eventual |
| Locks | Held across services (blocking) | No distributed locks |
| Failure | Coordinator crash → resources locked | Compensate + retry |
| Scalability | Poor (blocking, chatty) | Good |
| Best for | Rare, short, critical | Microservices (standard) |

**2PC blocking hai + coordinator failure pe resources locked** (fragile). **Saga non-blocking hai** (local commits) but **eventual consistency** (beech me system inconsistent dikh sakta hai — "order placed but payment pending"). Microservices me saga standard hai.

---

## 7. Saga ke challenges

- **Eventual consistency** — saga ke beech me system temporarily inconsistent (order created, payment pending). UI ise handle kare ("processing").
- **No isolation** — dusre transactions saga ke intermediate state dekh sakte hai (dirty reads across services). "Semantic locks" (status flags like "pending") se manage.
- **Compensation complexity** — har step ka reliable undo design karna hard (kuch cheezein truly undo nahi hoti — email bhej diya toh?).
- **Debugging** — distributed flow, especially choreography.

> **Recap (Topic 2 lesson 09):** saga me isolation nahi hota (unlike ACID). Isliye "pending" status flags (semantic locks) use karte hai taaki intermediate state clearly marked ho.

**Interview line:** *"Cross-service transaction ke liye main saga use karunga — local transactions with compensating actions, na ki 2PC (jo blocking + fragile hai). Orchestration-based (Temporal/Step Functions) for complex flows — flow visible, compensation centralized. Eventual consistency accept karke, intermediate states ko 'pending' status se mark karunga, aur compensations ko idempotent banaunga."*

---

## Quick Self-Check
1. Microservices me ek ACID transaction across services kyu nahi ho sakti?
2. Saga kya hai — local transactions + compensating actions?
3. Compensating transaction "rollback" se kaise alag (charge→refund, both traced)?
4. Compensation idempotent kyu honi chahiye (Topic 4)?
5. Choreography vs orchestration saga — control aur visibility?
6. Saga vs 2PC — blocking, consistency, kab kaunsa (Topic 2 recap)?
7. Saga me isolation kyu nahi, aur "semantic locks" (pending status) kaise help?

---
**Next:** `09-Migration-Patterns.md` — monolith ko microservices me safely le jaana.