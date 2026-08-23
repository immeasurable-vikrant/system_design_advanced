# 10 — Distributed Systems Playbook (Capstone)

> **Is lesson me:** Poore module ka synthesis — ek decision framework (kab consensus, kab leaderless, kaunsa consistency), ye concepts real designs me kaha aate hai, aur saare 5 topics ka final concept map. Ye tumhara interview reference hai distributed-systems questions ke liye.

Prerequisites: **saare pichle lessons (01-09).**

---

## 1. The mental model — 3 problems, many solutions

Poora module 3 fundamental problems (lesson 01) aur unke solutions ka hai:
```
PROBLEM                          SOLUTIONS
─────────────────────────────────────────────────────────────
Partial failure (dead vs slow?)  → Leader election + quorum (04)
                                 → Fencing tokens (05)
Network partition (C or A?)      → CAP theorem → per-component choice (02)
                                 → Consistency models spectrum (03)
No global clock (event order?)   → Vector clocks / logical clocks (07)
Data distribution                → Consistent hashing (06)
Replica drift                    → Merkle trees / anti-entropy (08)
```

**Golden principle:** *Distributed systems me tum guarantees nahi paate — tum trade-offs choose karte ho. Har design decision "kya sacrifice karu" hai: consistency ya availability, latency ya freshness, simplicity ya scalability.*

---

## 2. Decision framework — kya kab chahiye

### Consistency — kaunsa model? (lesson 02-03)
```
Money / inventory / locks?     → Strong (linearizable) — CP
Feeds / carts / sessions?      → Eventual — AP
User apna data dekhe?          → Read-your-writes
Chat message order?            → Causal
Per-component decide karo, poore system ke liye nahi!
```

### Coordination — leader chahiye? (lesson 04-05)
```
Ek node writes serialize kare?     → Single leader + Raft/ZK election
Job assignment / config agree?     → Consensus (Raft/etcd)
Lock protects external writes?     → Distributed lock + FENCING TOKENS
Always-writable, no bottleneck?    → Leaderless (no leader) — lesson 09
```

### Data distribution (lesson 06)
```
Keys ko nodes me baato, minimal reshuffling on scaling? → Consistent hashing
Fault tolerance?                                        → Replicate to N consecutive nodes
```

### Conflicts & repair (lesson 07-08)
```
Concurrent writes possible (leaderless)?  → Vector clocks (detect) + resolution (LWW/merge)
Replicas drift?                           → Merkle trees (anti-entropy repair)
```

---

## 3. Ye concepts real designs me kaha aate hai

Interview me "design X" pe ye concepts naturally aate hai:

| System | Distributed concepts |
|---|---|
| **Digital wallet** | Strong consistency (CP), single-leader, fencing tokens, linearizable |
| **Social feed** | Eventual consistency (AP), leaderless, gossip |
| **Chat system** | Causal consistency (message order), per-component (receipts = eventual) |
| **Job scheduler** | Leader election (Raft/ZK), fencing tokens (no double-execution) |
| **Key-value store** | Consistent hashing, quorum, vector clocks, Merkle trees (Dynamo — lesson 09) |
| **Config service (etcd)** | Raft consensus, linearizable |
| **Distributed cache** | Consistent hashing (Topic 3 + Topic 1) |
| **Kafka** | Leader per partition (ZK/KRaft), producer epoch (fencing), ISR quorum |

---

## 4. Interview one-liners (ready to say)

- **"Is your system CP or AP?"** → *"Per component. Transaction ledger CP (linearizable, single Postgres primary). Feed AP (eventual, leaderless). Notifications AP. Alag components, alag needs."* (lesson 02)
- **"How do you elect a leader?"** → *"Raft — majority-vote consensus with randomized election timeouts and terms. Quorum prevents split-brain. Or ZooKeeper ephemeral sequential znodes if ZK already in infra."* (lesson 04)
- **"How do you prevent a stale lock holder corrupting data?"** → *"Fencing tokens — monotonic number per lock, storage rejects writes with a token below the max seen. Moves the safety guarantee to the storage layer."* (lesson 05)
- **"How do you detect conflicting concurrent writes?"** → *"Vector clocks — if neither dominates the other, they're concurrent. Store as siblings, resolve with domain logic (union for carts) or LWW."* (lesson 07)
- **"How do you sync out-of-sync replicas efficiently?"** → *"Merkle trees — compare root hashes, traverse only differing branches, transfer only differing blocks. O(log N) instead of O(N)."* (lesson 08)

---

## 5. Full module concept map

```
FOUNDATION       why DS is hard: partial failure, no clock, asynchrony, consensus (01)
CONSTRAINTS      CAP + PACELC (02) → consistency models spectrum (03)
COORDINATION     leader election + Raft/ZK + split-brain (04) → fencing tokens (05)
DISTRIBUTION     consistent hashing + replication + rendezvous (06)
CONFLICTS        vector clocks: causality + conflict detection (07)
REPAIR           Merkle trees: anti-entropy + efficient sync (08)
SYNTHESIS        leaderless Dynamo model (09) → this playbook (10)
```

---

## 6. Saare 5 topics ab complete — the full curriculum

```
Topic 1 (Core Infra)       → request ka safar: network, LB, proxy, CDN, gateway, pool
Topic 2 (Database)         → data: RDBMS, indexing, transactions, replication, sharding
Topic 3 (Caching/Perf)     → speed: caching, rate limiting, bloom filters
Topic 4 (Comm/Messaging)   → communication: real-time, queues, fan-out, reliability
Topic 5 (Distributed Sys)  → theory: CAP, consistency, consensus, conflicts, repair
```

**Ye Topic 5 baaki sabki neenv hai:**
- Topic 2 ki replication/quorum/sharding → yahan ke CAP, consistency, consistent hashing pe khadi hai
- Topic 1 ki load balancing/consistent hashing → yahan ke distribution theory
- Topic 4 ki idempotency/Kafka epochs → yahan ke fencing/consensus
- Topic 3 ki distributed cache → yahan ke consistent hashing

Ek mature system design me tum ye layers ek saath use karte ho — infrastructure pe requests aati hai (T1), messaging se services decouple hoti hai (T4), cache se fast serve hoti hai (T3), database me data rehta hai (T2), aur ye sab **distributed systems principles** (T5) pe correctly coordinate karta hai.

---

## 7. The distributed systems mindset (final wisdom)

Jaate-jaate, ye 5 truths yaad rakho:
```
1. Failures NORMAL hai, exception nahi. Design for partial failure.
2. Network unreliable hai. Timeouts guesses hai. Retry with idempotency.
3. Clocks jhooth bolte hai. Order ke liye logical clocks (vector/Lamport).
4. Consistency vs availability ek trade-off hai. Per-component choose karo.
5. Tum guarantees nahi paate — tum trade-offs manage karte ho.
```

---

## Quick Self-Check (capstone)
1. 3 fundamental problems aur unke solutions (map bina dekhe)?
2. Consistency model kaise choose karo — money vs feed vs chat?
3. Kab leader chahiye vs leaderless? Kab consensus (Raft)?
4. Digital wallet vs social feed — kaunse distributed concepts (CP vs AP, etc.)?
5. 5 interview one-liners (CAP, leader, fencing, vector clocks, Merkle) — bol sakte ho?
6. Topic 5 baaki 4 topics ki neenv kaise hai (ek-ek example)?
7. "Distributed systems mindset" ke 5 truths?

---

**🎉 Topic 5 complete — aur poora curriculum complete!** Ab tum distributed systems ki theory samajhte ho — kyu ye mushkil hai, aur engineers ne consistency, coordination, conflicts, aur repair ko kaise solve kiya.

**Saare 5 topics:**
- **Topic 1 — Core Infrastructure**
- **Topic 2 — Database & Storage**
- **Topic 3 — Caching & Performance**
- **Topic 4 — Communication & Messaging**
- **Topic 5 — Distributed Systems**

Tumhare paas ab ek **complete, deep system design foundation** hai — infrastructure se leke data, performance, communication, aur underlying distributed-systems theory tak. Har layer ke fundamentals + trade-offs + interview framing.

**Agla step:** ye sab ko actual **end-to-end HLD problems** pe apply karo (Design WhatsApp, Twitter, Uber, a Key-Value Store, a Rate Limiter, a Payment System). Har design me tum ab confidently in 5 topics ke pieces ko ek coherent architecture me assemble kar paoge — client se request aane se leke distributed database me consistently data store hone tak, har decision ke peeche ka "kyu" ke saath. Whiteboard pe blank se pura system nikaalne ke liye tum ready ho. 🚀