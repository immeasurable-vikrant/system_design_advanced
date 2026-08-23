# 01 — How to Make Architecture Decisions

> **Is lesson me:** Architecture decisions lene ka **process** — trade-off thinking, "it depends" ko sahi tarike se kaise use karo, Architecture Decision Records (ADRs), aur reversible vs irreversible decisions. Ye poore module ki foundation hai — decisions **content** nahi, **skill** hai.

Prerequisites: Topics 1-6 helpful (building blocks). Ye Topic 7 ki pehli file hai.

---

## 1. Architecture me koi "right answer" nahi hota

Coding me ek function sahi ya galat ho sakta hai (test pass/fail). Architecture me **trade-offs** hote hai — har choice kuch deti hai, kuch leti hai. "Best" architecture nahi hoti; sirf **tumhare context ke liye sahi** architecture hoti hai.

```
"SQL ya NoSQL?"          → depends on access patterns, consistency needs
"Microservices?"         → depends on scale, team size, domain clarity
"Cache?"                 → depends on read/write ratio, staleness tolerance
```

**Yehi wajah interviewer "it depends" sunna chahta hai** — but *sahi tarike* se. Galat "it depends" = wishy-washy, no answer. **Sahi "it depends"** = tum trade-offs samajhte ho, aur ek clear choice lete ho *with justification*:

> ❌ *"It depends on the requirements."* (lazy, no value)
> ✅ *"If reads dominate and we can tolerate seconds of staleness, I'd add a cache — it cuts DB load ~90%. If we need strong consistency on every read, I'd skip it and scale with read replicas instead. Given this is a social feed, staleness is fine, so: cache."*

**The skill:** trade-offs name karo, ek assumption pick karo, decision lo, justify karo. **Assumption state karke aage badho** — analysis-paralysis me mat atko.

---

## 2. The decision process — 5 steps

```
1. UNDERSTAND the problem
   Requirements clear karo — functional + non-functional (lesson 02)

2. ESTIMATE the scale
   Back-of-envelope — QPS, storage, bandwidth (lesson 03)
   (Ye numbers decisions drive karte hai — 100 users vs 100M users = alag design)

3. IDENTIFY the key decisions
   Kaunse choices matter karte hai? (data store, comms, scale, consistency)

4. EVALUATE trade-offs
   Har option ke pros/cons vs YOUR requirements (Topics 1-6 ka gyan)

5. DECIDE + JUSTIFY
   Ek choice lo, assumption state karo, trade-off acknowledge karo
   ("main X choose kar raha hu kyunki Y; iska cost Z hai, jo acceptable hai kyunki...")
```

**Frontend analogy:** ye bilkul ek tech-choice jaisa hai jo tumne frontend me li hai — "React ya Vue? Redux ya Context? REST ya GraphQL?" Tum requirements dekhte ho (team, scale, complexity), trade-offs weigh karte ho, decide karte ho. Backend architecture wahi skill, bade stakes pe.

---

## 3. Reversible vs Irreversible decisions ⭐

Sab decisions equal weight ke nahi. Jeff Bezos ka framework:
- **Two-way door (reversible)** — galat ho toh easily undo. Fast decide karo, experiment karo. (E.g., ek library choice, ek cache TTL, ek API endpoint shape.)
- **One-way door (irreversible)** — undo karna bahut mehenga/impossible. Careful socho, zyada analysis. (E.g., core database choice, data model, sharding key, public API contract, monolith vs microservices.)

```
Reversible (decide fast):    caching strategy, retry config, which queue,
                             internal API shape, deployment tooling
Irreversible (decide slow):  primary database, shard key (Topic 2!),
                             public API contract, data model, sync vs async core
```

> **Recap (Topic 2 lesson 13):** shard key choice — "sabse important decision, badalna painful." Ye ek classic one-way door hai. Isliye us pe zyada socho. Ek cache TTL (two-way door) pe kam.

**Interview insight:** *"Main irreversible decisions (data model, database, shard key) pe zyada time dunga — kyunki galat hua toh migration painful. Reversible decisions (cache config, which library) fast le lunga aur iterate karunga."*

---

## 4. Architecture Decision Records (ADRs)

**ADR** = ek chhota document jo ek significant architecture decision, uska context, aur reasoning capture karta hai. Taaki 6 mahine baad (ya naya team member) samajh sake "hum ye kyu use kar rahe hai."

```
ADR Template:
  Title:     "ADR-007: Use PostgreSQL for the order service"
  Status:    Accepted (ya Proposed/Deprecated/Superseded)
  Context:   Kya problem hai? Kya constraints? (order data relational,
             transactions chahiye, moderate scale)
  Decision:  Kya decide kiya? (PostgreSQL, single primary + read replicas)
  Consequences: Iska result kya? (ACID milega; write scaling later shard karna
             padega; team ko SQL aati hai — good)
  Alternatives considered: MongoDB (rejected — need transactions),
             DynamoDB (rejected — complex relational queries)
```

**Kyu ADRs matter:** decisions ka **reasoning** preserve hota hai. Log baad me "ye kyu chuna?" nahi poochte; galat reasons pe reverse nahi karte; naye log context samajhte hai. **Frontend analogy:** ye ek detailed PR description / RFC jaisa hai jo "kyu" document karta hai, na sirf "kya."

Interview me ADR likhna nahi padta, but **ADR-style sochna** — "context → decision → consequences → alternatives" — exactly wahi structure hai jo interviewer sunna chahta hai.

---

## 5. Common decision traps (avoid these) ⚠️

| Trap | Kya galat | Fix |
|---|---|---|
| **Resume-driven design** | "Kubernetes/microservices/Kafka kyunki cool" | Problem-driven — kya ACTUALLY chahiye? |
| **Premature optimization** | Din 1 se sharding, jab 100 users hai | Simplest jo kaam kare; scale jab pain ho |
| **Over-engineering** | Har jagah CQRS/event-sourcing | Complexity sirf jab benefit > cost |
| **Analysis paralysis** | Perfect decision ka wait, kuch nahi banta | Assumption state karo, decide karo, iterate |
| **Ignoring the team** | Fancy tech jo team nahi jaanti | Team ki expertise ek real constraint hai |
| **Copying big-tech** | "Netflix aise karta hai" (100M users ke liye) | Tumhara scale alag — apne context ke liye design |

> **Golden principle:** *Boring technology is good. Proven, well-understood tools (Postgres, Redis, a monolith) zyadatar problems solve kar dete hai. Novelty ki keemat (bugs, learning, ops) tabhi pay karo jab problem genuinely demand kare.*

**Interview line:** *"Main simplest architecture se shuru karunga jo requirements meet kare — proven tools (Postgres, Redis, ek modular service). Complexity (sharding, microservices, event-sourcing) tab add karunga jab ek specific bottleneck ya requirement use justify kare, preemptively nahi. Har decision me main assumption state karke trade-off acknowledge karunga."*

---

## Quick Self-Check
1. Architecture me "right answer" kyu nahi hota — trade-offs kya matlab?
2. "It depends" ko sahi tarike se kaise use karo (galat vs sahi example)?
3. Decision process ke 5 steps?
4. Reversible (two-way door) vs irreversible (one-way door) — examples + kis pe zyada socho?
5. ADR kya hai — 4 parts? Kyu matter karta hai (frontend RFC analogy)?
6. 4 common decision traps (resume-driven, premature optimization, etc.)?
7. "Boring technology" principle kya kehta hai?

---
**Next:** `02-Requirements-And-NFRs.md` — decide karne se pehle, problem clearly samjho.