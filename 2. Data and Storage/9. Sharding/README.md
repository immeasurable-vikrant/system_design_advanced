# 03 — Database Sharding: Terms Explained (Companion to the Article)

> Ye file "Database Sharding - Complete Deep Dive" article ke saath side-by-side padhna. Jo bhi terms wo article assume kar leta hai (cardinality, scatter-gather, saga pattern, two-phase commit, etc.) unko yaha se-scratch explain kar raha hu, with frontend-relatable analogies jaha possible ho.

Prerequisites: `01-Hashing-Basics-Explained.md`, `02-Consistent-Hashing-Deep-Dive.md`
Pairs with: "Database Sharding - Complete Deep Dive" article

---

## 1. "High Cardinality" ka matlab kya hai?

Article me likha: *"High cardinality (many unique values — userId is good, country is bad)"*

**Cardinality** = kitne **unique/distinct values** ek column me ho sakte hai.

```
Column: country
Possible values: "India", "USA", "UK", "Germany", ... (~195 countries)
→ LOW cardinality (kam unique values, बहुत reuse hota hai)

Column: userId
Possible values: 1, 2, 3, ..., 100 million (har user ka apna unique ID)
→ HIGH cardinality (bahut zyada unique values)
```

**Shard key ke liye high cardinality kyu important hai?**

Agar tum `country` ko shard key banao, aur tumhare paas 10 shards hai, but sirf 195 possible values (countries) hai — aur unme se 60% users sirf "India" aur "USA" se hai — toh do shards **overloaded** ho jayenge aur baaki 8 shards mostly empty rahenge. Load evenly distribute hi nahi ho payega, chahe tum kitne bhi shards bana lo.

`userId` jaisa high-cardinality column use karne se, har unique value alag-alag "bucket" me evenly spread ho sakta hai (especially hashing ke saath).

**Frontend analogy:** Socho tum ek list ko `groupBy` kar rahe ho React me. Agar tum `groupBy(user => user.country)` karo, tumhare paas sirf ~195 groups honge (low cardinality) — kuch groups bahut bade honge. Agar `groupBy(user => user.id)` karo, har group me literally 1 hi item hoga (high cardinality) — perfectly even, but sharding ke liye ye extreme bhi useless hai (that's why we use `hash(userId) % N`, not userId directly, taaki multiple users same shard me evenly aaye).

---

## 2. "Scatter-Gather" pattern kya hai?

Article: *"Query ALL shards → each returns its top 10 → merge → pick global top 10 → Scatter-gather pattern"*

Ye ek simple hi cheez hai jiska naam heavy lagta hai:

```
Query: "Top 10 orders across ALL users, sorted by amount"

Tumhara data 4 shards me split hai. Kisi ek shard ko pata nahi
baaki shards me kya hai. Toh karna kya padega:

Step 1 (SCATTER):
  Query bhejo Shard1, Shard2, Shard3, Shard4 — sabko PARALLEL me
  "give me YOUR top 10 orders"

Step 2: Har shard apna local top 10 return karta hai
  Shard1 → [order A: $500, order B: $450, ...]
  Shard2 → [order C: $600, order D: $300, ...]
  Shard3 → [order E: $200, ...]
  Shard4 → [order F: $700, ...]

Step 3 (GATHER):
  Application layer pe saare results ko combine karo,
  phir se sort karo, aur global top 10 nikaalo
  → [F:$700, C:$600, A:$500, B:$450, ...]
```

**Frontend analogy:** Ye bilkul waisa hai jaise tum `Promise.all()` se 4 alag APIs ko parallel call karo, phir unke responses ko `.flat().sort()` karke merge karo. Same concept, bas yaha "APIs" = "database shards" hai.

**Problem:** Response time = **sabse slowest shard jitna time le**, plus merge karne ka extra overhead. Agar 4 me se ek shard slow/down ho, poori query slow ho jaati hai.

---

## 3. "Denormalize" kya hota hai (in this context)?

Article: *"Denormalize — Store duplicate data to avoid cross-shard JOINs → Write amplification"*

Normal (relational, non-denormalized) design me tum data ko **ek hi jagah** store karte ho, aur JOIN karke combine karte ho:

```
Table: orders        Table: users
┌────────┬─────────┐ ┌────────┬──────┐
│orderId │ userId  │ │userId  │ name │
├────────┼─────────┤ ├────────┼──────┤
│ 1      │ 101     │ │ 101    │ Bob  │
└────────┴─────────┘ └────────┴──────┘

Query "order with user name" → JOIN orders + users on userId
```

Ye tab tak fine hai jab tak dono tables **same shard** pe ho. Agar `orders` shard by `orderId` ho aur `users` alag shard pe ho — JOIN cross-shard ho jayega (expensive/impossible directly).

**Denormalization ka solution:** User ka naam **already copy karke** orders table me hi rakh do — duplicate data, but ab JOIN ki zarurat nahi:

```
Table: orders (denormalized)
┌────────┬─────────┬───────────┐
│orderId │ userId  │ userName  │  ← "Bob" yaha duplicate copy hai
├────────┼─────────┼───────────┤
│ 1      │ 101     │ Bob       │
└────────┴─────────┴───────────┘
```

**"Write amplification"** matlab: ab agar Bob apna naam change karke "Robert" kare, tumhe **saari jagah** update karna padega jaha bhi "Bob" copy kiya tha (potentially thousands of order rows) — ek hi jagah update karne ke bajaye, ek write **kayi writes** me amplify ho gaya. Ye trade-off hai: fast reads (no JOIN) vs expensive/complex writes.

**Frontend analogy:** Ye bilkul waisa hai jaise Redux/global state me tum `user.name` ko multiple components ke local state me bhi copy kar lo taaki har component apna prop-drilling na kare — but ab agar naam change ho, tumhe har jagah manually sync karna padega (bug-prone), instead of single source of truth se auto-update hona.

---

## 4. "Saga Pattern" aur "Two-Phase Commit" kya hai?

Article: *"Options: 1) Design so transactions are single-shard 2) Use saga pattern 3) Use two-phase commit (slow, avoid)"*

Pehle samjho problem: Ek normal (single-DB) transaction me tum multiple steps ek saath "all or nothing" kar sakte ho:

```
BEGIN TRANSACTION
  deduct $100 from Account A
  add $100 to Account B
COMMIT  -- dono ho gaya, ya dono nahi hua (atomic)
```

Agar `Account A` Shard1 pe hai aur `Account B` Shard2 pe — ye "all or nothing" guarantee dena **hard** ho jata hai, kyunki 2 alag independent databases hai.

**Two-Phase Commit (2PC)** — ek coordinator, dono shards se "commit karne ke liye ready ho?" pooch ke confirm leta hai, phir dono ko "ab commit karo" bolta hai:

```
Coordinator → Shard1: "Ready to commit?" → Shard1: "Yes" (locks the row)
Coordinator → Shard2: "Ready to commit?" → Shard2: "Yes" (locks the row)
Coordinator → both: "COMMIT NOW"
```

Problem: Dono shards ko **lock hold karna padta hai** jab tak coordinator confirm na kare — agar coordinator crash ho jaye beech me, shards locked hi reh jaate hai (slow, fragile, that's why article says "slow, avoid").

**Saga Pattern** — iske bajaye, transaction ko **chhoti independent steps** me todo, aur har step ka ek "undo/compensating action" define karo:

```
Step 1: Deduct $100 from Account A  →  succeeds
Step 2: Add $100 to Account B       →  FAILS (e.g., account doesn't exist)

Compensating action:
Step 1 ka undo: Add $100 BACK to Account A (refund)
```

Ye "eventually consistent" approach hai — beech me thodi der ke liye system inconsistent state me hota hai (Account A se paisa kat gaya but Account B me nahi pahuncha), but har failure ka ek **defined rollback/compensation step** hota hai jo system ko wapas consistent state me le aata hai. Ye 2PC se zyada scalable hai kyunki koi long locks nahi hote, but application code me zyada complexity aati hai (har step ka undo likhna padta hai).

**Frontend analogy:** Ye bilkul waisa hai jaise multi-step form submission me — agar step 3 fail ho jaye (jaise payment fail), tum step 1-2 ka data automatically "rollback"/clear kar do (jaise reserved inventory wapas release karna), instead of ek hi giant atomic transaction try karna.

---

## 5. Recap Table — Sharding article ke saare "assumed knowledge" terms

| Term | Simple meaning |
|---|---|
| Cardinality | Kitne unique values ek column me ho sakte hai |
| Scatter-gather | Sabhi shards ko parallel query bhejo, results merge karo |
| Denormalize | Duplicate data store karo taaki cross-shard JOIN avoid ho |
| Write amplification | Ek logical update, multiple physical writes me badal jata hai (duplication ki wajah se) |
| Two-Phase Commit | Coordinator dono shards se confirm leta hai before committing (safe but slow) |
| Saga Pattern | Transaction ko chhote steps + rollback/compensating actions me todna |
| Hot partition | Ek shard pe disproportionate traffic (uneven load) |

---

## Quick Self-Check
1. High cardinality kyu zaroori hai shard key ke liye?
2. Scatter-gather pattern me response time kis cheez pe depend karta hai?
3. Denormalization se konsa problem solve hota hai, aur uska trade-off kya hai?
4. Two-Phase Commit slow kyu hota hai?
5. Saga pattern 2PC se kaise different hai?