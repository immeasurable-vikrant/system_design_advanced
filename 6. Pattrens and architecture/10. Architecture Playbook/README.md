# 10 — Architecture Playbook (Capstone)

> **Is lesson me:** Poore module ka synthesis — patterns kaise choose karo, ek architecture decision framework, common pattern combinations, aur **poore 6-topic curriculum ka final map**. Ye tumhara master reference hai jab tum ek system architect karo.

Prerequisites: **saare pichle lessons (01-09), ideally poore Topics 1-6.**

---

## 1. The architecture decision framework

Jab ek system design karo, in order me socho:

```
STEP 1 — MONOLITH ya MICROSERVICES? (lesson 01)
  Early/small/unclear domain? → Modular monolith
  Scale/teams/clear domains?   → Microservices (evolve into, not start with)

STEP 2 — SERVICES KAISE BAAT KAREIN? (lesson 02, 05)
  User waiting?         → Sync (REST/GraphQL/gRPC)
  Background/decouple?  → Async (events — Kafka/queues, Topic 4)
  Public API?           → REST | Mobile? → GraphQL | Internal? → gRPC

STEP 3 — SERVICES KAISE DHUNDHEIN? (lesson 03)
  → Service discovery (Consul/K8s) + externalized config

STEP 4 — RELIABLE KAISE? (lesson 04)
  → Timeout + circuit breaker + retry + bulkhead + fallback (har call)

STEP 5 — DATA & CONSISTENCY? (lesson 06, 07, 08)
  Read/write diverge?        → CQRS
  Audit/history critical?    → Event sourcing
  Cross-service transaction? → Saga (compensations)
  Consistency level?         → Per-component (Topic 5: CP vs AP)

STEP 6 — EVOLVE KAISE? (lesson 09)
  → Strangler fig (incremental), never big-bang
```

**Golden principle:** *Patterns solutions hai specific problems ke. Pehle problem identify karo, phir pattern. "Microservices/CQRS/event-sourcing kyunki cool hai" = over-engineering. Har pattern complexity laata hai — sirf tab jab benefit > cost.*

---

## 2. Common pattern combinations (real architectures)

Patterns akele nahi, combinations me aate hai:

| Architecture | Patterns combined |
|---|---|
| **Modern microservices** | Microservices + API gateway + service mesh + circuit breaker + service discovery |
| **Event-driven system** | EDA + message queue + outbox + idempotency (Topic 4) |
| **High-scale reads** | CQRS + read replicas (Topic 2) + caching (Topic 3) |
| **Audit-critical (finance)** | Event sourcing + CQRS + saga |
| **Order/checkout flow** | Saga (orchestration) + circuit breaker + idempotency |
| **Legacy migration** | Strangler fig + anti-corruption layer + facade (gateway) |

---

## 3. The "design X" pattern checklist

Interview me "design a system" pe ye patterns naturally aate hai:

```
Design an e-commerce checkout:
  → Microservices (order/payment/inventory/shipping) — lesson 01
  → Saga orchestration for the transaction — lesson 08
  → Circuit breaker + retry on payment — lesson 04
  → Events for inventory/notification fan-out — lesson 05, Topic 4
  → Idempotency on payment (double-charge rok) — Topic 4

Design a social feed:
  → CQRS (write posts / read timelines) — lesson 06
  → Event-driven fan-out (Topic 4 lesson 09)
  → Eventual consistency (AP — Topic 5)
  → Caching read models (Topic 3)

Design a banking system:
  → Event sourcing (audit trail) — lesson 07
  → CQRS (balances + history) — lesson 06
  → Strong consistency / CP (Topic 5)
  → Saga for transfers — lesson 08
```

---

## 4. Interview one-liners (ready to say)

- **"Monolith or microservices?"** → *"Modular monolith first with clean bounded contexts. Extract services when a specific module needs independent scaling or team ownership — via strangler fig. Microservices are an evolution, not a starting point."*
- **"How do services communicate?"** → *"Sync via gRPC internally (fast, typed), REST/GraphQL at the edge. Async via events (Kafka) for decoupling and fan-out. User-facing = sync, background = async."*
- **"How do you handle a cross-service transaction?"** → *"Saga with compensating transactions, orchestration-based for visibility. Not 2PC — it's blocking and fragile. Accept eventual consistency, mark intermediate states 'pending', make compensations idempotent."*
- **"How do you keep the system resilient?"** → *"Every inter-service call wrapped in timeout + circuit breaker + retry (backoff/jitter, idempotent) + bulkhead + fallback. Usually via service mesh. Goal: no cascading failures."*
- **"How do you migrate a monolith?"** → *"Strangler fig — facade in front, extract leaf services first, verify each step, data last, anti-corruption layer to insulate from legacy. Never a big-bang rewrite."*

---

## 5. This module's concept map

```
STRUCTURE     monolith vs microservices (01) → API styles (02) → discovery (03)
RELIABILITY   resilience patterns: circuit breaker, bulkhead, fallback (04)
EVENT-BASED   event-driven arch (05) → CQRS (06) → event sourcing (07) → saga (08)
EVOLUTION     migration: strangler fig (09)
SYNTHESIS     this playbook (10)
```

---

## 6. 🎓 THE FULL CURRICULUM — all 6 topics

Tumne ab poora SDE-3 system design foundation complete kar liya:

```
Topic 1 — CORE INFRASTRUCTURE
  Networking, load balancing, proxies, CDN, API gateway, connection pooling
  → "Request client se server tak kaise pahunchti hai + scale"

Topic 2 — DATABASE & STORAGE
  RDBMS, SQL, indexing, transactions/ACID, isolation, WAL, replication, sharding,
  CAP, object storage
  → "Data kaise store, query, aur scale hoti hai"

Topic 3 — CACHING & PERFORMANCE
  Caching layers/strategies/eviction, Redis, rate limiting, bloom filters
  → "System kaise fast aur protected banao"

Topic 4 — COMMUNICATION & MESSAGING
  Real-time (WebSocket/SSE), message queues, delivery guarantees, idempotency,
  Kafka/SQS, DLQ, outbox, fan-out
  → "Components ek dusre se kaise baat karein (reliably)"

Topic 5 — DISTRIBUTED SYSTEMS
  CAP, consistency models, leader election, consensus, fencing tokens,
  consistent hashing, vector clocks, Merkle trees, leaderless replication
  → "Independent nodes kaise coordinate + agree karein"

Topic 6 — PATTERNS & ARCHITECTURE (ye)
  Microservices, API styles, resilience, event-driven, CQRS, event sourcing,
  saga, migration
  → "Ye sab ko ek coherent architecture me kaise organize karo"
```

**Kaise sab connect hota hai — ek complete system:**
```
Client request → [T1: DNS, CDN, LB, API gateway]
              → [T6: microservice architecture, resilience patterns]
              → [T4: async events between services, real-time to client]
              → [T3: cache checked first]
              → [T2: database — indexed, replicated, sharded]
              → [T5: all coordinated with correct consistency, consensus, conflict resolution]
              → [T6: CQRS read models, saga for transactions, event sourcing for audit]
```

Har layer ke fundamentals + trade-offs + interview framing — tumhare paas ab complete hai.

---

## 7. Final wisdom — the architect's mindset

```
1. PATTERNS ARE TOOLS, not goals. Problem pehle, pattern baad me.
2. EVERY PATTERN HAS A COST. Complexity, consistency, latency — kuch trade karo.
3. START SIMPLE, EVOLVE. Monolith → microservices jab pain real ho.
4. CONSISTENCY IS A SPECTRUM. Per-component choose karo (T5).
5. DESIGN FOR FAILURE. Sab fail hoga — resilience + idempotency built-in.
6. THERE'S NO PERFECT ARCHITECTURE. Sirf trade-offs jo tumhare context ke liye sahi ho.
```

---

## Quick Self-Check (capstone)
1. Architecture decision framework ke 6 steps (structure se evolution tak)?
2. E-commerce checkout ke liye kaunse patterns combine karoge?
3. Social feed vs banking system — alag patterns kyu (CQRS/ES/consistency)?
4. 5 interview one-liners (monolith, comms, saga, resilience, migration) — bol sakte ho?
5. 6 topics — har ek ek line me kya cover karta hai?
6. Ek request poore 6 topics se kaise flow karti hai?
7. "Architect's mindset" ke 6 principles?

---

**🎉 Topic 6 complete — aur POORA 6-TOPIC CURRICULUM complete!**

Tumne ek complete SDE-3 system design foundation build kar li hai — infrastructure, data, performance, communication, distributed systems theory, aur architectural patterns. Har topic:
- Zero se explained, frontend/JS analogies ke saath
- Deeply cross-linked (har concept doosre topics se connected)
- Interview framing + trade-offs har jagah
- ~65 lessons, ek connected knowledge graph

**Ab tum ready ho** kisi bhi HLD interview ke liye — blank whiteboard se, tum ek request ke poore safar (DNS → gateway → services → cache → database) ko design kar sakte ho, har layer pe sahi patterns choose karke, trade-offs justify karke, aur "kyu" explain karke.

**Final step:** ye knowledge ko **actual end-to-end designs** pe apply karo — Design WhatsApp (T1 gateway + T4 WebSocket/fan-out + T2 sharded message store + T5 consistency), Design Twitter (T6 CQRS + T4 fan-out + T3 caching + T2 sharding), Design Uber (T1 real-time + T5 geo-consistency + T4 events), Design a Payment System (T6 saga + T7 event sourcing + T5 strong consistency). Har design me ab tum confidently saare 6 topics ke pieces assemble kar paoge.

Ye raha tumhara complete system design brain. 🚀 All the best for SDE-3!