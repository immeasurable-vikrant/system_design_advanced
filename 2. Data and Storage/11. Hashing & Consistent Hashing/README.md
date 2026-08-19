# 11 — Hashing & Consistent Hashing

> **Is lesson me:** Hash function kya hai (from scratch), shard key + `hash % N` ka matlab, naive hashing ka problem, aur consistent hashing (ring + virtual nodes) with a fully worked numeric example. Ye sharding (lesson 13) ka direct prerequisite hai.

Prerequisites: `01-What-Is-A-Database.md` (bas basics)

---

## 1. Hash function kya hai

**Hash function** = ek function jo koi bhi input (string, number, object) ko ek **fixed-size number** me convert karta hai.

```
hash("bob@mail.com")   →  3829102938
hash("alice@mail.com") →  9123847561
hash("user_123")       →  1029384756
```

**3 important properties:**
1. **Deterministic** — same input hamesha same output. `hash("bob")` aaj bhi kal bhi same.
2. **Fixed output size** — input 1 char ho ya 10,000 char, output ek fixed-size number (e.g., 32/64-bit).
3. **Uniformly distributed** — 1000 alag inputs → outputs evenly spread (cluster nahi). Yahi property sharding/load-balancing ke kaam aati hai.

**Frontend analogy:** JS `Object`/`Map` me jab tum `user["bob@mail.com"] = {...}` karte ho, engine (V8) internally is key ko **hash** karta hai taaki O(1) me directly memory location pe jump kar sake. Same concept, bas distributed systems me scale bahut bada.

**Common hash functions (naam pehchaan lo):** MD5, SHA-256 (cryptographic), MurmurHash, CRC32, FNV (fast, non-cryptographic). Sharding me usually MurmurHash-type use hote hai — speed + even distribution chahiye, cryptographic security nahi.

---

## 2. Shard key + `hash(key) % N`

**Shard key** = ek column jise hash karke decide karte ho ki koi row **kaunse shard (server)** pe jayega.

Poora formula:
```
shard_id = hash(shard_key) % num_shards
```
Step by step:
```
Step 1: shard_key lo          → userId = "user_101"
Step 2: hash karo              → hash("user_101") = 8234759102
Step 3: num_shards se mod      → 8234759102 % 4 = 2
Step 4: result = shard index   → Shard 2
```

> **`%` (mod) recap:** `X % N` ka result hamesha `0` se `N-1` ke beech. `10 % 3 = 1`, `7 % 4 = 3`. Isliye mod se ek bada hash number ko valid shard-index (0 to N-1) me convert karte hai.

**Hash kyu, direct `userId % N` kyu nahi?**
- Shard key aksar **string** hoti hai (email, UUID) — strings pe `%` seedha kaam nahi karta. Pehle hash karke number banao.
- Agar IDs sequential/non-uniform ho (naye users consecutive IDs), hashing unhe scramble karke even distribution deta hai.

---

## 3. Naive hashing ka problem

`shard_id = hash(key) % N` kaam karta hai... **jab tak N constant hai.** Jaise hi tum ek naya server add karo (N badhe), **almost saari keys ka shard badal jaata hai:**

```
Before (N=4): hash("user_101") % 4 = 2   → Shard 2
After  (N=5): hash("user_101") % 5 = 4   → Shard 4  (moved!)
```

Ek server add karne pe bhi ~90%+ data ko re-shuffle karna padega — massive data movement, cache invalidation, downtime. **Production me ye disaster hai.** Isko fix karta hai **consistent hashing**.

---

## 4. Consistent Hashing — the ring (worked example)

Idea: servers aur keys **dono** ko ek **circular hash space (ring)** pe map karo. Real me ye `0` se `2^32 - 1`, but chhote numbers (0-100) se samajhte hai.

**Step 1: Servers ko ring pe place karo** (server name hash karke):
```
hash("ServerA") = 10
hash("ServerB") = 40
hash("ServerC") = 75

Ring (0 to 100, circular — 100 ke baad wapas 0):
   ServerA(10) → ServerB(40) → ServerC(75) → (wrap to 0)
```

**Step 2: Keys ko bhi ring pe place karo:**
```
hash("user_101") = 25
hash("user_102") = 60
hash("user_103") = 95
```

**Step 3: Rule — key apne se CLOCKWISE pehla server jise mile, usko assign:**
```
user_101 (25) → clockwise first server = ServerB(40)
user_102 (60) → clockwise first server = ServerC(75)
user_103 (95) → clockwise first (wrap around) = ServerA(10)
```

**Ab ek server ADD karo:** `ServerD` at `hash("ServerD") = 50`
```
Ring: ServerA(10) → ServerB(40) → ServerD(50) → ServerC(75)

Recheck:
  user_101 (25) → ServerB(40)  → SAME (no change!)
  user_102 (60) → ServerC(75)  → SAME (no change!)
  user_103 (95) → ServerA(10)  → SAME (no change!)
```

**Koi key move nahi hui!** Sirf wo keys jo `40` aur `50` ke beech padti (jo pehle ServerC ko jaati thi) ab ServerD ko jayengi. **Baaki sab untouched.**

Ye hi core benefit: naive `hash % N` me N change = ~100% keys move; consistent hashing me sirf **~1/N keys move** (N = servers). Formula: **O(K/N) keys move** (K = total keys).

---

## 5. Virtual Nodes — even distribution + graceful failure

**Problem:** Sirf 3-4 physical servers ring pe randomly land ho, toh unke beech gap **uneven** ho sakta hai:
```
ServerA(10) ──────────────── ServerB(85)  ← huge gap, B gets 75% load!
                             ServerC(90)  ← tiny gap, C almost idle
```
Hot spot — ek server overloaded, doosra idle.

**Solution: Virtual Nodes** — har **physical server** ko ring pe **multiple positions** (100-200) pe represent karo:
```
hash("ServerA#1") = 5,  hash("ServerA#2") = 33,  hash("ServerA#3") = 61, ...
// ServerA ke 150 virtual points ring pe scattered
```
Ab har server ke 150 chhote points mix ho ke evenly scattered hai. **Load statistically evenly distribute** ho jaata hai, chahe physical servers sirf 3 ho.

**Bonus:** Agar ServerA fail ho, uske 150 points ka load **kayi doosre servers me evenly** baant jaata hai (na ki sirf ek "next" server pe dump). Failure impact bhi evenly absorb.

---

## 6. Real-world usage

| System | Consistent hashing kaha |
|---|---|
| DynamoDB | Partition placement across storage nodes |
| Cassandra | Ring architecture, "vnodes" (virtual nodes) |
| CDNs (Akamai, Cloudflare) | Request routing to edge servers |
| Load balancers | Sticky sessions (same user → same backend) |

**Interview one-liner:** *"Consistent hashing solves the re-sharding problem — jab servers add/remove hote hai, sirf O(K/N) keys move hoti hai instead of almost all with naive modulo. Virtual nodes load ko uniformly distribute karte hai aur hot-spotting avoid karte hai."*

---

## Quick Self-Check
1. Hash function ke 3 properties kya hai?
2. `shard_id = hash(shard_key) % N` — har step ka kaam?
3. Direct `userId % N` kyu nahi, hash kyu zaroori?
4. Naive `hash % N` me server add karne pe kya problem hoti hai?
5. Consistent hashing ring pe key assign karne ka rule kya hai (clockwise)?
6. Server add hone pe consistent hashing me kitni keys move hoti hai, aur kyu?
7. Virtual nodes kaunse 2 problems solve karte hai?

---
**Next:** `12-Replication.md` — same data, multiple copies (fault tolerance + read scaling).