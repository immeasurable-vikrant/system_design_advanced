# 01 — Why Distributed Systems Are Hard

> **Is lesson me:** Distributed systems ki 3 fundamental problems — partial failure, unreliable network (asynchrony), aur no global clock — aur "consensus" kya hota hai. Ye poore module ka foundation hai; ye samajhe bina baaki sab (CAP, vector clocks, leader election) sirf ratna ban jaayega.

Prerequisites: Topics 1-4 helpful (especially Topic 2 replication). Warna bhi chalega.

---

## 1. Distributed system kya hai

**Distributed system** = kai independent machines (nodes) jo network pe baat karke ek single system ki tarah kaam karti hai. Har node ki apni memory/disk hai — koi **shared memory nahi.** Wo sirf **messages** bhej ke coordinate karte hai.

**Frontend analogy:** socho ek Google Doc jise 5 log ek saath edit kar rahe hai, alag laptops se. Koi single "source of truth" nahi jise sab instantly dekhte ho — har laptop ka apna view hai, aur wo network pe changes sync karte hai. Ab socho beech me kisi ka internet chala jaye, ya ek edit doosre se late pahunche — kya hoga? Yehi distributed systems ki saari complexity hai.

**Ek single machine (jaise tumhara laptop) me:** memory shared hai, ek clock hai, aur agar program crash kare toh **poora** crash karta hai (clean). Distributed me ye teeno guarantees toot jaati hai — aur wahi 3 core problems hai.

---

## 2. Problem 1 — Partial Failure (the defining problem) ⭐

Single machine: kaam karti hai ya nahi (binary). Distributed system: **kuch hisse kaam kar rahe, kuch nahi** — aur tumhe **pata nahi** kaunse.

```
Node A ne Node B ko request bheji, response nahi aaya. Kya hua?
  - B crash ho gaya?
  - B zinda hai but slow hai (abhi response aayega)?
  - Network ne request drop kiya (B ne dekhi hi nahi)?
  - B ne process kiya but response network me kho gaya?
  A ISME SE KOI FARAK NAHI KAR SAKTA. Bas "timeout" dikhta hai.
```

Ye **partial failure** hai — aur ye distributed systems ki **defining difficulty** hai. Ek node "up but unreachable" ho sakta hai, aur tum "slow" ko "dead" se distinguish nahi kar sakte.

**Iska bura consequence:** agar A maan le B dead hai (jabki B slow tha), aur A koi action le (jaise B ka kaam kisi aur ko de de), phir B "wapas" aa jaye — ab do nodes same kaam kar rahe (duplicate/conflict). Ye **split-brain** ka root hai (lesson 04), aur **fencing tokens** (lesson 05) ki zarurat.

---

## 3. Problem 2 — Unreliable Network (asynchrony)

Network guarantees nahi deta:
- Messages **lost** ho sakte hai
- **Delayed** ho sakte hai (koi upper bound nahi — 1ms ya 10 seconds, pata nahi)
- **Out of order** aa sakte hai
- **Duplicate** ho sakte hai

> **Recap (Topic 1 lesson 01):** TCP in me se kuch handle karta hai (retransmit, ordering) — but sirf ek connection ke andar. Across nodes, across failures, ye guarantees nahi hoti. Aur "no upper bound on delay" = tum kabhi sure nahi ho sakte message aayega ya nahi.

**Asynchrony ka matlab:** kyunki message delay ki koi limit nahi, tum **timeout** pe rely karte ho — aur timeout hamesha ek guess hai. Chhota timeout = slow-but-alive node ko galti se "dead" maan lena. Bada timeout = actual failures detect karne me der.

**Frontend analogy:** tumne `fetch` pe timeout set kiya hoga. Timeout hit hua — request fail hui ya server bas slow tha? Pata nahi. Tum retry karo toh duplicate ho sakta hai (isliye idempotency — Topic 4 lesson 05). Distributed systems me ye problem har jagah hai.

---

## 4. Problem 3 — No Global Clock ⭐

Single machine: ek clock, sab events order me. Distributed: **har machine ki apni clock**, aur wo **sync nahi** hoti.

```
Machine A: event X at "10:00:00.100"
Machine B: event Y at "10:00:00.120"
X, Y se pehle hua? Pata nahi!
  Agar A ki clock 50ms fast hai → real order Y→X hai, na ki X→Y
```

> **Clock skew:** machines ki clocks drift karti hai. NTP (time-sync protocol) bhi 10-200ms drift chhoD deta hai. Concurrent events ke liye ye enough hai galat order dene ke liye. **Physical timestamps distributed systems me jhooth bolte hai.**

Iska matlab: "kaunsa event pehle hua" ko physical time se decide **nahi** kar sakte. Isliye **logical clocks** (Lamport timestamps, vector clocks — lesson 07) invent hue — jo message-passing se order track karte hai, wall-clock se nahi.

**Frontend analogy:** socho 2 users ne ek collaborative doc me same line ek saath edit ki, alag timezones/clocks se. `Date.now()` se "kaunsa pehle" decide karo? Unki clocks alag hai — reliable nahi. Isliye Figma/Google Docs jaise tools special conflict-resolution (CRDTs, vector clocks) use karte hai.

---

## 5. In sab se jo problems aati hai

Ye 3 fundamental limitations poore module ke topics generate karti hai:

```
Partial failure    → "kaun leader hai?" confusion → LEADER ELECTION (04)
                   → expired-lock corruption → FENCING TOKENS (05)
No global clock    → event ordering impossible → VECTOR CLOCKS (07)
Network partition  → C ya A choose karo → CAP THEOREM (02)
                   → consistency ka spectrum → CONSISTENCY MODELS (03)
Replicas drift     → efficiently sync karo → MERKLE TREES (08)
Data distribute    → minimal disruption → CONSISTENT HASHING (06)
```

Har topic in fundamental problems me se kisi ka **solution** hai. Isliye pehle problems samjho, phir solutions natural lagenge.

---

## 6. Consensus — the holy grail

Bahut saari distributed problems ek hi core problem me reduce hoti hai: **consensus** = kai nodes ko **ek value pe agree** karana, **despite failures aur unreliable network.**

Examples jaha consensus chahiye:
- "Kaun leader hai?" (leader election — lesson 04)
- "Ye transaction commit hua ya nahi?" (distributed transactions — Topic 2)
- "In events ka order kya hai?"
- "Cluster me kaunse nodes members hai?"

Consensus **mushkil** hai kyunki partial failure + asynchrony ke saath, nodes ka agree karna guarantee karna non-trivial hai. **Raft** aur **Paxos** (lesson 04) famous consensus algorithms hai jo isse solve karte hai — carefully, majority-voting se.

> **FLP impossibility (bas awareness):** ek famous theoretical result kehta hai ki ek **fully asynchronous** system me (jaha message delay unbounded ho), consensus **guarantee** nahi kiya ja sakta agar ek bhi node fail ho sakta ho. Practically, Raft/Paxos timeouts use karke ise "mostly works" bana dete hai — but ye batata hai problem kitni deep hai.

**Frontend analogy:** socho 5 log group chat me dinner spot decide kar rahe hai, but kuch ke messages late aa rahe, kuch temporarily offline. Sabko ek hi jagah pe reliably agree karana — while log aate-jaate rahe — surprisingly hard hai. Consensus algorithms exactly ye formalize karte hai.

---

## Quick Self-Check
1. Distributed system single machine se fundamentally kaise alag (shared memory, clock, failure)?
2. Partial failure kya hai? "Slow" ko "dead" se distinguish kyu nahi kar sakte, aur iska bura consequence?
3. Network asynchrony ka matlab (unbounded delay)? Timeout hamesha guess kyu hai?
4. Clock skew kya hai? Physical timestamps distributed me kyu "jhooth bolte hai"?
5. Ye 3 problems kaunse module topics generate karti hai (map)?
6. Consensus kya hai, aur kyu mushkil hai?

---
**Next:** `02-CAP-Theorem-And-PACELC.md` — the fundamental trade-off.