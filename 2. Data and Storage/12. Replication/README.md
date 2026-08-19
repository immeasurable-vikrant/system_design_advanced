# 12 — Replication (Same Data, Multiple Copies)

> **Is lesson me:** Replication kyu, leader-follower model, sync vs async, replication lag + "read-your-own-writes", quorum (R+W>N), failover + split-brain + fencing tokens, multi-leader + CRDTs. Ye "scale reads + fault tolerance" ka solution hai (sharding "scale writes" ka — lesson 13).

Prerequisites: `10-WAL-Durability.md` (WAL replication ko power karta hai)

---

## 1. Scaling ka problem (context)

Ek single database server ki 2 fundamental limits:
- **Storage limit** — ek machine ke disk pe infinite data nahi
- **Throughput limit** — ek machine sirf itne reads/writes/sec handle kar sakta hai

Do solutions:
1. **Vertical scaling** — bada machine (more CPU/RAM/disk). Simple, but limit + mehenga.
2. **Horizontal scaling** — multiple machines. Yahi se **replication** aur **sharding** aate hai:
   - **Replication** → read throughput + fault tolerance
   - **Sharding** → storage + write throughput (lesson 13)

---

## 2. Replication kya hai

**Replication** = ek hi data ki **multiple copies** alag machines (replicas/nodes) pe maintain karna.

**Analogy:** Ek popular library jiske paas har book ki ek original copy hai. Building jal gayi → sab books lost. Replication = copies bana ke multiple branch libraries me rakhna. Readers kisi bhi branch se padh sakte hai; nayi book aaye toh decide karna padta hai kaun-si branch pehle paaye aur baaki kaise sync ho.

**Kyu replicate:**
| Reason | Explanation |
|---|---|
| **High availability** | Ek node marе, doosre serve karein |
| **Read scalability** | Read load multiple replicas pe distribute |
| **Geo-locality** | Data users ke paas rakho (low latency) |
| **Disaster recovery** | Poora data-center fail ho toh bhi survive |

> **Important:** Replication **storage capacity nahi badhata** — har replica pe **poora hi data** hota hai, sirf copies badh rahi. (Storage badhane ke liye sharding — lesson 13.)

---

## 3. Leader-Follower (Primary-Replica) — most common

```
              writes
                │
                ▼
          [Leader / Primary]   ← saare WRITES yaha
             /        \
      replicate      replicate
           /            \
   [Follower 1]      [Follower 2]  ← READS yaha (+ leader se bhi)
```

- **Leader (Primary):** saare **writes** accept karta hai
- **Followers (Replicas):** leader se changes copy karte hai (WAL stream — lesson 10), aur **reads** serve karte hai

```
Client writes → Leader
Client reads  → Leader OR any Follower
```

Read-heavy apps ke liye ye great hai — 1 leader writes handle kare, 10 followers reads distribute karein.

---

## 4. Synchronous vs Asynchronous replication

Ye decide karta hai: leader, follower ke update hone ka **wait karega ya nahi**, before confirming write to client.

```
SYNCHRONOUS:
  Client write → Leader writes → follower confirm ka WAIT → phir client ko success
  ✅ Strong consistency (follower hamesha up-to-date)
  ❌ Slower writes (network round-trip)
  ❌ Follower down → write bhi block ho sakta hai

ASYNCHRONOUS:
  Client write → Leader writes → TURANT client ko success (follower background me update)
  ✅ Fast writes
  ❌ Follower thodi der purana data dikha sakta hai
  ❌ Leader crash before replicating → wo writes lose ho sakte hai
```

**Most production systems (MySQL/Postgres default) async use karte hai** — speed ke liye thoda consistency trade-off.

**Semi-synchronous:** ek follower synchronous (at least ek backup guaranteed), baaki async. MySQL/Postgres practically ye use karte hai.

---

## 5. Replication Lag + "read-your-own-writes"

**Replication lag** = leader pe write hone, aur wahi change follower pe visible hone ke beech ka time delay.

```
t=0ms:   Client writes "balance=500" to Leader
t=0ms:   Leader confirms (async)
t=50ms:  Follower actually applies the update
→ lag = 50ms. Is window me follower PURANA data dikhata hai.
```

**Kyu ye real problem hai:** Agar reads follower se serve ho rahe hai, lag window me client ko **stale data** mil sakta hai:
```
User ek comment post karta hai (write → Leader)
User turant page refresh (read → Follower, lag chal raha hai)
→ User apna hi comment nahi dekh pata for a few hundred ms!
= "read-your-own-writes inconsistency"
```

**Lag kab badhta hai:** slow/overloaded follower, network issues, high write volume (follower catch-up nahi kar pa raha).

**Fixes:**
- Critical reads ("apna khud ka data") hamesha **leader se** serve karo
- "Read-your-writes": recent-write wale user ko thodi der leader se serve karo
- Monotonic reads: ek session me user ko ek hi replica pe pin karo (taaki peeche na jaye)
- Lag monitor karo; threshold cross kare toh us follower ko traffic dena band

> **CAP connection (lesson 14):** replication lag actually CAP theorem ka real-world manifestation hai — async replication choose karna = "availability + eventual consistency" (AP-leaning) choose karna.

---

## 6. Quorum (R + W > N) — leaderless replication

Leaderless systems (DynamoDB, Cassandra) me koi fixed leader nahi — client directly multiple replicas ko likhta/padhta hai.

```
N = total replicas
W = kitne replicas write confirm kare (before success)
R = kitne replicas read kare (before returning)
```

**Formula `R + W > N` overlap guarantee karta hai** → latest data milega.

**Example N=3, W=2, R=2:**
```
WRITE: Node1 ✅, Node2 ✅, Node3 (slow/timeout) → W=2 satisfied, success
READ:  Node1 + Node2 query → jo latest version hai wo return

Overlap guaranteed: write touched {N1,N2}, read touches any 2.
2+2=4 > 3 → kam se kam 1 common node (jisne latest write dekha).
```

Agar `R+W <= N` (jaise W=1,R=1,N=3) → overlap guarantee nahi → stale data possible.

**Common configs:**
| Config | Behavior | Use case |
|---|---|---|
| W=N, R=1 | All must ACK write; any 1 read | Write-heavy consistency |
| W=1, R=N | Fast writes; read all for latest | Read-heavy |
| W=2, R=2, N=3 | Balanced, tolerates 1 failure | General distributed |
| W=3, R=3, N=5 | Strong, tolerates 2 failures | Financial |

> **Frontend analogy:** 3 cache layers (browser, CDN, service worker). Sirf 1 update + sirf 1 read = stale possible. "Majority" update + "majority" read = guaranteed overlap.

---

## 7. Failover, split-brain, fencing tokens

Jab leader marta hai, ek follower ko **promote** karna padta hai (failover).

**Challenges:**
- **Split-brain:** purana leader temporarily disconnect hua tha (crash nahi), naya leader ban gaya, purana wapas aaya — ab **2 leaders**! Dono writes accept → corruption.
- **Data loss:** async replicas peeche ho sakte hai → unreplicated writes lost.
- **Stale DNS:** clients purane leader ko route karte reh sakte hai.

**Split-brain fix — fencing tokens:** har naye leader ko ek **monotonically increasing number** (fencing token) milta hai. Storage layer sirf **highest token** wale writes accept karta hai:
```
Old leader: token=1
New leader (failover): token=2
Old leader wapas aata hai, writes with token=1
→ Storage: "token=1 superseded by 2, REJECT"
New leader writes with token=2 → accepted
```

> **Frontend analogy:** search-as-you-type me purani in-flight API response ko discard karna agar ek newer request already resolve ho chuki hai (request-ID check / `AbortController`). Same idea — stale "authority" ko reject karo.

**Other fixes:** consensus protocols (Raft, Paxos) ensure sirf ek leader elect ho; short-TTL service discovery for routing.

---

## 8. Replication topologies

- **Single-leader** — ek leader (jo humne discuss kiya, most common)
- **Multi-leader** — multiple leaders, each accepts writes (multi-datacenter), but **conflict resolution** complex
- **Leaderless** — koi fixed leader nahi, quorum-based (Cassandra, DynamoDB)

### Multi-leader conflict resolution
Do leaders same data simultaneously update kare → conflict. Strategies:
- **Last-Write-Wins (LWW)** — simple but lossy (ek update gayab)
- **Application-level merge** — custom logic per data type
- **CRDTs** — conflict-free replicated data types

> **CRDT kya hai (quick):** special data structures jo is tarah design ki gayi hai ki concurrent updates **automatically, predictably merge** ho jaye. E.g., increment-only counter — har node apna delta track karta hai (`{A:+1}, {B:+1}`), merge = sum all deltas = correct +2. Merge operation mathematically commutative/associative hota hai, isliye order/source se farak nahi padta. **Google Docs/Figma-style collab editing** aise techniques use karta hai.

---

## 9. When to use / NOT

✅ **Use replication:** high availability (99.99%+), read >> write traffic, geo-distributed users, disaster recovery.
❌ **Careful:** strong consistency on every read (adds latency), **write volume is bottleneck** (replication writes ko help nahi karta — sharding chahiye), tight budget.

---

## Quick Self-Check
1. Replication ke 4 benefits? Ye storage kyu nahi badhata?
2. Leader-follower me writes aur reads kaha jaate hai?
3. Sync vs async replication — trade-offs?
4. Replication lag kya hai, aur "read-your-own-writes" problem kaise hoti hai + 2 fixes?
5. `R + W > N` kya guarantee karta hai aur kyu (overlap logic)?
6. Split-brain kya hai, fencing token isse kaise rokta hai?
7. CRDT normal "last-write-wins" se better kyu (concurrent updates)?

---
**Next:** `13-Sharding.md` — different data, different machines (scale writes + storage).