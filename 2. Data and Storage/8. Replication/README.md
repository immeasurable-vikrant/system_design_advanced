# 04 — Database Replication: Terms Explained (Companion to the Article)

> "Database Replication - Complete Deep Dive" article kaafi terms use karta hai jo assume kar leta hai tumhe pata hai — WAL, quorum formula (R+W>N), fencing tokens, CRDTs. Yaha in sabko from-scratch explain kar raha hu.

Prerequisites: Section B2 of Database Fundamentals doc (Replication basics)
Pairs with: "Database Replication - Complete Deep Dive" article

---

## 1. "WAL" (Write-Ahead Log) kya hai?

Article: *"Primary writes to its local storage (WAL + commit)"*

**WAL = Write-Ahead Log.** Simple idea: database, data ko actual table me update karne se **PEHLE**, ek append-only log file me likh deta hai "main ye change karne wala hu".

```
Client: "Set balance = 500 for user 101"

Step 1: WAL me pehle likho:
  [LOG] "user 101: balance 300 → 500" (append-only, disk pe safely likha)

Step 2: Ab actual table me update karo:
  users table: user_101.balance = 500

Step 3: Commit confirm karo client ko
```

**Ye zaroori kyu hai?** Agar database crash ho jaye **exactly Step 2 ke beech me** (aadha update ho chuka, aadha nahi) — restart hone pe database WAL ko dekh ke replay kar sakta hai: "achha, ye change complete nahi hua tha, ise phir se apply karta hu." Isse **durability** guarantee hoti hai (ACID ka "D" — jo Database Fundamentals doc me discuss hua tha).

**Replication me WAL ka role:** Primary jo bhi apne WAL me likhta hai, wahi log **followers ko bhi stream** kar deta hai — followers usi log ko apne pe replay karke apna data update karte hai. Matlab replication actual mechanism hi WAL streaming pe based hota hai.

**Frontend analogy:** Ye bilkul Redux ke "action log" jaisa concept hai — har action ek log entry hai, aur current state = saare logged actions ko sequentially apply karne ka result. Agar app crash ho jaye, tum actions replay karke state rebuild kar sakte ho (event sourcing pattern - agar suna ho).

---

## 2. Quorum Formula: `R + W > N` — iska matlab kya hai?

Article ye formula deta hai but explain nahi karta ki ye guarantee kya deta hai.

Pehle variables samjho:
```
N = total replicas (e.g., 3 servers total)
W = kitne replicas ko WRITE confirm karna zaroori hai before success
R = kitne replicas ko READ karna zaroori hai before result return karna
```

**Example: N=3, W=2, R=2**

```
WRITE "balance=500":
  Client → Node1 (ACK), Node2 (ACK), Node3 (timeout/slow)
  W=2 satisfied (Node1 + Node2 confirmed) → write successful, Node3 update hoga background me later

READ "balance":
  Client → query Node1 AND Node2 (R=2)
  Dono se response milta hai, jo bhi "sabse latest version" hai wo return karo
```

**Ye guarantee kyu deta hai ki tumhe hamesha latest data milega?**

Kyunki `R + W > N` (2+2=4 > 3) matlab **overlap guaranteed hai**. Jitne bhi nodes tum write ke time confirm karwate ho (W), aur jitne bhi read ke time query karte ho (R) — inka combined count total nodes (N) se zyada hai, isliye **kam se kam 1 node aisa zaroor milega jo dono me common ho** — matlab wo node jisne latest write receive kiya tha, wahi read set me bhi shamil hoga.

```
Pigeonhole principle:
  Total nodes = 3
  Write touched 2 nodes: {Node1, Node2}
  Read touches 2 nodes: {Node2, Node3}  (ya koi bhi combination of 2)

  2 + 2 = 4 > 3 → overlap guaranteed
  {Node1, Node2} ∩ {Node2, Node3} = {Node2}  ← ye node latest data batayega!
```

Agar `R + W <= N` hota (jaise W=1, R=1, N=3), toh overlap guarantee nahi hai — write sirf Node1 ko gaya ho sakta hai, read Node3 se ho sakta hai — **stale data mil sakta hai**.

**Frontend analogy:** Socho tumhare paas 3 cache layers hai (browser cache, CDN, service worker cache). Agar tum sirf 1 layer update karo aur sirf 1 layer se read karo, ho sakta hai tumhe purana data mil jaye. Agar tum "majority" layers ko update karo aur "majority" se read karo, guarantee milta hai ki kam se kam ek updated layer se cross hoga.

---

## 3. "Fencing Tokens" — split-brain se bachne ka tarika

Article: *"use fencing tokens (monotonically increasing IDs)"*

Pehle **split-brain** problem samjho: Primary crash hone ke baad, ek replica **naya primary** bann jata hai (failover). Problem: agar purana primary **actually crash nahi hua tha**, sirf network se temporarily disconnect hua tha, aur wapas aa gaya — ab **2 nodes khud ko primary samajh rahe hai** simultaneously! Dono independently writes accept karna shuru kar sakte hai → data corruption/conflict.

**Fencing token ka solution:** Har baar jab koi node "primary" banta hai, usko ek **hamesha badhta hua number (monotonic ID)** milta hai:

```
Old Primary elected → fencing token = 1
[network issue, old primary disconnects temporarily]
New Primary elected (failover) → fencing token = 2

Storage layer ka rule: "sirf highest fencing token wale writes accept karo"

Old Primary wapas aata hai, write bhejta hai with token=1
Storage: "token=1 already superseded by token=2, REJECT this write"

New Primary writes with token=2
Storage: "token=2 is highest so far, ACCEPT"
```

Storage layer purane (stale) token wale writes ko **reject** kar deta hai, chahe wo technically "primary" hi kyu na samajhta ho khud ko. Isse dual-writing/corruption ruk jata hai.

**Frontend analogy:** Ye bilkul waisa hai jaise tum ek search-as-you-type feature banate ho — agar user fast type kare, purani API request (jo abhi bhi in-flight hai) ka response wapas aaye, uske "request ID" ko check karke discard kar dete ho agar ek newer request already resolve ho chuka hai (race condition handling — `AbortController` ya request-id-check pattern se familiar hoge).

---

## 4. "CRDTs" (Conflict-free Replicated Data Types) kya hai?

Article: *"CRDTs — conflict-free replicated data types (counters, sets)"*

Multi-primary replication me, agar 2 primaries **same data ko simultaneously** update kare, conflict aa sakta hai. CRDT ek special data structure hai jo is tarah design ki gayi hai ki conflicting updates **automatically, predictably merge** ho jaye — bina kisi manual conflict resolution logic ke.

**Simplest example — counter:**

```
Normal approach (BAD for concurrent updates):
  Node A: counter = 5, sets counter = 6 (increment)
  Node B: counter = 5, sets counter = 6 (increment, unaware of A's change)
  Merge: dono "6" bhejte hai — final value 6, but actually dono ne increment kiya tha,
         real answer 7 hona chahiye tha! Data lost.

CRDT approach (Increment-only counter, aka "G-Counter"):
  Har node apna alag "delta" track karta hai:
  Node A: {A: +1}
  Node B: {B: +1}
  Merge rule: sum all deltas = {A:+1, B:+1} → total = 2 increments happened
  → Correctly merges to +2, no data lost, no conflict resolution logic needed!
```

Real CRDTs (sets, counters, etc.) is tarah design kiye jaate hai ki "merge" operation **mathematically commutative aur associative** ho — matlab chahe kis order me updates aaye, kis node se aaye, final merged result hamesha same/correct aata hai.

**Frontend analogy:** Agar tumne Figma/Google Docs jaisa real-time collaborative editor dekha hai — jaha 2 log same document simultaneously edit karte hai bina conflicts ke — unke andar CRDT-jaisi (ya OT - Operational Transformation) techniques hoti hai jo automatically merge decide karti hai.

---

## Quick Self-Check
1. WAL kis problem ko solve karta hai (crash ke case me)?
2. `R + W > N` formula kya guarantee deta hai, aur kyu?
3. Split-brain problem kya hai, aur fencing token isse kaise rokta hai?
4. CRDT normal "last write wins" approach se better kyu hai concurrent updates ke liye?