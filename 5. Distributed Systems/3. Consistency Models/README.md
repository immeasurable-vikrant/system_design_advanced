# 03 — Consistency Models (The Full Spectrum)

> **Is lesson me:** "Strong vs eventual" ke beech ka poora spectrum — linearizable, sequential, causal, read-your-writes, monotonic reads, eventual. Har ek ka exact guarantee, latency, aur kab use karo. Plus linearizability vs serializability (jo interview me confuse hota hai). Topic 2 lesson 14 me spectrum touch kiya tha; ye full depth.

Prerequisites: `02-CAP-Theorem-And-PACELC.md`

---

## 1. Consistency model kya hai

**Consistency model** = ek distributed store aur uske clients ke beech ka **contract** — ki write ke baad read **kya value** return kar sakta hai. Ye answer karta hai: "Agar main X likhu, **kab** aur **kahan** X wapas padh sakta hu?"

**Analogy:** apni profile picture update karo.
- **Strong consistency:** sab **turant** nayi picture dekhte hai — jaise ek single room me poster badalna.
- **Eventual consistency:** kuch friends thodi der purani picture dekhte hai — jaise printed photos mail karna; alag time pe pahunchti hai, but **eventually** sabke paas nayi hai.

**Ye binary nahi hai** (consistent vs inconsistent) — ek **spectrum** hai:
```
STRONG ◄──────────────────────────────────────────► WEAK
Linearizable → Sequential → Causal → Read-your-writes → Monotonic → Eventual
More guarantees, higher latency, lower availability  ←→  Fewer guarantees, faster, more available
```

---

## 2. Linearizability (strongest) ⭐

Har operation ek **single point** pe atomically execute hua lagta hai, apne start aur end ke beech. Saare clients **same order** dekhte hai, **real-time** ke respect me.
```
Client 1: Write X = 5 → ACK
(ab X=5 GLOBALLY visible)
Client 2: Read X → 5 (guaranteed, kyunki write already ACK ho chuka)
```
**Properties:**
- Ek write acknowledge hone ke baad, **har** subsequent read (kisi bhi client se) wo write dekhta hai
- Operations ka **real-time ordering** — "before"/"after" reason kar sakte ho
- Effectively "ek single copy of data" jaisa behave karta hai

**Cost:** highest latency (saare nodes ko coordinate karna padta hai). **Examples:** Google Spanner (TrueTime), ZooKeeper (sync reads), CockroachDB.

**Frontend analogy:** jaise ek single shared variable jise sab instantly, consistently dekhte hai — jaise ek single server ka in-memory value. Distributed me ye guarantee dena mehenga hai.

---

## 3. Sequential Consistency

Saare clients operations ko **same order** me dekhte hai, but wo order **real-time** respect nahi karta — sirf har client ke apne program-order ko.
```
Linearizable: agar A, B se pehle complete hua (real-time), toh A pehle ordered
Sequential:   A aur B reorder ho sakte hai agar alag clients se ho
              (bas sabko same order dikhe)
```
Thoda weaker, thoda easier. **Example:** ZooKeeper default reads (linearizable nahi jab tak `sync()` na call karo).

---

## 4. Causal Consistency

Jo operations **causally related** hai (ek ne doosre ko cause kiya), wo sab nodes same order me dekhte hai. **Concurrent** operations (no causal link) alag order me dikh sakte hai.
```
Alice: question post karti hai
Bob:   Alice ke question ka REPLY karta hai (causally after)
→ Sab dekhte hai: question PHIR reply (correct order)

Dave: ek unrelated comment post karta hai (concurrent)
→ Dave ka comment kisi bhi order me dikh sakta (koi causal link nahi)
```
**Rule:** agar A ne B ko cause kiya (B ne A padha phir likha), sab A-before-B dekhte hai. Unrelated cheezein koi bhi order.

**Tracked via:** vector clocks / Lamport timestamps (lesson 07!). **Example:** MongoDB causal consistency sessions.

**Frontend analogy:** chat me reply hamesha original message ke baad dikhna chahiye (causal). But do alag conversations ke messages koi bhi order me — unrelated hai.

---

## 5. Read-Your-Writes

Ek client hamesha **apne khud ke writes** dekhta hai. Doosre clients stale dekh sakte hai.
```
User name = "Alice" update kare (primary pe likha)
User apna profile refresh kare → "Alice" dikhna CHAHIYE (apna write)
  (chahe replica abhi purana naam dikha raha ho doosron ko)
```
> **Recap (Topic 2 lesson 12):** yaad hai "read-your-own-writes" problem — user apna comment post karke refresh kare aur wo na dikhe (replication lag)? Ye model use fix karta hai.

**Implementation:** recently-written keys ke reads primary se, ya per-user write-timestamp track karke replica se sirf tab padho jab wo catch up kar chuka ho, ya sticky sessions.

**Example:** DynamoDB strongly consistent reads, Facebook TAO.

---

## 6. Monotonic Reads

Ek baar client ne ek value padhi, wo subsequent reads me **kabhi purani value nahi** dekhega. "Time me peeche nahi jaana."
```
Bina monotonic reads:
  User Replica A se padhe → 10 comments dikhe
  User refresh kare, Replica B (lagging) pe jaye → sirf 8 comments
  → User ko lagta comments GAYAB ho gaye!
Monotonic reads: ye kabhi nahi hoga (peeche nahi jaayega)
```
**Solution:** user ko ek session ke liye ek hi replica pe pin karo, ya read-version track karke sirf us-version-ya-newer replicas se serve karo.

---

## 7. Eventual Consistency (weakest)

Agar naye writes ruk jaye, **eventually** saare replicas same value pe converge kar jaayenge. **Kab** ya **kis order** me clients updates dekhenge — koi guarantee nahi.

| Guarantee | Eventual consistency deta hai? |
|---|---|
| Convergence | Yes — sab eventually agree |
| Read freshness | No guarantee |
| Ordering | No guarantee |
| Duration | Typically ms to seconds |

**Cost:** lowest latency, highest availability. **Examples:** DNS propagation, Cassandra, DynamoDB default reads.

> **Recap (Topic 2 lesson 14 — BASE):** eventual consistency BASE ka "E" hai. Replication lag (Topic 2 lesson 12) ka window = "not yet consistent"; lag khatam = "eventually consistent."

---

## 8. Comparison + when to use

| Model | Ordering | Latency | Example |
|---|---|---|---|
| Linearizable | Global real-time | High | Spanner, CockroachDB |
| Sequential | Same order for all | Med-High | ZooKeeper |
| Causal | Cause-effect respected | Medium | MongoDB sessions |
| Read-your-writes | Own writes visible | Low-Med | DynamoDB consistent read |
| Monotonic reads | No backward time | Low | Session pinning |
| Eventual | None (converges) | Lowest | Cassandra, DynamoDB default |

| Use case | Model | Kyu |
|---|---|---|
| Bank transfers | Linearizable | Phantom money nahi |
| Social feed | Eventual | Thoda staleness ok |
| Chat messages | Causal | Causal order me dikhein |
| Shopping cart | Read-your-writes | User apne additions dekhe |
| Distributed lock | Linearizable | Globally agree hona zaroori |
| Inventory count | Linearizable | Oversell unacceptable |

**Interview me sirf ye 3 kaafi hai:** **Strong (linearizable)** money/critical ke liye, **Eventual** baaki sab ke liye, **Read-your-writes** compromise (write ke baad primary se, warna replica se).

---

## 9. Linearizability vs Serializability (classic confusion) ⭐

Ye do alag cheezein hai, log mix karte hai:
- **Linearizability** = **single-object, real-time ordering.** Har read latest write dekhta hai globally. (Consistency model — ye lesson.)
- **Serializability** = **multi-object transactions.** Result kisi **serial execution order** ke equivalent hai. (Isolation level — Topic 2 lesson 09!)

```
Linearizability: "ek object pe operations real-time order me"
Serializability:  "multiple transactions ka net effect = koi serial order"
Strict serializability (Spanner): DONO — serial AND real-time order
```

> **Recap (Topic 2 lesson 09):** Serializable isolation level — transactions aise chale jaise ek-ek karke. Wo **serializability** hai. **Linearizability** ek alag guarantee hai (single-object recency). Interview me clarify karo kaunsa.

---

## Quick Self-Check
1. Consistency model kya hai (contract)? Profile-picture analogy se strong vs eventual?
2. Linearizability ka guarantee — write ACK ke baad reads?
3. Causal consistency kya preserve karta hai (reply-after-question), kya nahi?
4. Read-your-writes kis Topic-2 problem (replication lag) ko fix karta hai?
5. Monotonic reads kya rokta hai (backward time-travel)?
6. Eventual consistency kya guarantee karta hai (convergence) aur kya nahi (freshness/order)?
7. Interview ke liye kaunse 3 models kaafi hai?
8. **Linearizability vs serializability** — difference (single-object vs multi-txn)?

---
**Next:** `04-Leader-Election-And-Consensus.md` — nodes me se ek coordinator chunna.