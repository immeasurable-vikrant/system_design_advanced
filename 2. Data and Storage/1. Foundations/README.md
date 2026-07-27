# Database Fundamentals + Sharding Prerequisites

> Ye doc do sections me split hai. **Section A** basics cover karta hai — what is a database, DBMS, SQL vs NoSQL — jo indexing article me assume kar liya gaya tha. **Section B** specifically wahi cheezein hai jo "Database Sharding" article ne prerequisite bola hai: Database Concepts (CAP, replication/partitioning terminology) + Consistent Hashing. Agar Section A already clear hai tumhe, seedha Section B pe jump kar sakte ho.

Prerequisites for this doc: none
Feeds into: Database Indexing (already read), Database Sharding (next), Key-Value Store, Chat System, URL Shortener

---

# SECTION A: Fundamentals

## A1. What is a Database?

Database ek **organized collection of data** hai jo persistently (disk pe, crash ke baad bhi safe) store hoti hai, aur jisse efficiently query/update kiya ja sake.

Compare karke socho: agar tum data ko ek simple text file ya in-memory JS object me rakhoge, toh:
- App restart hone pe data gayab (no persistence)
- Multiple users/processes ek saath safely read/write nahi kar payenge (no concurrency control)
- Millions of records me kuch dhundhna slow hoga (no indexing/query engine)
- Partial failure pe data corrupt ho sakta hai (no durability guarantees)

Database ye saari problems solve karta hai — persistence, concurrent access, fast querying, aur data integrity.

## A2. What is a DBMS?

**DBMS (Database Management System)** wo software hai jo database ko manage karta hai — actual engine jo data store, retrieve, update, aur secure karta hai. Jab log bolte hai "MySQL" ya "MongoDB", woh DBMS ka naam hai, database khud tumhara actual data hai jo us DBMS ke andar rehta hai.

```
Database = the data itself (tables, documents, records)
DBMS     = the software that manages that data (MySQL, MongoDB, Postgres, etc.)
```

DBMS ka kaam:
- **Storage management** — disk pe data ko efficiently organize karna
- **Query processing** — SQL/query language ko samajhna aur execute karna
- **Concurrency control** — multiple users ek saath read/write kare toh conflicts avoid karna
- **Transaction management** — ACID guarantees dena (agla section me detail)
- **Security** — access control, authentication
- **Backup & recovery** — crash ke baad data restore karna

## A3. Why do we even use a Database (instead of files)?

| Without DBMS (flat files) | With DBMS |
|---|---|
| Manual data organization | Structured storage (tables/documents) |
| No easy way to query ("find all users older than 25") | Powerful query languages (SQL, etc.) |
| Concurrent writes can corrupt data | Built-in concurrency control |
| No relationship enforcement | Foreign keys, constraints, validations |
| Data loss risk on crash | Durability guarantees (WAL, replication) |
| Scaling is painful | Built-in replication, sharding, indexing support |

Interview me ye samajhna important hai: **DBMS just isn't "a place to put data" — it's an entire engine that guarantees correctness, speed, and durability at scale.**

## A4. SQL vs NoSQL — the big picture split

Sabse pehla fork jo koi bhi database choose karte waqt aata hai:

```
                    Databases
                   /          \
                SQL            NoSQL
            (Relational)    (Non-Relational)
```

### SQL (Relational / RDBMS)
- Fixed schema — tables ke columns predefined hote hai
- Data rows/columns me, relationships via **foreign keys**
- Query language: SQL (Structured Query Language)
- Strong consistency, ACID transactions by default
- Scaling: mostly vertical (bigger machine), horizontal scaling harder (that's literally why "sharding" is a whole separate deep topic!)

```
Examples: PostgreSQL, MySQL, Oracle, SQL Server
```

### NoSQL (Non-Relational)
"NoSQL" khud ek single type nahi hai — ye ek **umbrella term** hai multiple different data models ke liye jo traditional RDBMS structure follow nahi karte. Schema flexible hota hai, aur horizontal scaling built-in design goal hota hai.

NoSQL ke andar 4 major categories hai:

| NoSQL Type | Description | Examples |
|---|---|---|
| **Key-Value** | key → value, simplest model | Redis, DynamoDB, Memcached |
| **Document** | JSON-like flexible documents | MongoDB, Couchbase |
| **Column-Family** | Data grouped by columns, huge write throughput | Cassandra, HBase, Bigtable |
| **Graph** | Nodes + edges, relationship-heavy queries | Neo4j, Amazon Neptune |

(Time-series DBs like InfluxDB/TimescaleDB are sometimes counted as a 5th category too, optimized for timestamp-indexed append-heavy data.)

### Quick comparison

| | SQL (RDBMS) | NoSQL |
|---|---|---|
| Schema | Fixed | Flexible/dynamic |
| Scaling | Vertical (mostly) | Horizontal (built for it) |
| Consistency | Strong (ACID) | Often eventual (BASE) |
| Relationships | Joins | Denormalized / embedded |
| Best for | Transactions, structured data | Scale, flexible/evolving data |

**Interview me golden line:** "SQL vs NoSQL" isn't "old vs new" or "bad vs good" — it's a trade-off based on your access pattern. Real systems mix both (polyglot persistence) — e.g., Postgres for orders/transactions + Redis for caching + Elasticsearch for search, all in the same product.

---

# SECTION B: Prerequisites for "Database Sharding" article

Ye woh concepts hai jo Sharding article ne explicitly prerequisite bola hai: **Database Concepts** aur **Consistent Hashing**.

## B1. ACID vs BASE

Ye do consistency philosophies hai jo batati hai database writes/transactions ko kaise handle karega.

### ACID (traditional RDBMS ka promise)
- **Atomicity** — Transaction ya toh pura hoga, ya bilkul nahi hoga (partial nahi)
- **Consistency** — Transaction database ko ek valid state se dusre valid state me le jayega
- **Isolation** — Concurrent transactions ek dusre ko interfere nahi karenge
- **Durability** — Ek baar commit ho gaya, toh data permanently safe hai (crash ke baad bhi)

### BASE (distributed NoSQL systems ka philosophy — scale ke liye practical trade-off)
- **Basically Available** — System mostly available rahega
- **Soft state** — State time ke saath change ho sakta hai even without input (replication lag ke wajah se)
- **Eventual consistency** — Agar naya write na ho, toh eventually saare replicas same value dikhayenge — turant nahi

**Remember:** ACID = correctness first, availability second. BASE = availability first, correctness eventually.

## B2. CAP Theorem

Distributed database sirf 2 out of 3 guarantee kar sakta hai, teeno simultaneously nahi:

- **Consistency (C)** — Har read latest write dekhega
- **Availability (A)** — Har request ko response milega (error nahi)
- **Partition Tolerance (P)** — System network split ke bawajood kaam karega

Real world me **network partition hamesha ho sakta hai**, so effectively choice sirf **CP vs AP** ke beech hoti hai:

```
CP systems (consistency over availability during partition):
  Examples: MongoDB (default config), HBase, Zookeeper
  Use case: Banking, inventory — stale data dikhana galat hai

AP systems (availability over consistency during partition):
  Examples: Cassandra, DynamoDB, CouchDB
  Use case: Social media feed, shopping cart — thoda stale chalega,
            but system down nahi hona chahiye
```

## B3. Replication vs Partitioning vs Sharding — terminology jo mostly confuse hoti hai

Ye teeno alag concepts hai but log inko interchangeably use kar dete hai. Sharding article se pehle ye clear karna zaroori hai:

- **Replication** — Same data ki **multiple copies** different machines pe rakhna (fault tolerance + read scaling ke liye). Data same rehta hai har jagah.
- **Partitioning** — Data ko **logical chunks** me todna based on some rule (e.g., by date range, by category).
- **Sharding** — Partitioning ka ek specific type jaha alag partitions **alag physical machines/servers** pe rakhe jaate hai — horizontal scaling ke liye. Har shard independent database hota hai apne aap me.

```
Replication:  [Server A: full data] → [Server B: full data copy] → [Server C: full data copy]
Partitioning: [Data: rows 1-1000] → [Chunk 1] , [rows 1001-2000] → [Chunk 2]
Sharding:     [Chunk 1 → Server A] , [Chunk 2 → Server B] , [Chunk 3 → Server C]
```

Real systems dono combine karte hai: data ko shard karo scaling ke liye, phir har shard ko replicate karo fault tolerance ke liye.

## B4. Consistent Hashing (Sharding ka direct prerequisite — deep dive)

Ye woh concept hai jo actually sharding article me heavily use hoga.

### Problem jo consistent hashing solve karta hai

Naive approach: `server = hash(key) % number_of_servers`

Ye kaam toh karta hai, but jaise hi tum ek server add ya remove karte ho, `number_of_servers` change ho jata hai — aur **almost saari keys ka mapping badal jata hai**. Matlab ek server add karne pe bhi 90%+ data re-shuffle karna padega. Production me ye disaster hai — cache invalidation, massive data movement, downtime.

```
Example with naive modulo hashing (3 servers):
  hash("user123") % 3 = 1  → Server 1
  hash("user456") % 3 = 0  → Server 0

Now add a 4th server:
  hash("user123") % 4 = 3  → Server 3  (moved!)
  hash("user456") % 4 = 2  → Server 2  (moved!)

Almost every key remaps → massive unnecessary data movement.
```

### Consistent Hashing ka solution

Idea: Servers aur keys, dono ko ek **hash ring** (0 se 2^32-1 tak circular space) pe map karo. Key apne aage jo bhi pehla server milta hai ring pe clockwise, usko assign hoti hai.

```
Hash Ring (0 to 2^32-1, circular):

         Server A (hash=10)
        /                  \
  Server D (hash=300)    Server B (hash=80)
        \                  /
         Server C (hash=180)

Key "user123" → hash = 45 → next server clockwise = Server B (80)
Key "user456" → hash = 200 → next server clockwise = Server D (300)
```

**Jab ek server add/remove hota hai:**

```
Remove Server B (hash=80):
  Only keys that were mapped to B (i.e., keys between A and B) move to next server (C).
  Baaki sab keys unaffected!

→ Only ~1/N keys move (N = number of servers), instead of almost all keys.
```

Ye hi consistent hashing ka core benefit hai: **minimal data movement on scale up/down.**

### Virtual Nodes (real implementations ka important detail)

Problem: Agar sirf 4-5 physical servers ring pe randomly place ho, toh unke beech ka gap uneven ho sakta hai — matlab kuch servers zyada load le lenge (hot spot).

Solution: Har physical server ko ring pe **multiple virtual points** (e.g., 100-200 virtual nodes per server) pe place karo. Isse load zyada uniformly distribute hota hai, aur agar ek server fail ho, toh uska load bohot saare doosre servers me evenly baant jata hai (na ki sirf ek adjacent server pe).

```
Without virtual nodes: Server A gets a big chunk (uneven), Server B a small chunk
With virtual nodes:    Server A's 150 virtual points scattered around ring
                        → load evenly spread, failure impact evenly spread too
```

### Real-world usage
- **DynamoDB** — partitioning strategy consistent hashing pe based hai
- **Cassandra** — ring-based architecture, virtual nodes (called "vnodes") use karta hai
- **CDNs** (like Akamai) — request routing ke liye consistent hashing use karte hai
- **Load balancers** — sticky sessions ke liye bhi ye pattern use hota hai

### Interview one-liner
> "Consistent hashing solves the re-sharding problem — jab servers add/remove hote hai, sirf O(K/N) keys move hoti hai (K=total keys, N=servers), instead of almost all keys with naive modulo hashing. Virtual nodes load ko uniformly distribute karte hai aur hot-spotting avoid karte hai."

---

## Quick Self-Check Before Reading "Database Sharding"

**Section A (fundamentals):**
1. Database vs DBMS — difference kya hai?
2. SQL vs NoSQL choose karte waqt kya trade-offs dekhte ho?
3. NoSQL ke 4 major categories kaun se hai aur ek-ek example?

**Section B (sharding prerequisites):**
4. ACID vs BASE — trade-off kya hai?
5. CAP theorem me during a network partition, kya choice hoti hai — C ya A?
6. Replication, Partitioning, aur Sharding — teeno me exact difference kya hai?
7. Naive `hash % N` approach me problem kya hai jab server count change hota hai?
8. Consistent hashing isse kaise solve karta hai, aur virtual nodes kyu zaroori hai?

---

## Suggested Reading Order (for your prep)

```
1. Database Fundamentals (Section A)                 ✅ done
2. Sharding Prerequisites — CAP, Consistent Hashing   ✅ done (Section B)
3. Database Indexing — Complete Deep Dive             ✅ already read
4. Database Sharding — Complete Deep Dive             → next
5. Key-Value Store (HLD)
6. URL Shortener (HLD)
7. Chat System (HLD)
```