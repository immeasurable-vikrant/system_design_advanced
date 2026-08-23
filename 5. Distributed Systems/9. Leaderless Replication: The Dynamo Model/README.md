# 09 — Leaderless Replication: The Dynamo Model (Synthesis)

> **Is lesson me:** Sab pichle concepts ek real system me kaise combine hote hai — Amazon Dynamo (aur Cassandra/DynamoDB/Riak) ka leaderless replication model. Consistent hashing + quorum + vector clocks + Merkle trees + fencing — sab yahan ek coherent design me fit hote hai. Ye module ka synthesis hai jo dikhata hai ye "separate" topics actually ek family hai.

Prerequisites: **lessons 02-08** (CAP, consistency, consistent hashing, vector clocks, Merkle trees)

---

## 1. Leaderless replication kya hai (recap + context)

> **Recap (Topic 2 lesson 12):** teen replication topologies — single-leader, multi-leader, **leaderless**. Leaderless me **koi fixed leader nahi** — client directly multiple replicas ko likhta/padhta hai (Cassandra, DynamoDB).

Ye "Dynamo model" kehlata hai (Amazon ka 2007 ka famous paper jisne Cassandra/Riak/DynamoDB ko inspire kiya). Iska poora design in module ke concepts se bana hai. Chalo dekhein kaise sab fit hote hai.

**Kyu leaderless:** single leader ek bottleneck + single point of failure hai (Topic 2). Leaderless = **always writable** (koi bhi node), high availability (AP-leaning — lesson 02), no failover downtime. Cost: consistency mushkil (conflicts possible).

---

## 2. Building block 1 — Consistent Hashing for placement

Data kis node pe jayega? **Consistent hashing** (lesson 06).
```
Har key → ring pe hash → clockwise pehle N nodes = uske replicas (preference list)
Node add/remove → sirf ~K/N keys move (minimal disruption)
Virtual nodes → even distribution
```
Ye "koi central directory nahi" deta hai — koi bhi node hash karke jaan sakta hai ki ek key kaunse nodes pe hai. Decentralized.

---

## 3. Building block 2 — Quorum for consistency

Bina leader ke, "latest data" kaise ensure karo? **Quorum (R + W > N)** — lesson pe recap:

> **Recap (Topic 2 lesson 12):**
> ```
> N = total replicas, W = write quorum, R = read quorum
> R + W > N → read aur write sets OVERLAP → latest data guaranteed
> ```
```
N=3, W=2, R=2:
  WRITE: 2 nodes confirm kare → success (3rd async)
  READ:  2 nodes se padho → jo latest version wo return
  Overlap: 2+2=4 > 3 → kam se kam 1 common node (jisne latest dekha) ✅
```

**Tunable consistency:** W aur R adjust karke consistency-vs-latency choose karo:
```
W=N, R=1 → writes slow but strong, reads fast
W=1, R=N → writes fast, reads slow but strong
W=2, R=2, N=3 → balanced (tolerates 1 failure)
```
> **Recap (Topic 2 lesson 14 — CAP):** `R+W>N` = CP-leaning; `R+W<=N` = AP-leaning (fast but stale possible). Tunable consistency = per-query CAP choice.

---

## 4. Building block 3 — Vector Clocks for conflicts

Leaderless me do clients same key ek saath likh sakte hai (no leader to serialize). Conflict! **Vector clocks** (lesson 07) detect karte hai:
```
Client 1 writes → VC=[2,0]
Client 2 writes → VC=[0,2]  (concurrent — neither dominates)
→ CONFLICT detected → dono versions "siblings" ki tarah store
→ Next read pe resolve (LWW, ya client merge — e.g., shopping cart union)
```
Bina vector clocks, ek write silently doosre ko overwrite kar deta (lost update). Vector clocks conflict ko **visible** banate hai taaki safely resolve ho.

---

## 5. Building block 4 — Handling temporary failures (hinted handoff)

Ek replica node temporarily down ho toh write kaha jaye? **Hinted handoff:**
```
Write ke liye Node C down hai (jo replica hona chahiye)
→ Write ek doosre node D ko jaata hai ek "hint" ke saath ("ye C ka hai")
→ C wapas aane pe, D use wo data handoff kar deta hai
```
Isse system **writable rehta hai** even during failures (availability — AP). Ye quorum ke saath kaam karta hai (W nodes me temporary substitutes count ho sakte hai — "sloppy quorum").

---

## 6. Building block 5 — Merkle Trees for repair

Async writes, hinted handoffs, missed updates → replicas **drift** ho jaate hai. **Merkle trees** (lesson 08) background me sync karte hai:
```
Anti-entropy: replicas periodically Merkle roots compare karein
→ O(log N) me differing data blocks dhundo
→ Sirf wo blocks transfer karke sync karo
```
Ye "eventually consistent" guarantee ko back karta hai — even agar real-time writes miss ho, background repair sab ko converge kar deta hai.

**Read repair:** jab read quorum pe stale replica detect ho (versions mismatch), system use turant latest se update kar deta hai (read ke time hi repair).

---

## 7. Building block 6 — Gossip for membership

Nodes ko kaise pata kaun-kaun cluster me hai (aur kaun down)? **Gossip protocol:**
```
Har node periodically kuch random nodes se apna state exchange kare
("main zinda hu, ye nodes maine dekhe, ye down lagte hai")
→ Information poore cluster me "gossip" ki tarah phailti hai
→ Eventually sab ko consistent membership view milta hai
```
Decentralized — koi central registry nahi. Failure detection + membership dono.

---

## 8. Sab ek saath — the complete picture

```
┌─────────────── DYNAMO-STYLE LEADERLESS SYSTEM ──────────────────┐
│                                                                  │
│  Client koi bhi node ko write/read kar sakta (no leader)         │
│                                                                  │
│  PLACEMENT:    Consistent hashing ring → N replicas per key (06) │
│  CONSISTENCY:  Quorum R+W>N, tunable (Topic 2 / lesson 02)       │
│  CONFLICTS:    Vector clocks detect concurrent writes (07)       │
│  FAILURES:     Hinted handoff keeps writes flowing               │
│  REPAIR:       Merkle trees anti-entropy + read repair (08)      │
│  MEMBERSHIP:   Gossip protocol (decentralized failure detection) │
│  SAFETY:       Fencing/epochs where coordination needed (05)     │
│                                                                  │
│  Result: AP system — always available, eventually consistent    │
└──────────────────────────────────────────────────────────────────┘
```

**Ye poori beauty hai:** module ke "alag-alag" topics actually **ek coherent system** ke pieces hai. Dynamo paper ne inhe combine karke ek always-available, horizontally-scalable, self-healing datastore banaya — jo Cassandra, Riak, DynamoDB ka foundation bana.

**Used by:** Cassandra, DynamoDB, Riak, Voldemort. Ye AP-leaning hai — isliye social feeds, shopping carts, sessions ke liye (Topic 2 lesson 14 ka AP use cases).

---

## 9. Single-leader vs Leaderless (the choice)

| | Single-leader (Postgres) | Leaderless (Cassandra) |
|---|---|---|
| Writes | Ek leader (serialize) | Koi bhi node |
| Consistency | Strong (easy) | Tunable (quorum) |
| Conflicts | Nahi (leader serializes) | Possible (vector clocks) |
| Availability | Leader down = write outage | Always writable |
| CAP | CP-leaning | AP-leaning |
| Best for | Transactions, money | Scale, availability, feeds |

**Interview insight:** *"Leaderless replication high availability + write scalability deta hai, but consistency ke liye quorum tuning aur conflict resolution (vector clocks) chahiye. Money/transactions ke liye single-leader (strong consistency) better; feeds/carts/sessions ke liye leaderless (always available)."*

---

## Quick Self-Check
1. Leaderless replication kya hai, aur kyu (always writable, no SPOF)?
2. Consistent hashing leaderless me kya role (placement, preference list)?
3. Quorum R+W>N bina leader ke consistency kaise deta hai (overlap)?
4. Vector clocks leaderless me kya solve karte hai (concurrent write conflicts)?
5. Hinted handoff kya hai (temporary failure me writable rehna)?
6. Merkle trees + read repair replicas ko kaise converge karte hai?
7. Gossip protocol kya karta hai (membership, decentralized)?
8. Single-leader vs leaderless — kab kaunsa?

---
**Next:** `10-Distributed-Systems-Playbook.md` — sab tie together, decision framework.