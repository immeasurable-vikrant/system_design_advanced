# 06 — CQRS (Command Query Responsibility Segregation)

> **Is lesson me:** CQRS pattern — reads aur writes ko alag models me separate karna, kyu aur kab, read models kaise sync hote hai, aur trade-offs. Ye Topic 2 (read replicas), Topic 3 (caching, denormalization), aur Topic 4 (events) ko ek pattern me combine karta hai.

Prerequisites: `05-Event-Driven-Architecture.md`, Topic 2 lesson 12 (replication)

---

## 1. CQRS kya hai

**CQRS (Command Query Responsibility Segregation)** = writes (commands) aur reads (queries) ke liye **alag models** use karna. Ek hi data model dono ke liye optimize karne ke bajaye, tum do specialized models banate ho — ek writes ke liye, ek reads ke liye.

```
TRADITIONAL (ek model dono ke liye):
  Reads + Writes → Same Model → Same Database
  (ek schema jo dono compromise kare)

CQRS (alag models):
  Writes (Commands) → Write Model → Write DB (normalized, transactional)
                                        │
                                   sync (events)
                                        ▼
  Reads (Queries) → Read Model → Read DB (denormalized, query-optimized)
```

> **Recap (Topic 2 lesson 02):** normalized data (no duplication, easy writes) vs denormalized (fast reads, no JOINs). CQRS is trade-off ko **explicit** banata hai — write side normalized (correctness), read side denormalized (speed). Dono ko alag optimize karo.

---

## 2. Kyu — the motivation

Ek single model dono ke liye aksar dono ke liye **suboptimal** hota hai:
```
Write side chahta hai:  normalized (no duplication), transactional, validated
Read side chahta hai:   denormalized (no JOINs), pre-computed, fast

Ek model dono nahi de sakta:
  Normalized → reads me complex JOINs (slow — Topic 2 lesson 07)
  Denormalized → writes me duplication/consistency issues
```

CQRS bolta hai: **stop compromising, use two models.** Write model correctness ke liye, read model(s) speed ke liye.

**Real example — e-commerce order:**
```
Write model (normalized): orders, order_items, users, products (separate tables, FKs)
Read model (denormalized): ek "order_summary" document —
  { orderId, customerName, items: [...], total, status } — ek read, no JOINs
```

**Frontend analogy:** ye Redux me normalized store (source of truth) vs memoized selectors (derived, denormalized view for rendering) jaisa hai. Tum writes normalized state pe karte ho, but reads ek derived/computed shape se serve karte ho jo UI ke liye optimized hai. CQRS wahi backend + database level pe.

---

## 3. Read models kaise sync hote hai

Write model badle toh read model(s) update hone chahiye. Usually **events** se (EDA — lesson 05):
```
1. Command aaye → Write model update → Write DB
2. Write model ek event emit kare ("OrderUpdated")
3. Read model updater event consume kare
4. Read DB (denormalized view) update kare
5. Queries read DB se serve hote hai (fast)
```

> ⚠️ **Eventual consistency (Topic 5 lesson 03):** read model async update hota hai — ek chhota lag ho sakta hai jab write ho chuka but read model abhi update nahi hua. Ye CQRS ka core trade-off hai. Zyadatar cases me acceptable (Topic 5 — "read-your-writes" ke liye write ke baad write-model se padho, ya UI optimistically update kare).

---

## 4. Multiple read models (the power)

CQRS ki asli power — **ek write model, multiple read models**, har ek alag query pattern ke liye:
```
Write Model (orders)
     │ events
     ├──▶ Read Model 1: SQL denormalized (order history — Topic 2)
     ├──▶ Read Model 2: Elasticsearch (order search — Topic 3)
     ├──▶ Read Model 3: Redis (real-time order status — Topic 3)
     └──▶ Read Model 4: Analytics warehouse (reporting)
```
Har read model apne use case ke liye best store use karta hai (polyglot — Topic 2 lesson 14). Ye "purpose-built views" pattern hai.

---

## 5. CQRS trade-offs

| Pros | Cons |
|---|---|
| Reads aur writes independently scale | Complexity (do models, sync logic) |
| Har side optimized (fast reads, clean writes) | Eventual consistency (read lag) |
| Multiple read models (purpose-built) | More infrastructure (read DBs, event pipeline) |
| Read/write load isolation | Data duplication (read models me copies) |

**⚠️ CQRS over-engineering hai zyadatar apps ke liye.** Ye complexity laata hai — sirf tab use karo jab read aur write needs **genuinely diverge** karti ho.

---

## 6. Kab CQRS use karo

**✅ Use:** read aur write load bahut alag (read-heavy, ya alag scaling), complex read queries jo write schema pe slow ho, multiple read representations chahiye (search + list + analytics), event sourcing ke saath (lesson 07 — natural fit), collaborative domains (high write contention).

**❌ Avoid:** simple CRUD (ek model kaafi), reads aur writes similar, strong consistency zaroori (read lag acceptable nahi), small app (complexity worth nahi).

**Interview line:** *"CQRS tab jab read aur write requirements genuinely diverge karein — e.g., ek write-heavy transactional core + multiple read-optimized views (search, dashboard, analytics). Write model normalized correctness ke liye, read models denormalized speed ke liye, events se synced. But ye eventual consistency + complexity laata hai, toh simple CRUD ke liye over-engineering — main isse justify hone pe hi use karunga."*

> **Note:** CQRS ka ek "lite" version — bina alag databases ke, bas read replicas (Topic 2 lesson 12) ko reads ke liye + primary ko writes ke liye. Ye already ek mild CQRS hai. Full CQRS (alag models + event sync) tab jab wo kaafi na ho.

---

## Quick Self-Check
1. CQRS kya hai — reads/writes alag model kyu?
2. Ek model dono ke liye kyu suboptimal (normalized vs denormalized — Topic 2)?
3. Read model write model se kaise sync hota hai (events)?
4. Iska core trade-off kya (eventual consistency — Topic 5)?
5. "Multiple read models" ki power kya (purpose-built views)?
6. Redux normalized-store + selectors analogy kaise fit hoti hai?
7. Kab CQRS use karo, kab over-engineering? "CQRS-lite" (read replicas) kya?

---
**Next:** `07-Event-Sourcing.md` — state ki jagah events store karna.