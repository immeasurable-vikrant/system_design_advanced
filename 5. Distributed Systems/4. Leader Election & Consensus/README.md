# 04 — Leader Election & Consensus

> **Is lesson me:** Kai nodes me se ek "leader" (coordinator) kaise chuna jaata hai — Bully algorithm, Raft (industry standard), ZooKeeper. Plus consensus kya hai, distributed locking, aur split-brain problem + solutions. Topic 2 lesson 12 me leader-follower + split-brain touch kiya tha; yahan election mechanics + consensus.

Prerequisites: `01-Why-Distributed-Systems-Are-Hard.md`, `02-CAP`

---

## 1. Leader kyu chahiye

**Leader election** = ek distributed system me ek node ko **coordinator (leader)** chunna, baaki **followers** hote hai. Leader decisions leta hai, kaam assign karta hai, ya writes handle karta hai — taaki multiple nodes independently act karke conflicts na banaye.

**Analogy:** ek classroom group project. Bina leader ke, sab ek hi section pe kaam karte hai (conflict) ya kuch parts koi nahi karta (gaps). Ek team-lead elect karo → ek banda tasks assign kare, progress track kare, conflicts resolve kare. Lead absent ho → group naya chun le.

```
Bina leader:
  Node A: "Main job 42 process karunga"
  Node B: "Main job 42 process karunga"   ← DUPLICATE work!
  Node C: "Main job 42 process karunga"

Leader ke saath:
  Leader: "Node A → job 42, Node B → job 43, Node C → job 44"
  Followers: sirf assigned kaam karein
```

**Key use cases:** database primary (saare writes handle kare — Topic 2 lesson 12), job scheduler coordinator, distributed lock manager, Kafka partition leader.

---

## 2. Consensus kya hai (foundation)

> **Recap (lesson 01):** consensus = kai nodes ko ek value pe agree karana despite failures. Leader election ek **consensus problem** hai — "kaun leader hai" pe sabko agree karna hai.

Consensus algorithms (Raft, Paxos) ye guarantee karte hai:
- **Agreement:** saare healthy nodes same value pe agree karein
- **Validity:** agreed value kisi node ne actually propose ki thi
- **Termination:** eventually decision ho jaye

Ye distributed systems ka core building block hai — database replication, config management (etcd), locking — sab consensus pe khade hai.

---

## 3. Distributed Locking (quick — prerequisite)

Kabhi-kabhi tumhe ek resource pe **mutual exclusion** chahiye across nodes — ek time pe sirf ek node use kare. Ye **distributed lock** hai.
```
Node A: acquire lock "resource-X" → granted (TTL=30s)
Node B: acquire lock "resource-X" → wait (A ke paas hai)
Node A: kaam karke release → ab B ko mil sakta
```
Implementation: Redis (Redlock), ZooKeeper, etcd, ya database advisory locks.

> ⚠️ **Distributed locks ke saath ek deep problem hai** — TTL expire ho jaye but holder ko pata na chale (GC pause) → wo stale write kar de. Ye exactly fencing tokens (lesson 05) solve karte hai. Abhi bas samjho: distributed lock = cross-node mutual exclusion, but tricky.

---

## 4. Approach 1 — Bully Algorithm (simplest)

Sabse **highest ID** (ya priority) wala node hamesha leader banta hai.
```
Nodes: [N1, N2, N3, N4, N5]  (N5 = highest priority)
N5 leader hai. N5 crash ho gaya.

N3 detect kare N5 down (missed heartbeat)
N3 → ELECTION bheje higher nodes (N4, N5) ko
N4 → "Main zinda hu, main le lunga"
N4 → ELECTION bheje N5 ko (no response, dead)
N4 → khud ko leader declare kare
N4 → COORDINATOR message sabko: "Main naya leader"
```
**Pros:** simple, deterministic. **Cons:** highest-ID hamesha jeetta (chahe slow ho); flaky top-node = frequent elections.

---

## 5. Approach 2 — Raft (industry standard) ⭐

Leader **majority vote** se elect hota hai. **Terms** (epochs) aur **randomized timeouts** use karta hai conflicts avoid karne ko.

```
1. Election timeout expire (random 150-300ms)
2. Follower → CANDIDATE bane
3. Apna term increment kare, khud ko vote de
4. Baaki nodes se vote maange
5. Nodes YES vote de agar: is term me vote nahi diya AND candidate ka log up-to-date
6. Majority mile → LEADER bane
7. Heartbeats bheje (leadership maintain kare)
```

**Example (5 nodes):**
```
Term 1: N1 leader, heartbeats bhej raha
Term 1: N1 crash (heartbeats ruk gaye)
...150ms...
Term 2: N3 ka election timeout pehle fire hua (random!)
Term 2: N3 → Candidate, khud ko vote (1/5)
Term 2: N3 baaki se vote maange → N2 YES (2/5), N4 YES (3/5) ← MAJORITY!
Term 2: N3 ab leader, heartbeats bhejta hai
```

**Randomized timeouts kyu?** Taaki saare nodes ek saath election shuru na karein (jo votes split kar deta aur election delay karta). Random timeout se ek node pehle fire karta hai, wahi jeet jaata hai.

**Majority (quorum) kyu?** > **Recap (Topic 2 lesson 12):** quorum = majority. 5 nodes me 3+ votes chahiye. Isse **split-brain impossible** ho jaata hai — do partitions dono majority nahi paa sakte.

**Used by:** etcd (Kubernetes ka core), CockroachDB, TiKV, MongoDB (modified Raft), Kafka KRaft.

---

## 6. Approach 3 — ZooKeeper Ephemeral Nodes

ZooKeeper ke coordination primitives use karta hai. Nodes **ephemeral sequential znodes** banate hai — **lowest sequence number** leader banta hai.
```
/election/
  ├── node_0000000001 (Server A) ← LEADER (lowest)
  ├── node_0000000002 (Server B)
  └── node_0000000003 (Server C)

Har server ek ephemeral sequential znode banata hai
"Kya main lowest hu?" → HAAN → leader; NAHI → apne se just pehle wale ko WATCH karo
Leader crash → ephemeral node auto-delete (session died)
Next node deletion dekhta hai → naya leader
```
- **Ephemeral** = session marne pe (crash/disconnect) auto-delete
- **Sequential** = ZooKeeper incrementing numbers deta hai

**Used by:** Kafka (legacy), HBase, Hadoop.

---

## 7. Comparison

| Feature | Bully | Raft | ZooKeeper |
|---|---|---|---|
| Complexity | Low | High | Medium (ZK use karta) |
| Consistency | Weak | Strong (linearizable) | Strong (ZAB) |
| Split-brain safe? | No | **Yes (quorum)** | **Yes (quorum)** |
| External dependency | None | None | ZK cluster chahiye |
| Used by | Simple systems | etcd, CockroachDB | Kafka, HBase, Hadoop |

**Interview:** *"Naya distributed system scratch se → Raft embed karo (no external dependency, etcd/CockroachDB use karte hai). Already ZooKeeper hai infra me (Kafka/Hadoop) → ZK use karo. Bully sirf simple systems ke liye."*

---

## 8. Split-Brain Problem ⭐ (the killer)

Sabse khatarnak failure: **do nodes dono khud ko leader samajhte hai.**
```
Partition A: N1 (purana leader, abhi bhi khud ko leader samajhta) + N2
Partition B: N3 (naya leader elect hua) + N4 + N5
        ── NETWORK PARTITION (baat nahi ho paa rahi) ──
DANGER: dono leaders writes accept kar rahe → DATA CORRUPTION!
```
> **Recap (Topic 2 lesson 12):** split-brain — purana leader temporarily disconnect hua, naya ban gaya, purana wapas aaya → 2 leaders. Yahan iske solutions detail me:

**Solutions:**
1. **Majority Quorum** (Raft/ZK) — 5 nodes, 3+ votes chahiye leader banne ko. Partition A (2 nodes) majority nahi paa sakta → purana leader step down. Partition B (3 nodes) majority → naya leader. **Sirf ek partition me leader ho sakta hai.**
2. **Fencing Tokens** (lesson 05) — har leader ko incrementing token. Storage sirf highest token wale writes accept kare. Purana leader (low token) ke writes reject.
3. **Lease-Based** — leader ek time-limited lease rakhe (30s). Renew na kar paye (partition) → lease expire → step down. Naya leader sirf purani lease expire hone ke baad.

---

## Quick Self-Check
1. Leader kyu chahiye — group-project analogy se, aur "job 42" duplicate problem?
2. Consensus kya hai, aur leader election kaise ek consensus problem hai?
3. Distributed lock kya hai, aur uski deep problem (TTL expire) kya?
4. Bully algorithm kaise kaam karta hai + con?
5. Raft me leader kaise elect hota hai (terms, majority)? Randomized timeouts kyu?
6. ZooKeeper ephemeral+sequential znodes se leader kaise (lowest number)?
7. Split-brain kya hai, aur quorum ise kaise rokta hai?

---
**Next:** `05-Fencing-Tokens.md` — expired leaders/locks se data bachana.