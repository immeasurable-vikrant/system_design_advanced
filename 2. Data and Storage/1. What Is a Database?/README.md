# 01 — What Is a Database? (From Absolute Zero)

> **Is lesson me:** Data "persist" karna kyu zaroori hai, JS object/file me data rakhne se kya problem hai, DBMS kya hota hai, aur databases ke kitne types hai (overview). Koi prior knowledge assume nahi.

Prerequisites: none — ye sabse pehli file hai.

---

## 1. Ek problem se shuru karte hai

Socho tum ek chhota app bana rahe ho — ek to-do list. Frontend me tum aise likhoge:

```js
let todos = [];
todos.push({ id: 1, text: "Buy milk", done: false });
```

Ye kaam karta hai... jab tak app khula hai. Ab ye 4 problems aayenge:

**Problem 1 — Data gayab ho jata hai (no persistence)**
Jaise hi tum browser tab band karo ya server restart ho, `todos` array **RAM (memory)** me tha, aur RAM clear ho jaata hai. Saara data gone.

> **RAM kya hai?** RAM (Random Access Memory) tumhare computer ki "working memory" hai — bahut fast, but **temporary**. Jab program band ho ya power jaye, RAM khali ho jaati hai. Isliye JS variables, arrays, React state — sab RAM me hote hai aur temporary hote hai.
>
> **Disk (SSD/HDD)** iska ulta hai — slower, but **permanent**. Tumhari files, photos, installed apps sab disk pe hote hai, power jaane pe bhi bache rehte hai. (VS Code me tumhara code disk pe save hota hai — isliye laptop band karke kholne pe bhi wahin milta hai.)

**Problem 2 — Do log ek saath likhe toh gadbad (no concurrency control)**
Agar 100 users ek saath `todos.push()` kare, ya do requests same time pe same data change kare — data corrupt ho sakta hai (race condition). JS single-threaded hai isliye tumne ye kabhi nahi dekha, but backend pe ye real problem hai.

**Problem 3 — Bahut saara data me dhundhna slow (no fast querying)**
Agar 10 million todos ho aur tumhe "saare done=false wale" chahiye, tumhe har ek check karna padega — `array.filter()` 10 million items pe. Slow.

**Problem 4 — Crash ke beech me data corrupt (no durability)**
Agar likhte-likhte power chali jaye, aadha-adhura data bach sakta hai, ya sab kharab ho sakta hai.

**Database in chaaron problems ko solve karta hai.** Yehi uska poora reason-for-existence hai.

---

## 2. Database kya hai (definition)

**Database** = ek organized collection of data jo:
- **Persistently** store hoti hai (disk pe — crash/restart ke baad bhi safe)
- **Concurrently** access ho sakti hai (bahut users/processes ek saath, safely)
- **Efficiently** query ho sakti hai (millions of records me bhi fast)
- **Integrity** maintain karti hai (data valid rehta hai, aadha-adhura nahi)

Simple mental model: database = ek super-powered, persistent, multi-user version of your data structures (arrays/objects), jo disk pe rehta hai aur reliability guarantee karta hai.

---

## 3. DBMS kya hai (database se alag cheez)

Log aksar "MySQL", "Postgres", "MongoDB" ko "database" bol dete hai, but technically:

```
Database  = the actual data (tumhari tables, records, documents)
DBMS      = the SOFTWARE that manages that data
            (MySQL, PostgreSQL, MongoDB — ye DBMS hai)
```

**DBMS (Database Management System)** wo program hai jo:
- Data ko disk pe efficiently organize karta hai (storage management)
- Tumhari queries samajhta aur execute karta hai (query processing)
- Multiple users ke concurrent access ko safe banata hai (concurrency control)
- Transactions ki correctness guarantee karta hai (ACID — lesson 08 me detail)
- Access control/security handle karta hai
- Crash ke baad recover karta hai (backup & recovery)

**Frontend analogy:** DBMS = ek "runtime engine" jaisa hai, bilkul jaise V8 (JS engine) tumhara JS code run karta hai. Tum V8 ko "JavaScript" nahi bolte — wo engine hai jo JavaScript run karta hai. Waise hi Postgres ek engine hai jo tumhare data ko manage karta hai.

Ek aur important baat: DBMS ek **alag se chalne wala program (process)** hai. Jab Postgres install karke chalu karte ho, wo background me ek server ki tarah chalta rehta hai (jaise ek Express server chalta rehta hai), aur wait karta hai ki koi usse connect karke data maange. (Iska poora mechanism lesson 04-05 me.)

---

## 4. File me data kyu nahi rakhte? (DBMS vs plain files)

Tum soch sakte ho "main data ko ek JSON file me likh deta hu disk pe — persistence ho gayi na?" Chalo dekhte hai kyu ye kaafi nahi:

| Plain file (JSON/CSV) | DBMS |
|---|---|
| Data manually organize karna padega | Structured storage (tables/documents) |
| "age > 25 wale users" dhundhna = poori file padho | Powerful queries (indexes se fast) |
| Do writes ek saath = file corrupt | Built-in concurrency control |
| Koi relationship enforcement nahi | Foreign keys, constraints, validation |
| Crash pe data loss | Durability guarantees (WAL — lesson 10) |
| Scale karna painful | Replication, sharding built-in support |

Bottom line: **DBMS sirf "data rakhne ki jagah" nahi hai — ye ek poora engine hai jo correctness, speed, aur durability guarantee karta hai, especially scale pe.**

---

## 5. Databases ke types — bird's eye view

Sabse pehla bada fork jab koi database choose karta hai: **SQL vs NoSQL.**

```
                    Databases
                   /          \
                SQL            NoSQL
            (Relational)    (Non-Relational)
```

### SQL / Relational (RDBMS)
- Data **tables** (rows + columns) me, fixed schema (columns predefined)
- Tables relationships se juDe (foreign keys)
- Query language: **SQL**
- Strong consistency, ACID transactions
- Examples: **PostgreSQL, MySQL, Oracle, SQL Server**
- *(Ye lesson 02-03 me detail me — ye tumhara bread-and-butter hoga)*

### NoSQL / Non-Relational
"NoSQL" ek single cheez nahi — ye ek **umbrella term** hai un databases ke liye jo traditional table-structure follow nahi karte. Flexible schema, horizontal scaling built-in. 4 major types:

| Type | Kaise store karta hai | Examples | Kab use |
|---|---|---|---|
| **Key-Value** | `key → value` (simplest) | Redis, DynamoDB | Cache, sessions, fast lookups |
| **Document** | JSON-like flexible documents | MongoDB, Couchbase | Flexible/evolving data, catalogs |
| **Column-Family** | Data columns me grouped, huge write throughput | Cassandra, HBase | Feeds, timelines, massive writes |
| **Graph** | Nodes + edges (relationships) | Neo4j, Neptune | "Friends of friends", networks |

*(Time-series DBs jaise InfluxDB/TimescaleDB kabhi-kabhi 5th category mana jaata hai — timestamp-heavy data ke liye.)*

### Quick SQL vs NoSQL comparison

| | SQL (RDBMS) | NoSQL |
|---|---|---|
| Schema | Fixed | Flexible |
| Scaling | Vertical (mostly) | Horizontal (built for it) |
| Consistency | Strong (ACID) | Often eventual (BASE) |
| Relationships | JOINs | Denormalized / embedded |
| Best for | Transactions, structured data | Scale, flexible data |

> **Golden line (interview ke liye):** "SQL vs NoSQL" is *not* "old vs new" ya "bad vs good" — it's a **trade-off based on your access pattern**. Real systems dono mix karte hai (**polyglot persistence**) — e.g., Postgres for orders + Redis for cache + Elasticsearch for search, ek hi product me.

Har type kab exactly choose karna hai — iska poora decision framework lesson 14 me hai (jab tum consistency, scaling, sab samajh chuke hoge).

---

## Quick Self-Check
1. RAM aur disk me fundamental difference kya hai, aur database disk kyu use karta hai?
2. "Database" aur "DBMS" me difference kya hai — ek example do.
3. Plain JSON file me data rakhne ke 4 problems kya hai jo DBMS solve karta hai?
4. SQL aur NoSQL ka core difference kya hai (schema + scaling ke terms me)?
5. NoSQL ke 4 types kaunse hai, ek-ek example ke saath?
6. "Polyglot persistence" ka matlab kya hai?

---
**Next:** `02-RDBMS-Tables-Keys-Relationships.md` — ab RDBMS ko detail me kholte hai (tables, keys, relationships).