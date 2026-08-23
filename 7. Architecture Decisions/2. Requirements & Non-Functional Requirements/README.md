# 02 — Requirements & Non-Functional Requirements (NFRs)

> **Is lesson me:** Kisi bhi design se pehle, problem ko clearly define karna — functional requirements (system KYA kare) vs non-functional requirements (system KAISA ho — fast, reliable, scalable), the "-ilities", aur interview me scope kaise clarify karo. Ye woh step hai jo zyadatar log skip karte hai aur galat system design kar dete hai.

Prerequisites: `01-How-To-Make-Architecture-Decisions.md`

---

## 1. Requirements pehle, design baad me

Sabse badi mistake: requirements clarify kiye bina design shuru karna. "Design Twitter" bola gaya — but kaunsa Twitter? 1000 users ka ya 500M ka? Sirf posting ya DMs bhi? Ye clarify kiye bina tum galat cheez design karoge.

**Interview me pehle 5 minute requirements pe kharch karo.** Ye weakness nahi — ye **senior signal** hai. Junior turant boxes banane lagta hai; senior pehle problem samajhta hai.

Requirements 2 tarah ke hote hai:
```
FUNCTIONAL:      System KYA karta hai (features, behavior)
NON-FUNCTIONAL:  System KAISA karta hai (fast, reliable, scalable, secure)
```

---

## 2. Functional Requirements (FRs) — kya karega

**Functional requirements** = system ke features/behaviors — user kya kar sakta hai.
```
"Design a URL shortener" ke FRs:
  - User ek long URL de → short code mile
  - Short code pe jaao → long URL pe redirect
  - (optional) User custom alias de sake
  - (optional) Click analytics dekhe
```

**Scope narrow karo (crucial):** interviewer sab kuch expect nahi karta. Core 2-3 features pick karo, baaki "out of scope for now" bol do. Ye focus dikhata hai.
```
"Design Twitter" — scope:
  IN:  post tweet, view timeline (feed), follow users
  OUT (abhi): DMs, search, trends, ads, verification — "in ko baad me"
```

**Frontend analogy:** ye ek feature spec / user stories jaisa hai — "as a user, I can post a tweet." Pehle core user journeys define karo, phir build karo.

---

## 3. Non-Functional Requirements (NFRs) — kaisa hoga ⭐

**NFRs** = system ki **qualities** — ye decide karte hai architecture (FRs sirf features batate hai; NFRs design drive karte hai). Ye "-ilities" kehlate hai:

| NFR | Sawaal | Design impact |
|---|---|---|
| **Scalability** | Kitne users/requests? Growth? | Sharding, replication, horizontal scaling (Topics 1-2) |
| **Availability** | Kitna uptime? (99.9%? 99.99%?) | Redundancy, failover, multi-region (Topic 5) |
| **Latency/Performance** | Kitna fast? (p99 < 200ms?) | Caching, CDN, indexing (Topics 1-3) |
| **Consistency** | Strong ya eventual ok? | CAP choice, replication (Topic 5) |
| **Durability** | Data lose ho sakta hai? | Replication, WAL, backups (Topic 2) |
| **Reliability** | Failures kaise handle? | Retries, DLQ, circuit breakers (Topic 4, 6) |
| **Security** | Auth, encryption, compliance? | TLS, auth, secrets (Topic 1) |
| **Maintainability** | Team kaise evolve kare? | Clean architecture, modularity (Topic 6) |
| **Cost** | Budget constraints? | Managed vs self-hosted, storage tiers |

**Ye NFRs hi architecture decide karte hai.** Example:
```
"99.99% availability" → single server nahi chalega → redundancy + failover
"p99 < 100ms globally" → CDN + edge + caching zaroori
"Strong consistency on payments" → CP system, single-leader (Topic 5)
"Handle 1M writes/sec" → sharding zaroori (Topic 2)
```

---

## 4. Availability numbers (interview me pata hone chahiye)

"High availability" vague hai — numbers me socho:
```
99%     = 3.65 DAYS downtime/year   (bad)
99.9%   = 8.76 HOURS/year           ("three nines" — decent)
99.99%  = 52 MINUTES/year           ("four nines" — good, standard SaaS)
99.999% = 5 MINUTES/year            ("five nines" — very hard, expensive)
```
Har extra "nine" **exponentially mehenga** hai (more redundancy, more regions, more ops). Isliye poocho "kitni availability ACTUALLY chahiye?" — banking = five nines; ek internal tool = 99% theek.

> **Recap (Topic 5 lesson 02 — CAP):** availability ek CAP choice se juda hai. High availability (AP) chahiye toh consistency trade karni padegi (partition ke time). "99.99% + strong consistency" ek tension hai — clarify karo kaunsa priority.

---

## 5. Interview me clarifying questions (the script)

"Design X" sunne pe, ye poocho (pehle 3-5 min):
```
SCALE:
  - Kitne users? (DAU — daily active users)
  - Kitne requests/sec? Read:write ratio?
  - Data volume? Growth rate?

FEATURES (scope):
  - Core features kaunse? Kya out-of-scope?
  - Kaun use karta hai? (consumers? internal?)

NFRs:
  - Latency requirements? (real-time? seconds ok?)
  - Consistency? (strong ya eventual?)
  - Availability target?

CONSTRAINTS:
  - Existing tech stack? (tumhare case me FastAPI/Python!)
  - Team size? Timeline? Budget?
```

**Ye numbers likh lo** — ye estimation (lesson 03) aur har decision drive karenge.

**Frontend analogy:** ye ek product/design kickoff jaisa hai — pehle "kya banana hai, kiske liye, kitne scale pe, kaunse constraints" clarify karte ho, phir architecture. Skip kiya toh galat cheez banaoge.

---

## 6. FRs → NFRs → Architecture (the flow)

```
1. FRs define karo        → "post tweet, view feed, follow"
2. Scale estimate karo    → "300M DAU, read-heavy 100:1" (lesson 03)
3. NFRs derive karo       → "low latency feed, eventual consistency ok, HA"
4. Architecture decide    → "CQRS + fan-out + cache + sharded DB" (Topics 2-6)

Har architecture decision kisi NFR ko satisfy karta hai:
  Low latency      → cache + CDN (Topic 3, 1)
  Eventual OK      → async fan-out (Topic 4)
  Read-heavy       → read replicas + cache (Topic 2, 3)
  HA               → replication + failover (Topic 5)
```

**Interview line:** *"Design shuru karne se pehle main scope aur scale clarify karunga — DAU, read:write ratio, latency/consistency requirements. Ye NFRs architecture drive karenge: agar read-heavy + eventual-consistency-ok hai (jaise feed), main caching + async fan-out lunga; agar write-heavy + strong-consistency (jaise payments), sharding + single-leader. Requirements pehle, architecture unse follow karti hai."*

---

## Quick Self-Check
1. Requirements clarify kyu — design se pehle? Ye senior signal kyu?
2. Functional vs non-functional requirements — KYA vs KAISA?
3. Scope narrow karna kyu important (Twitter example)?
4. 5 NFRs ("-ilities") aur unka design impact?
5. NFRs architecture kaise drive karte hai (2 examples)?
6. Availability numbers — 99.9% vs 99.99% kitna downtime? Har nine mehenga kyu?
7. Interview me clarifying questions ke 3 categories (scale/features/NFRs)?

---
**Next:** `03-Capacity-Estimation.md` — numbers jo har decision drive karte hai.