# 02 — RDBMS: Tables, Keys, Relationships & Normalization

> **Is lesson me:** Relational database ka poora structure — tables/rows/columns, primary key, foreign key, relationships (1-to-1, 1-to-many, many-to-many), aur normalization basics (data ko sahi tarike se organize karna). Ye sab lesson 03 (SQL) ki foundation hai.

Prerequisites: `01-What-Is-A-Database.md`

---

## 1. Table, Row, Column — the building blocks

Relational database me data **tables** me hota hai. Ek table ek spreadsheet jaisa hai:

```
Table: users
┌──────┬───────────┬─────────────────┬──────┐
│ id   │ username  │ email           │ age  │   ← COLUMNS (fields)
├──────┼───────────┼─────────────────┼──────┤
│ 1    │ bob_123   │ bob@mail.com    │ 25   │   ← ROW (one record)
│ 2    │ alice_x   │ alice@mail.com  │ 30   │   ← ROW
│ 3    │ eve_99    │ eve@mail.com    │ 22   │   ← ROW
└──────┴───────────┴─────────────────┴──────┘
```

- **Column** = ek field/attribute (jaise `email`). Har column ka ek **data type** hota hai (integer, text, boolean, timestamp, etc.) — ye fixed hai (yehi "fixed schema" ka matlab).
- **Row** (aka record/tuple) = ek single entry (jaise ek poora user).
- **Table** = rows ka collection, same columns ke saath.

**Frontend analogy:** Ek table = ek array of objects jaha har object ki **exactly same keys** ho:
```js
// Ye ek "table" jaisa hai:
const users = [
  { id: 1, username: "bob_123", email: "bob@mail.com", age: 25 },
  { id: 2, username: "alice_x", email: "alice@mail.com", age: 30 },
];
// har object = ek row, keys = columns, saari objects ki keys same = fixed schema
```

---

## 2. Primary Key — har row ki unique identity

**Primary key** = ek column (ya columns ka combination) jo har row ko **uniquely identify** karta hai. Do rows ka same primary key nahi ho sakta.

```
Table: users — primary key = id
  id=1 → Bob    (koi aur row id=1 nahi ho sakti)
  id=2 → Alice
```

Rules:
- **Unique** — koi duplicate nahi
- **Not null** — kabhi khali nahi ho sakta
- **Stable** — ideally kabhi change nahi hona chahiye

Usually ye ek auto-incrementing `id` (1, 2, 3...) hota hai, ya ek UUID (`"a3f9-..."`). Kyu? Kyunki `email` bhi unique ho sakta hai, but log email change kar lete hai — primary key stable hona chahiye, isliye ek meaningless-but-permanent `id` best hota hai.

**Frontend analogy:** React me tum list render karte waqt `key={user.id}` dete ho — wo `id` exactly ek primary key hi hai, jo har item ko uniquely identify karta hai taaki React unhe track kar sake.

---

## 3. Foreign Key — tables ko jodna (the "Relational" part)

Yahi wo cheez hai jo relational database ko "relational" banati hai.

Socho tumhe posts bhi store karne hai. Har post kisi user ne banayi hai. Tum har post me poora user object copy nahi karoge (waste + update nightmare) — tum bas **user ka id** store karoge:

```
Table: users                    Table: posts
┌────┬──────────┐              ┌────┬─────────┬────────────┐
│ id │ username │              │ id │ user_id │ caption    │
├────┼──────────┤              ├────┼─────────┼────────────┤
│ 1  │ bob_123  │◄─────────────│ 101│ 1       │ "Sunset!"  │
│ 2  │ alice_x  │◄──────┐      │ 102│ 1       │ "Coffee"   │
└────┴──────────┘       └──────│ 103│ 2       │ "Hello"    │
                               └────┴─────────┴────────────┘
                                        │
              posts.user_id ────────────┘ POINTS TO users.id
              = this is a FOREIGN KEY
```

**Foreign key** = ek column jo doosri table ke primary key ko point karta hai. Yaha `posts.user_id` ek foreign key hai jo `users.id` ko reference karta hai.

Iska matlab: post 101 aur 102, user 1 (Bob) ke hai; post 103, user 2 (Alice) ka hai.

**Referential integrity:** Database ye relationship **enforce** karta hai. Agar tum `posts` me `user_id = 999` daalne ki koshish karo aur user 999 exist hi nahi karta — database **reject** kar dega. Isse "orphan" records (jinke parent hi na ho) ban hi nahi sakte. Ye ek bahut bada benefit hai jo plain files/objects nahi dete.

**Frontend analogy:** Ye bilkul waisa hai jaise Redux normalized state — tum `posts` me poora user object nahi rakhte, sirf `userId` rakhte ho, aur render time pe `users[post.userId]` se lookup karte ho. Database ye pattern natively support karta hai aur guarantee bhi deta hai ki reference kabhi broken na ho.

---

## 4. Relationships ke types

### One-to-Many (sabse common)
Ek user ke **kayi** posts ho sakte hai, but ek post sirf **ek** user ka hota hai.
```
users (one) ──────< posts (many)
   Bob ──────┬──── post 101
             ├──── post 102
             └──── post 105
```
Implementation: "many" side (posts) me foreign key (`user_id`) rakho. Yehi upar wala example hai.

### One-to-One
Ek user ka exactly **ek** profile, aur ek profile exactly **ek** user ka.
```
users (one) ────── user_profiles (one)
   Bob ─────────── Bob's profile (bio, avatar, etc.)
```
Implementation: kisi ek table me foreign key + unique constraint. (Aksar performance/organization ke liye ek bade table ko do me toD dete hai.)

### Many-to-Many
Ek student kayi courses le sakta hai, aur ek course me kayi students ho sakte hai.
```
students (many) ────< enrollments >──── courses (many)
```
Implementation: ek **junction/join table** banate hai jo dono ke foreign keys rakhta hai:
```
Table: enrollments
┌────────────┬───────────┐
│ student_id │ course_id │   ← dono foreign keys
├────────────┼───────────┤
│ 1          │ 50        │   (student 1, course 50)
│ 1          │ 51        │   (student 1 bhi course 51 me)
│ 2          │ 50        │   (student 2 bhi course 50 me)
└────────────┴───────────┘
```

**Frontend analogy:** Many-to-many ka junction table = React me jab tum do lists ke beech mapping rakhte ho ek third array me, jaise `[{userId, roleId}]` — same idea.

---

## 5. Normalization — data ko sahi tarike se organize karna

**Normalization** = data ko is tarike se organize karna ki **duplication (redundancy) minimum ho** aur har fact **sirf ek jagah** stored ho.

### Problem: un-normalized (denormalized) table
```
Table: orders (BAD — everything in one table)
┌────────┬──────────┬───────────────┬─────────────┐
│orderId │ userName │ userEmail     │ product     │
├────────┼──────────┼───────────────┼─────────────┤
│ 1      │ Bob      │ bob@mail.com  │ Laptop      │
│ 2      │ Bob      │ bob@mail.com  │ Mouse       │  ← Bob ka data DUPLICATE
│ 3      │ Bob      │ bob@mail.com  │ Keyboard    │  ← phir se duplicate
└────────┴──────────┴───────────────┴─────────────┘
```
Problems:
- Bob ne apna email change kiya? **3 jagah** update karna padega. Ek bhi miss ho gayi → inconsistent data.
- Storage waste — same data baar-baar.

### Solution: normalized (split into related tables)
```
Table: users                Table: orders
┌────┬──────┬──────────────┐  ┌────────┬─────────┬───────────┐
│ id │ name │ email        │  │orderId │ user_id │ product   │
├────┼──────┼──────────────┤  ├────────┼─────────┼───────────┤
│ 1  │ Bob  │ bob@mail.com │  │ 1      │ 1       │ Laptop    │
└────┴──────┴──────────────┘  │ 2      │ 1       │ Mouse     │
                              │ 3      │ 1       │ Keyboard  │
                              └────────┴─────────┴───────────┘
```
Ab Bob ka email **sirf ek jagah** (`users` table me). Change karna ho → ek hi update. Orders bas `user_id` se link hai.

**Normal forms (bas awareness ke liye — ratne ki zarurat nahi abhi):**
- **1NF:** har cell me single value (arrays/lists nahi), har row unique
- **2NF:** 1NF + har non-key column poore primary key pe depend kare
- **3NF:** 2NF + non-key columns ek dusre pe depend na kare (sirf key pe)

Practically: **3NF tak normalize karna** most apps ke liye sweet spot hai. "Har fact ek jagah, foreign keys se link" — bas yehi mindset kaafi hai shuru me.

### Important: Normalization vs Denormalization is a TRADE-OFF
- **Normalized** = no duplication, easy updates, **but** reads me JOINs lagte hai (multiple tables combine karna — slow-ish).
- **Denormalized** = duplicate data, fast reads (no JOIN), **but** updates painful (har copy update karo).

Chhote/normal systems: normalize karo (default). Massive-scale systems (jaise sharded databases): kabhi-kabhi deliberately denormalize karte hai fast reads ke liye — ye lesson 13 (Sharding) me detail me aayega. Abhi bas ye samajh lo ki dono valid choices hai, context pe depend karta hai.

---

## Quick Self-Check
1. Table, row, column — teeno ko ek array-of-objects analogy se explain karo.
2. Primary key ke 3 rules kya hai, aur email ko primary key kyu nahi banate?
3. Foreign key kya hai, aur "referential integrity" kya guarantee karta hai?
4. One-to-many, one-to-one, many-to-many — teeno kaise implement hote hai?
5. Normalization kya solve karta hai? Ek un-normalized table ka problem batao.
6. Normalized vs denormalized ka core trade-off kya hai?

---
**Next:** `03-SQL-Language-Basics.md` — ab in tables se baat kaise karte hai (SQL).