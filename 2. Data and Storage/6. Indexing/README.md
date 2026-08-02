# 05 — Database Indexing: Terms Explained (Companion to the Article)

> Frontend engineer ke liye sabse bada gap indexing article me ye hai: tum "disk I/O", "disk reads", "memory vs disk" jaisi cheezon se directly kabhi deal nahi karte (browser/JS engine ye sab abstract kar deta hai). Ye file wahi gap fill karti hai — kyu "3-4 disk reads = milliseconds" jaisi lines matter karti hai.

Prerequisites: `01-Hashing-Basics-Explained.md` (Hash Index section ke liye)
Pairs with: "Database Indexing - Complete Deep Dive" article

---

## 1. "Disk read" itni badi deal kyu hai?

Frontend me tumhara data mostly **RAM (memory)** me hota hai — JS objects, arrays, React state. RAM access **nanoseconds** me hota hai (bahut fast).

Databases apna data **disk pe** store karte hai (persistent storage — RAM crash/restart pe khali ho jati hai, disk pe data bacha rehta hai — yehi "Durability" hai jo Fundamentals doc me discuss hua tha).

**Problem: Disk access RAM se HAZAAR guna slower hota hai:**

```
RAM access:        ~100 nanoseconds     (0.0000001 seconds)
SSD disk access:    ~100,000 nanoseconds (0.0001 seconds)   → ~1000x slower than RAM
HDD disk access:  ~10,000,000 nanoseconds (0.01 seconds)    → ~100,000x slower than RAM
```

Isliye database design ka **core goal** hi ye hota hai: **jitna kam ho sake utne kam disk reads karo** — kyunki har extra disk read = extra milliseconds jo user ko wait karwa raha hai.

**Frontend analogy:** Ye bilkul waisa hai jaise ek slow network API call — agar tumhari app 10 sequential API calls kare instead of 1 batched call, poori page load slow ho jaati hai. Database ke liye "disk read" = frontend ke liye "network call" jaisa hi expensive resource hai — minimize karna hai.

---

## 2. B-Tree "depth = 3-4" ka matlab kya hai (aur ye important kyu hai)

Article kehta hai: *"Depth = ~3-4 for millions of rows, Each lookup = 3-4 disk reads = milliseconds"*

B-Tree ek **tree structure** hai (jaise DOM tree — parent-child relationships, but data ko sorted range me organize karta hai). "Depth" matlab root se leaf tak kitne **levels** traverse karne padte hai:

```
                    [Root]                    ← Level 1 (1 disk read)
                   /       \
            [Node]           [Node]           ← Level 2 (1 disk read)
           /      \          /      \
      [Node]    [Node]   [Node]   [Node]      ← Level 3 (1 disk read)
        |
     [Leaf → actual row data]                 ← Level 4 (1 disk read = final data)
```

**Har level = ek disk read** (kyunki har node disk pe ek alag jagah store hota hai, aur usko fetch karna ek "disk I/O operation" hai).

Ab magic ye hai: B-Tree **balanced** hota hai, aur bahut **wide** (har node me bahut saari sorted keys hoti hai, jaise 100-500 keys per node). Isliye:

```
1 million rows  → depth ~3
1 billion rows  → depth ~4-5 (barely increases!)
```

Ye **logarithmic growth** hai (`O(log N)`) — data 1000x badh jaye, depth sirf 1-2 level badhta hai. Isliye chahe rows millions/billions me ho, lookup sirf **3-5 disk reads** me ho jata hai — jo milliseconds me complete ho jata hai.

**Compare karo without index:**

```
Without index: "full table scan" = HAR row ko check karna padega
  10 million rows → potentially 10 million disk reads (worst case)
  → seconds ya usse zyada time lagega

With index (B-Tree): sirf 3-4 disk reads
  → milliseconds
```

Yehi wajah hai article me bola gaya: *"O(N) → SLOW"* vs *"O(log N) → FAST"* — ye Big-O notation directly disk reads ki count se correlate karta hai.

---

## 3. Hash Index range queries kyu support nahi karta? (deep reason)

Article bolta hai Hash index range queries (`WHERE age > 25`) support nahi karta. Reason samjho — ye hashing ki fundamental property se related hai (`01-Hashing-Basics-Explained.md` yaad karo):

```
hash(24) = 9123847   (koi random number)
hash(25) = 302981    (bilkul unrelated random number)
hash(26) = 8123049   (phir se unrelated)
```

Hash function ki property yahi hai ki **similar inputs, completely unrelated/random outputs** dete hai (isse "avalanche effect" bolte hai). Iska matlab: agar tumhe `age > 25` chahiye (matlab 26, 27, 28...), hash values me koi **order/pattern nahi hai** jisse tum "range" nikal sako — tumhe pata hi nahi konse hash buckets me ye consecutive ages honge, kyunki hash unko completely scatter kar deta hai.

**B-Tree isliye range support karta hai** kyunki wo data ko **sorted order me hi rakhta hai** (hash nahi karta) — isliye `age > 25` ke liye bas tree me 25 dhundo, phir sequentially aage badhte jao — sab sorted hai isliye contiguous milega.

```
B-Tree (sorted):     [10, 15, 20, 25, 30, 35, 40]  → range easy hai, consecutive hai
Hash Index (scattered): bucket7, bucket2, bucket9, bucket1  → koi order nahi, range impossible
```

---

## 4. "Index-only scan" / Covering Index — actually kya bachaata hai

Article: *"Covering index... name and email are already IN the index → return directly! No table access needed"*

Normal index ke saath query flow (2 hops):

```
Query: SELECT name, email FROM users WHERE age > 25

Hop 1: Index pe jao, "age > 25" wale rows ke row-IDs nikaalo (index disk read)
Hop 2: Har row-ID ke liye, actual TABLE pe jao aur uska name/email fetch karo
       (ye ek ALAG disk location hai — extra disk read PER ROW!)

Agar 1000 rows match kare → 1000 EXTRA disk reads (Hop 2 ke liye)!
```

Covering index ke saath (index me hi `name`, `email` bhi store hai):

```
Hop 1: Index pe jao, "age > 25" match karo, aur wahi se name/email bhi mil gaya
→ Table tak jaane ki zarurat hi nahi — poori query sirf index se resolve ho gayi!
```

Yehi wajah hai "index-only scan" fast hota hai — kyunki tum ek poore extra "hop to table" (jo random disk locations pe hota hai, expensive) ko completely skip kar dete ho.

---

## 5. Quick real-world analogy for the whole article

Socho ek phone book (physical, purane zamane ka):

```
Full table scan     = poori phone book ka har page palatna, naam dhundhne ke liye
B-Tree index        = phone book already alphabetically sorted hai, binary search jaisa jump karo
Hash index          = ek magic lookup table jo exact naam se seedha page number bata de,
                       but "sabhi 'A' se shuru hone wale naam do" nahi bata sakta (order nahi hai)
Covering index      = phone book ke index page pe hi naam + number + address, sab likha hai
                       (asal listing tak jaane ki zarurat nahi)
```

---

## Quick Self-Check
1. Disk read RAM access se kitna slower hota hai, aur database design me ye kyu matter karta hai?
2. B-Tree "depth ~3-4" ka seedha relation kis cheez se hai (disk reads)?
3. Hash index range queries kyu support nahi kar sakta — root cause kya hai?
4. Covering index kaunsa extra "hop" bachata hai?