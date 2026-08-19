# 09 — Bloom Filters (Probabilistic Membership)

> **Is lesson me:** Bloom filter kya hai (ek space-efficient probabilistic structure), kaise "definitely NOT / probably YES" answer deta hai, false positives kyu hote hai (aur false negatives kyu NAHI), sizing, real use cases (cache penetration se LSM-trees tak), variants, aur HyperLogLog ka zikr. Ye "expensive lookup avoid karo" ka ultimate tool hai.

Prerequisites: `05-Cache-Failure-Modes.md` (cache penetration bridge), helpful: DB lesson 11 (hashing)

---

## 1. Bloom filter kya hai

**Bloom filter** = ek space-efficient **probabilistic** data structure jo ek sawaal ka jawaab deta hai: **"Kya ye element set me hai?"**

Do possible answers:
- **"Definitely NOT in set"** — 100% sure, kabhi galat nahi
- **"Probably in set"** — most likely haan, but **chhota chance galat** (false positive)

**Kabhi false negative nahi deta** — agar wo "nahi hai" bolе, toh pakka nahi hai.

**Analogy:** Ek bouncer with a guest list, jiski memory kharab hai but jo kabhi kisi list-wale chehre ko nahi bhoolta. Agar bouncer bole "tum list pe NAHI ho" → tum pakka nahi ho. Agar bole "familiar lagte ho, jao" → chhota chance hai ki kisi aur se confuse kar raha (false positive). Fast + bahut kam jagah leta hai (poori list store nahi karni padti).

---

## 2. Kaise kaam karta hai — step by step

Socho ek **bit array** (light switches, sab OFF/0 se shuru) + **kuch hash functions** (yaad hai DB lesson 11 — hash = input → number).

Setup: 10-slot bit array, 3 hash functions.
```
Bit array: [0,0,0,0,0,0,0,0,0,0]
            0 1 2 3 4 5 6 7 8 9
```

**INSERT "cat":** 3 hash functions se positions nikaalo, un bits ko 1 karo:
```
hash1("cat")=1, hash2("cat")=4, hash3("cat")=7
Bit array: [0,1,0,0,1,0,0,1,0,0]
              ↑     ↑     ↑
```

**INSERT "dog":**
```
hash1("dog")=2, hash2("dog")=4 (cat ke saath collision — fine, 1 hi rehta), hash3("dog")=9
Bit array: [0,1,1,0,1,0,0,1,0,1]
```

**QUERY "cat"** — positions 1,4,7 check:
```
pos1=1 ✓, pos4=1 ✓, pos7=1 ✓ → SAARE 1 → "probably yes" (aur sahi, humne insert kiya tha)
```

**QUERY "bird"** — positions 1,5,9:
```
pos1=1 ✓, pos5=0 ✗ ← STOP. Ek bhi 0 mila → "DEFINITELY NOT in set"
```

**QUERY "fox"** (the false positive) — positions 1,2,9:
```
pos1=1 ✓ (cat ne set kiya), pos2=1 ✓ (dog ne), pos9=1 ✓ (dog ne)
→ SAARE 1 → "probably yes" — but humne "fox" kabhi insert NAHI kiya! FALSE POSITIVE.
```
"fox" ke bits **doosre elements ne** coincidentally set kar diye the. Yahi false positive ka mechanism.

---

## 3. The two guarantees (core insight)

| Filter kahe | Matlab | Galat ho sakta? |
|---|---|---|
| **"Definitely NOT"** | Kam se kam ek bit 0 tha | **Kabhi nahi** — insert kiya hota toh saare bits 1 hote |
| **"Probably yes"** | Saare bits 1 hai | **Kabhi-kabhi** — doosre elements ne wo bits set kiye ho sakte |

**Ye asymmetry hi kaam ki cheez hai:** "no" answer pe **100% bharosa** karo, occasional false "yes" tolerate karo.

**Kyu false negative IMPOSSIBLE:** insert sirf bits ko 0→1 karta hai, kabhi 1→0 nahi. Agar element insert hua, uske saare hash-positions guaranteed 1 hai. Isliye "nahi hai" kabhi galat nahi ho sakta.

---

## 4. False positive rate — control kaise

False positive rate 3 cheezon pe depend karta hai:

| Parameter | Symbol | Effect |
|---|---|---|
| Bit array size | m | Bada m → kam collisions → kam FP |
| Hash functions | k | Optimal k FP minimize karta (kam/zyada dono bura) |
| Inserted elements | n | Zyada elements → zyada bits set → zyada FP |

Formula: `P(false positive) ≈ (1 - e^(-kn/m))^k`

**Practical sizing (memorize the shape):**
```
1M elements,   1% FP  → ~1.2 MB, 7 hash functions
1M elements,   0.1% FP→ ~1.8 MB, 10 hash functions
100M elements, 1% FP  → ~120 MB
1B elements,   1% FP  → ~1.2 GB
```
Compare: 1 billion URLs (avg 50 bytes) store karna = **50 GB**. Bloom filter = **1.2 GB** = **40x kam memory**. Yahi iski superpower — extreme space efficiency.

**Tuning:** FP zyada aa raha? → bit array bada karo (`m` badhao). ~10 bits per element → FP < 1%.

---

## 5. Key properties

| Property | Value |
|---|---|
| False negatives | **NEVER** (guaranteed) |
| False positives | Possible (tunable, ~1%) |
| Deletion | **NOT supported** (bit unset karne se doosre elements affect) |
| Space | O(m) bits — actual elements se bahut kam |
| Insert / Query time | O(k) — sirf k hash computations |

**Deletion kyu nahi:** multiple elements same bit share karte hai. "A" ke liye bit clear karo → shayad "B" ka bhi zaroori bit clear ho gaya → "B" ke liye false negative ban gaya (core guarantee toot gayi). Counting Bloom Filter (neeche) ise solve karta hai.

---

## 6. Real use cases

| Use case | Kaun | Kaise |
|---|---|---|
| **Cache penetration rok** | Many | Valid keys bloom me; "definitely not" → DB/cache bypass (lesson 05!) |
| **LSM-tree disk read avoid** | Cassandra, RocksDB, LevelDB | "Kya is SSTable me key ho sakti?" — nahi toh disk read skip (DB lesson 10) |
| **Duplicate content detection** | Medium, Twitter | "User ne ye post dekha?" |
| **Malicious URL check** | Chrome Safe Browsing | URL local bloom me? "nahi" → no network call |
| **Username availability** | Signup flows | "Definitely available" → instant, no DB |
| **Weak password check** | HaveIBeenPwned | Password hash breach-set me? |

### Cache penetration example (lesson 05 ka bridge, ab poora)
```
Bloom filter me saari VALID user IDs.
Request "user:99999999" aayi:
  → Bloom se poocho: "ye ID exist karti?" → "DEFINITELY NOT"
  → turant reject, DB/cache dono bypass ✅
Malicious actor random IDs se DB flood nahi kar sakta.
```

### URL shortener example (DB lesson 16 se connect)
```
Naya short code generate karne se pehle: "ye code already hai?"
  Bloom "NO" → definitely available → seedha likho (DB check skip)
  Bloom "MAYBE" → DB confirm karo
→ ~99% code-generations me DB lookup bacha (1% FP pe)
```

---

## 7. Variants (awareness)

| Variant | Deletion? | Extra |
|---|---|---|
| **Standard Bloom** | No | Simplest, most common |
| **Counting Bloom** | Yes | Bits ki jagah counters (delete pe decrement) |
| **Cuckoo Filter** | Yes | Better space for FP < 3% |
| **Scalable Bloom** | No | Dynamically grow karta hai |

---

## 8. Bloom vs alternatives + HyperLogLog

| Approach | Memory | False positives | Deletion | Answers |
|---|---|---|---|---|
| Bloom Filter | Very low | Yes (tunable) | No | "Is X present?" |
| HashSet | High (stores elements) | Never | Yes | "Is X present?" + retrieve |
| **HyperLogLog** | Very low | N/A | No | **"How MANY unique?"** (count, not membership) |

> **HyperLogLog (DB lesson 07 me dekha tha):** bloom filter "membership" batata hai (hai/nahi); **HyperLogLog "cardinality" batata hai** (kitne unique). E.g., "aaj kitne unique visitors?" — HLL ~12KB me billions count kar leta hai (approximate, ~2% error). Dono probabilistic + space-efficient, but alag sawaal. Redis me dono available (`PFADD`/`PFCOUNT` for HLL).

---

## 9. When to use / NOT

**✅ Use when:** space efficiency > absolute accuracy; false positives ok but false negatives NOT; bahut bade set me fast membership; expensive operation (disk/network) avoid karna hai for definitely-absent elements; set write-once/rarely-updated.

**❌ Don't when:** deletion chahiye (→ Cuckoo Filter); false positives unacceptable (→ exact set); set chhota hai (→ HashSet); actual elements retrieve karne hai (bloom sirf yes/no); exact count chahiye (→ HyperLogLog).

---

## Quick Self-Check
1. Bloom filter ke 2 possible answers, aur kaunsa 100% trustworthy hai?
2. Insert aur query kaise hote hai (bits + hash functions)?
3. False positive kyu hota hai? False negative kyu IMPOSSIBLE hai?
4. FP rate kis 3 cheezon pe depend? Kam karne ke liye kya badhao?
5. Bloom filter 1B URLs pe kitni memory bachata hai (vs actual store)?
6. Deletion kyu nahi support karta? Kaunsa variant fix karta?
7. Cache penetration bloom se kaise rukta hai (lesson 05 se pura)?
8. Bloom filter vs HyperLogLog — kaunsa "membership", kaunsa "count"?

---
**Next:** `10-Performance-Playbook.md` — sab combine, real designs me.