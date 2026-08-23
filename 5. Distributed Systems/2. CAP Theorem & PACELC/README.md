# 02 — CAP Theorem & PACELC

> **Is lesson me:** CAP theorem — teen properties (Consistency, Availability, Partition tolerance), kyu sirf 2 mil sakti hai, CP vs AP systems, aur PACELC (jo CAP ko extend karta hai — normal operation me bhi trade-off). Topic 2 lesson 14 me basics dekhe the; ye full depth + per-component framing.

Prerequisites: `01-Why-Distributed-Systems-Are-Hard.md`

---

## 1. CAP theorem

**CAP theorem** kehta hai: ek distributed system ek saath in teen me se **sirf 2** guarantee kar sakta hai:

```
C — Consistency:  har read latest write dekhega (saare nodes same data, same time)
A — Availability: har request ko response milega (chahe stale data)
P — Partition tolerance: network toot jaye (nodes baat na kar paaye) toh bhi system chale
```

**The catch:** distributed system me **network partitions HONGI** (routers fail, cables toot te, cloud AZs connectivity kho dete hai — lesson 01 ka "unreliable network"). Toh **P optional nahi hai** — tumhe P rakhna hi padega.

Iska matlab real choice hai: **C ya A (partition ke DAURAN).**

```
Normal operation me:  teeno milte hai (C + A + P)
Network partition me:  C YA A choose karo:

  Choose C: system stale data serve karne se INKAAR kare → kuch requests FAIL
  Choose A: system jo data hai wo serve kare → kuch responses STALE ho sakte hai
```

> **Recap (Topic 2 lesson 14):** ye wahi CAP hai jo BASE aur consistency spectrum me dekha tha. Key insight repeat: **CAP sirf partition ke time bite karta hai** — normal operation me tum dono achhe se de sakte ho.

---

## 2. The bank branch analogy

Do bank branches (Node A, Node B) jo account data share karte hai. Beech ki phone line toot gayi (partition).

**Consistency choose karo (CP):**
```
Branch B pe customer ₹500 withdraw karna chahta hai
Branch B, Branch A se balance verify nahi kar sakti (partition)
Branch B: "Sorry, system unavailable. Baad me aayein."
→ Koi galat transaction nahi, but service down.
```

**Availability choose karo (AP):**
```
Branch B pe customer ₹500 withdraw karna chahta hai
Branch B verify nahi kar sakti, but last known balance se serve kar deti hai
Customer ₹500 nikaal leta hai
Meanwhile Branch A pe bhi customer ₹500 nikaalta hai (balance already ₹500 tha)
→ Total ₹1000 nikaal liye ₹500 ke account se! (inconsistency)
But dono branches "available" rahe.
```

Ye exactly woh double-spend hai jo Topic 2 lesson 09 (write skew) me dekha tha — CAP uska distributed-systems framing hai.

---

## 3. CP vs AP systems

### CP (Consistency + Partition tolerance)
Partition ke dacoran, system **unavailable** ho jaata hai stale data serve karne ke bajaye.

| System | Kyu CP |
|---|---|
| Postgres (single primary) | Writes ek node pe; primary unreachable → writes fail |
| MongoDB (default) | Writes primary pe; primary down → writes fail until election |
| HBase | Strong consistency, single region server per row range |
| ZooKeeper / etcd | Consensus-based; majority na mile toh reads bhi refuse |

**Kab choose:** financial transactions, inventory, koi bhi system jaha **galat data, no data se bura** hai.

### AP (Availability + Partition tolerance)
Partition ke daoran, system requests serve karta rehta hai but **stale data** de sakta hai.

| System | Kyu AP |
|---|---|
| Cassandra | Kisi bhi node pe writable; conflicts baad me resolve (LWW) |
| DynamoDB | Configurable; default eventually consistent (AP) |
| DNS | Cached (possibly stale) records; hamesha available |
| CDN | Cached content serve kare origin down ho toh bhi |

**Kab choose:** social feeds, product catalogs, DNS, sessions — jaha stale data kuch seconds ke liye acceptable hai.

---

## 4. PACELC — CAP ka extension (interview edge) ⭐

CAP sirf partition ke baare me baat karta hai. **PACELC** aage jaata hai: **normal operation me kya?**

```
PAC: Partition ke time → Availability YA Consistency
ELC: Else (no partition) → Latency YA Consistency
```

Matlab: **partition na ho tab bhi**, tum latency ko consistency ke liye trade karte ho. Strong consistency ke liye saare replicas ka acknowledge ka wait karna padta hai (slow). Eventual consistency turant return karti hai (fast) but stale ho sakti hai.

| System | Partition ke time | Normal operation |
|---|---|---|
| Postgres | PC (unavailable) | EC (consistent, sync replication me higher latency) |
| Cassandra | PA (available, stale) | EL (low latency, eventual) |
| DynamoDB | PA/PC (configurable) | EL/EC (per-read configurable) |
| MongoDB | PC (writes fail) | EC (consistent reads from primary) |

**Insight:** partition rare hai, but latency-vs-consistency trade-off **har request pe** hota hai. Isliye PACELC practically zyada relevant hai — ye batata hai ki strong consistency **hamesha** thodi latency cost pe aati hai (replicas ka wait), sirf partition ke time nahi.

> **Recap (Topic 2 lesson 12):** sync replication = wait for followers = strong but slow (EC in PACELC). Async replication = fast but stale (EL). PACELC isko formalize karta hai.

---

## 5. The key interview insight — per component, not whole system ⭐

**Galat:** "Mera system CP hai" (too simplistic).

**Sahi:** *"Payment/wallet path ke liye strong consistency (single Postgres primary, writes ke baad reads primary se) — CP. Feed/notification path ke liye eventual consistency with read replicas — AP, 1-2s staleness accept karke. Alag components ki alag consistency needs hoti hai."*

Tum poore system ke liye CP ya AP nahi choose karte — **per component** choose karte ho, us component ke kaam ke hisaab se:

```
Transaction ledger  → CP (paisa lose/duplicate nahi kar sakte)
User feed           → AP (post 2s late dikhe, chalega)
Notifications       → AP (eventually deliver, complete fail se better)
Shopping cart       → AP (available rahe, merge later)
Distributed lock    → CP (globally agree hona zaroori)
Inventory count     → CP (oversell unacceptable)
```

Yehi Topic 2 lesson 14 ka "polyglot" + per-access-pattern decision hai, CAP ke lens se.

---

## Quick Self-Check
1. CAP ke teen properties? P optional kyu nahi hai?
2. Partition ke daoran real choice kya hai (C ya A)? Normal operation me?
3. Bank branch analogy se CP vs AP behavior?
4. CP system aur AP system ke 2-2 examples + kab choose?
5. **PACELC** kya add karta hai (normal operation trade-off)? Ye kyu practically zyada relevant?
6. "Mera system CP hai" kyu galat hai — sahi framing kya (per-component)?
7. Ek chat system ke alag parts ke liye alag consistency — example?

---
**Next:** `03-Consistency-Models.md` — "strong vs eventual" ke beech ka poora spectrum.