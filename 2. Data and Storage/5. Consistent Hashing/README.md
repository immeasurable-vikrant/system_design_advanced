# 02 — Consistent Hashing: Deep Dive with Worked Example

> Ye topic Section B8 (Database Fundamentals doc) me cover hua tha, but yaha ek **fully worked numeric example** ke saath, taaki "ring", "clockwise", "virtual nodes" jaise words sirf theory na rahe — actual numbers pe dekh sako kaise kaam karta hai.

Prerequisites: `01-Hashing-Basics-Explained.md`
Feeds into: Database Sharding

---

## 1. Recap — problem kya hai

Naive approach: `shard_id = hash(key) % N`

Jaise hi `N` (number of servers) change hota hai, **almost saari keys ka shard badal jata hai** (previous file me dikhaya). Consistent hashing isko fix karta hai.

---

## 2. Ring kya hota hai (actual numbers ke saath)

Socho hash function ka output range hai `0` se `100` tak (real me ye `0` se `2^32 - 1` tak hota hai, but chhote numbers se samajhna easy hai).

Ye ek **circle/ring** hai — matlab 100 ke baad wapas 0 pe aa jata hai (jaise clock ki 12 ke baad 1 aata hai).

**Step 1: Servers ko ring pe place karo**

Har server ka naam bhi hash karo, aur uska hash value hi uski position ring pe ban jaati hai:

```
hash("ServerA") = 10
hash("ServerB") = 40
hash("ServerC") = 75

Ring (0 to 100, circular):

        0/100
          │
   ServerA(10)
          │
    ...
          │
   ServerB(40)
          │
    ...
          │
   ServerC(75)
          │
    ...back to 0
```

**Step 2: Har key (data) ko bhi hash karke ring pe place karo**

```
hash("user_101") = 25
hash("user_102") = 60
hash("user_103") = 95
```

**Step 3: Rule — key apne se "clockwise direction me sabse pehla jo server mile", usko assign hoti hai**

```
user_101 (pos 25) → clockwise jaate hue pehla server = ServerB(40)   → assigned to ServerB
user_102 (pos 60) → clockwise jaate hue pehla server = ServerC(75)   → assigned to ServerC
user_103 (pos 95) → clockwise jaate hue pehla server = ServerA(10)  (ring wrap around) → assigned to ServerA
```

Visual:

```
                0/100
                 │
        ServerA(10) ◄──── user_103(95) [wraps around]
                 │
                 │
        user_101(25)
                 │
        ServerB(40) ◄──── user_101(25) assigned here
                 │
        user_102(60)
                 │
        ServerC(75) ◄──── user_102(60) assigned here
                 │
              (back to 0)
```

---

## 3. Ab ek server add karte hai — dekho kitni keys move hoti hai

Naya `ServerD` add kar rahe hai: `hash("ServerD") = 50`

```
Ring ab: ServerA(10) → ServerB(40) → ServerD(50) → ServerC(75) → back to 0
```

Recheck karte hai har key ka assignment:

```
user_101 (pos 25) → clockwise first server = ServerB(40)  → SAME as before, no change!
user_102 (pos 60) → clockwise first server = ServerC(75)  → SAME as before, no change!
user_103 (pos 95) → clockwise first server = ServerA(10)  → SAME as before, no change!
```

Koi bhi key move nahi hui! Kyu? Kyunki `ServerD(50)` sirf `ServerB(40)` aur `ServerC(75)` ke **beech** aaya hai — sirf wahi keys affect hongi jo is specific range (40 to 50) me padti thi, jo pehle `ServerC` ko assign hoti thi.

Agar koi key hoti `pos = 45` (ServerB aur ServerD ke beech), sirf wahi move hoti — `ServerC` se `ServerD` pe. **Baaki sab untouched.**

**Ye hi core magic hai:** naive `hash % N` me N change hote hi ~100% keys move hoti (previous file me dikhaya), consistent hashing me sirf `~1/N` keys move hoti hai (jahan N = total servers).

---

## 4. Virtual Nodes — problem aur solution

**Problem:** Agar sirf 3-4 physical servers randomly ring pe land ho, toh unke beech ka **gap uneven** ho sakta hai:

```
ServerA(10) ────────────────── ServerB(85)   ← huge gap, ServerB gets 75% of all keys!
                                ── ServerC(90) ← tiny gap, ServerC gets almost nothing
```

Ye "hot spot" problem hai — ek server bahut zyada load le raha hai, doosra almost idle hai. Wajah: hash function random hai, isliye 3-4 points ring pe evenly spread hone ki guarantee nahi.

**Solution: Virtual Nodes**

Har **physical server** ko ring pe multiple (jaise 100-200) **alag-alag positions** pe represent karo, using different hash inputs:

```js
// Real system me aisa kuch hota hai:
hash("ServerA#1") = 5
hash("ServerA#2") = 33
hash("ServerA#3") = 61
hash("ServerA#4") = 88
// ... ServerA ke 150 aise virtual points ring pe scattered
```

Ab ring pe ServerA ke 150 chhote-chhote points hai, ServerB ke 150, ServerC ke 150 — sab mix ho ke evenly scattered hai. Result: load **statistically evenly distribute** ho jata hai, chahe physical server sirf 3 hi ho.

**Bonus benefit:** Agar `ServerA` fail ho jaye, uske 150 virtual points ka load **150 alag jagah se, kayi doosre servers me evenly split** ho jata hai — na ki sirf ek single "next" server pe dump ho jaye. Isse failure ka impact bhi evenly absorb hota hai.

---

## 5. Quick real-world mapping

| System | Consistent Hashing kaha use hota hai |
|---|---|
| DynamoDB | Partition placement across storage nodes |
| Cassandra | Ring architecture, "vnodes" (virtual nodes) |
| CDNs (Akamai, Cloudflare) | Request routing to edge servers |
| Load balancers | Sticky sessions — same user, same backend server |

---

## Quick Self-Check
1. Ring pe server aur key dono ko place karne ke liye kya use karte hai?
2. Key assign karne ka rule kya hai (clockwise wala)?
3. Jab ek naya server add hota hai, exactly kitni keys move hoti hai aur kyu?
4. Virtual nodes ka problem kya solve karte hai?
5. Agar ek physical server fail ho jaye (with virtual nodes), uska load kaise distribute hota hai?