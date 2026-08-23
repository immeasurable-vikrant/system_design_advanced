# 07 — Vector Clocks (Causality & Conflict Detection)

> **Is lesson me:** Distributed systems me events ka order kaise track karo jab koi global clock nahi (lesson 01) — Lamport timestamps (precursor), vector clocks (the real thing), causality detect karna (happened-before vs concurrent), aur conflict resolution. Ye leaderless replication (lesson 09) me conflicts detect karne ka tool hai.

Prerequisites: `01-Why-Distributed-Systems-Are-Hard.md` (no global clock), `03-Consistency-Models` (causal)

---

## 1. Vector clocks kya hai

**Vector clocks** = distributed system me events ke beech **causality** track karne ka mechanism. Ye answer karte hai: "Kya event A, event B se pehle hua, ya wo concurrent the?"

**Analogy:** teen dost mail se collaborative story likh rahe hai. Har ek apne bheje letters ka counter rakhta hai. Alice jab letter bheje, apna counter AUR Bob/Carol ke latest dekhe counters include kare. Bob ko Alice se `[Alice:3, Bob:1, Carol:2]` mile → wo jaanta hai Alice ne Bob ka pehla letter aur Carol ke pehle 2 dekh liye. Agar baad me `[Alice:2, Bob:0, Carol:1]` mile → wo letter actually **pehle** likha gaya tha (past se hai, naya conflicting version nahi).

---

## 2. The problem — physical clocks lie (recap)

> **Recap (lesson 01):** clock skew — machines ki clocks 10-200ms drift karti hai (NTP ke saath bhi). Isliye:
```
Node A: write X=1 at T=100ms
Node B: write X=2 at T=120ms
A pehle hua? Agar A ki clock 50ms fast hai → real order B→A hai!
Physical timestamps DISTRIBUTED me jhooth bolte hai.
```
Isliye humein **logical clocks** chahiye jo message-passing se order track karein, wall-clock se nahi.

---

## 3. Lamport Timestamps (precursor)

Leslie Lamport ka logical clock — **per node ek single counter.**
```
Rules:
  1. Har event se pehle apna counter increment karo
  2. Message bhejte waqt apna counter include karo
  3. Message receive karte waqt: counter = max(yours, received) + 1
```
```
Node A: counter=1 → msg bheje (ts=1)
Node B: receive → counter = max(0,1)+1 = 2 → local event → counter=3
```

**Limitation:** agar `L(A) < L(B)`, tum jaante ho A **shayad** B se pehle hua. But tum "A ne B ko cause kiya" ko "A aur B concurrent the" se **distinguish nahi kar sakte.** Total order milta hai, but concurrency info kho jaati hai. Ye vector clocks fix karte hai.

---

## 4. Vector Clocks — how they work ⭐

Har node ke liye ek **array of counters** — system ke har node ka ek entry.
```
N nodes ke liye: VC = [count_node1, count_node2, ..., count_nodeN]

Rules:
  1. Node i pe har event se pehle: VC[i]++ (apna entry badhao)
  2. Message bhejte waqt: poora vector include karo
  3. Node j se message receive karte waqt:
       har entry k ke liye: VC[k] = max(VC[k], received_VC[k])
       phir VC[i]++ (apna badhao)
```
```
Node A: VC=[1,0,0] → msg bheje [1,0,0]
Node B: receive → VC=[1,1,0] → local event → VC=[1,2,0]
Node C: local event → VC=[0,0,1]... (apne events track karta)
```

---

## 5. Detecting relationships (the whole point)

Do vector clocks VC_A aur VC_B compare karke:

| Relationship | Condition | Matlab |
|---|---|---|
| **A happened-before B** | VC_A ka har entry ≤ VC_B ka, aur kam se kam ek strictly < | A causally B se pehle |
| **B happened-before A** | Ulta (VC_B ka har entry ≤ VC_A ka) | B causally A se pehle |
| **Concurrent** | Neither dominates | A, B independent — **CONFLICT** |

```
[2,1,0] vs [3,2,1] → first happened-before second (har entry ≤) → ordered
[2,1,0] vs [1,2,0] → concurrent! (A[0]=2>1 but A[1]=1<2 — neither dominates) → CONFLICT
[2,1,0] vs [2,1,0] → same event
```

**Ye killer feature hai:** vector clocks **concurrent (conflicting) writes ko detect** kar sakte hai — jo Lamport timestamps nahi kar sakte. Leaderless systems (lesson 09) me jaha do nodes same key ek saath likh de, ye conflict pakadte hai.

**Frontend analogy:** socho collaborative editing me har edit apne saath "maine kaunse edits dekhe" ka snapshot le. Do edits jinhone ek-dusre ko nahi dekha = concurrent = conflict = merge chahiye. Vector clocks isko formalize karte hai.

---

## 6. Conflict resolution — concurrent detect hone ke baad

Jab vector clocks concurrent writes detect kare, system ko conflict resolve karna padta hai:

| Strategy | Kaise | Used by |
|---|---|---|
| **Last-Writer-Wins (LWW)** | Physical timestamp se winner pick, loser discard | Cassandra, DynamoDB (default) |
| **Client-side resolution** | Saare conflicting versions client ko return, client merge kare | Riak (siblings) |
| **CRDTs** | Data structures jo automatically conflict-free merge ho | Riak 2.0, Redis CRDT |
| **Application merge** | Domain-specific logic (sets ka union, counters ka max) | Custom |

> **Recap (Topic 2 lesson 12):** CRDTs aur LWW dekhe the multi-leader conflict me. Yahan vector clocks wo mechanism hai jo pehle **detect** karta hai ki conflict hai, phir ye strategies resolve karti hai.

### Shopping cart example (classic)
```
User phone pe milk add kare:  VC=[2,0]
User laptop pe bread add kare: VC=[0,2]  (concurrent — neither dominates!)
→ System dono siblings store kare
→ Next read pe client dono versions dekhe: {milk}, {bread}
→ Merge strategy = UNION → cart = {milk, bread}
→ Merged version naya VC=[2,2] (dono siblings ko dominate karta)
```
Shopping cart ke liye union safe hai (item lose nahi hota). LWW yahan bura hoga (ek item gayab). Isliye conflict resolution **domain-specific** hoti hai.

---

## 7. Real-world + limitations

**DynamoDB:** simplified vector clocks conflict detection ke liye; practically LWW prefer karta hai simplicity ke liye.
**Riak:** canonical example — concurrent writes = siblings, client merge kare.

> ⚠️ **Vector clocks ki cost:** size O(N nodes). 1000+ nodes pe vector bahut bada ho jaata hai. Riak ne "clock pruning" kiya (purane entries hatao) jo false conflicts create kar sakta hai. Isliye vector clocks **leaderless, moderate-node** systems ke liye best hai — massive clusters ke liye nahi.

**Kab use:** multiple nodes same key concurrently likh sakte ho (leaderless), conflicts **detect** karne hai (silently overwrite nahi), application merge kar sakti ho.
**Kab nahi:** single-leader (no concurrent writes by design), LWW acceptable ho, ya bahut zyada nodes ho.

---

## Quick Self-Check
1. Vector clocks kya answer karte hai (A before B, ya concurrent)?
2. Physical clocks distributed me kyu use nahi kar sakte (lesson 01 recap)?
3. Lamport timestamps ki limitation kya (concurrency distinguish nahi kar sakte)?
4. Vector clock ke 3 rules (increment, send, receive)?
5. "Happened-before" vs "concurrent" — vector clocks se kaise detect (dominance)?
6. Vector clocks ka killer feature kya (conflict detection)?
7. Shopping cart me concurrent adds kaise resolve (union, not LWW)?
8. Vector clocks ki cost/limitation (O(N nodes))?

---
**Next:** `08-Merkle-Trees.md` — out-of-sync replicas ko efficiently sync karna.