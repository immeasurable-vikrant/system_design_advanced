# 01 — Hashing Basics (Foundation for Sharding + Consistent Hashing)

> Ye file isliye banayi kyunki "Database Sharding" article me line aati hai: **"Hash the shard key and mod by number of shards"** — aur ye assume kar leta hai ki tumhe pata hai hash function kya hota hai, shard key kya hai, aur mod kyu kar rahe hai. Frontend se aane wale liye ye 3 cheezein bilkul obvious nahi hoti. Isliye pehle isko clear karte hai, phir baaki articles easily samajh aayenge.

Prerequisites: none (ye sabse pehli file hai padhne ke liye)
Feeds into: Consistent Hashing, Database Sharding

---

## 1. Sabse pehle: "Hash" hota kya hai?

Ek **hash function** ek aisa function hota hai jo:

```
input (kuch bhi — string, number, object)
        │
        ▼
   HASH FUNCTION
        │
        ▼
output (ek fixed-size number, usually bahut bada)
```

Simple example (real hash functions isse complex hote hai, but idea yahi hai):

```js
hash("bob@mail.com")   →  3829102938
hash("alice@mail.com") →  9123847561
hash("user_123")       →  1029384756
```

**3 important properties jo tumhe yaad rakhni hai:**

1. **Deterministic** — same input hamesha same output dega. `hash("bob@mail.com")` aaj bhi 3829102938 dega, kal bhi.
2. **Fixed output size** — chahe input 1 character ka ho ya 10,000 characters ka, output hamesha ek fixed-size number hota hai (e.g., 32-bit ya 64-bit integer).
3. **Uniformly distributed (ideally)** — agar tum 1000 alag-alag inputs hash karo, outputs random tarike se **evenly spread** hone chahiye — na ki sab ek jagah cluster ho jaye. Ye property hi sharding/consistent hashing ke kaam aati hai (load evenly distribute karne ke liye).

### Frontend analogy (JS ke through samjho)

Tumne JS me `Object` ya `Map` use kiya hoga:

```js
const user = {};
user["bob@mail.com"] = { name: "Bob", age: 25 };
```

Andar-andar, JS engine (V8) is key (`"bob@mail.com"`) ko internally **hash karta hai** taaki O(1) me directly us memory location pe jump kar sake, bina saare keys check kiye. Yehi concept hai jo databases aur distributed systems bhi use karte hai — bas scale bahut zyada bada hota hai (millions of servers/rows ke across).

**Common real hash functions ke naam** (tumhe implement nahi karne, bas naam pehchaan lo agar kahi mention ho): MD5, SHA-256, MurmurHash, CRC32, FNV. Sharding/consistent hashing me generally **MurmurHash** ya similar "fast, non-cryptographic" hash functions use hote hai — cryptographic security nahi chahiye yaha, sirf speed + even distribution chahiye.

---

## 2. "Shard Key" kya hota hai?

Shard key bas ek **column/field ka naam** hai jisko tum hash karne ke liye choose karte ho, taaki decide kar sako ki koi particular row/record **kaunse shard (server) pe jayega**.

```
Table: users
┌─────────┬─────────┬──────┐
│ userId  │ name    │ age  │
├─────────┼─────────┼──────┤
│ 101     │ Bob     │ 25   │
│ 102     │ Alice   │ 30   │
│ 103     │ Eve     │ 22   │
└─────────┴─────────┴──────┘

Agar shard key = userId, toh har row ka shard,
uske userId ki value pe depend karega.
```

Shard key **koi bhi column** ho sakta hai — `userId`, `email`, `orderId`, `country` — but jaisa "Database Sharding" article ne bataya, sab columns equally achhe nahi hote (high cardinality + even distribution wale best hote hai — is baat ka detail `05-Sharding-Deep-Dive-Explained.md` me hai).

**Important distinction (jo articles me implicit tha):**
- **Shard key** = wo column jisko hash karke decide karte ho konsa shard
- **Primary key** = row ko uniquely identify karne wala column (jaise userId, orderId khud)

Kabhi-kabhi dono same column hote hai (e.g., `userId` shard key bhi hai aur primary key bhi), but conceptually alag purpose hai.

---

## 3. "Mod by number of shards" — ye kyu?

`%` (modulo/mod operator) tumhe JS me pata hi hoga:

```js
10 % 3 = 1   // 10 ko 3 se divide karo, remainder = 1
7 % 4 = 3    // 7 ko 4 se divide karo, remainder = 3
```

Mod operator ka guarantee ye hai: `X % N` ka result **hamesha 0 se (N-1) ke beech** hoga.

Ab poori line wapas dekhte hai: **"Hash the shard key and mod by number of shards"**

```
shard_id = hash(shard_key) % num_shards
```

Step by step:

```
Step 1: shard_key value lo         →  userId = "user_101"
Step 2: use hash karo               →  hash("user_101") = 8234759102 (koi bhi bada number)
Step 3: num_shards se mod karo      →  8234759102 % 4 = 2
Step 4: result = shard index        →  Shard 2
```

**Kyu ye 2-step process (hash + mod) zaroori hai, sirdh userId % 4 kyu nahi kar dete?**

Kyunki `userId` khud sequential ho sakta hai (101, 102, 103...) — agar directly `userId % 4` karoge, toh:
```
101 % 4 = 1
102 % 4 = 2
103 % 4 = 3
104 % 4 = 0
105 % 4 = 1   ← pattern repeat, but ye theek bhi hai agar IDs perfectly sequential + uniform hai
```
Problem tab aati hai jab shard key **string** ho (email, UUID) — unko directly `%` nahi kar sakte (strings pe modulo operator kaam nahi karta), isliye pehle unhe hash karke **number** banate hai, phir uss number ko mod karte hai. Aur agar userId bhi non-uniform pattern me ho (jaise saare naye users ke IDs consecutively aa rahe hai), toh hashing unko achhe se scramble/randomize kar deta hai taaki distribution even rahe.

**One line summary:** Hash → koi bhi key (string/number) ko ek bade random-looking number me convert karta hai. Mod → us bade number ko `0` se `N-1` ke range me la deta hai, jo directly ek valid shard index banta hai.

---

## 4. Ye approach ka problem (jo Consistent Hashing solve karta hai)

Agar `num_shards` (N) change ho jaye — matlab tum ek naya server add karo — toh:

```
Before (N=4): hash("user_101") % 4 = 2   → Shard 2
After  (N=5): hash("user_101") % 5 = 4   → Shard 4  (moved!)
```

Almost **saari keys ka shard remap ho jata hai** jab N change hota hai — bas is wajah se ki mod operation directly N pe depend karta hai. Ye exact problem hai jiska solution "Consistent Hashing" hai (uski poori deep dive `04-Consistent-Hashing-Deep-Dive.md` me hai).

---

## Quick Self-Check
1. Hash function ke 3 important properties kya hai?
2. Shard key aur Primary key me difference kya hai?
3. `hash(key) % N` — is formula ka har part kya kaam karta hai?
4. Directly `userId % N` kyu nahi karte, hash karne ki zarurat kyu padi?
5. Jab N (number of shards) change hota hai, kya problem hoti hai?