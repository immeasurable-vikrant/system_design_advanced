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

Ye woh concepts hai jo Sharding article ne explicitly prerequisite bola hai: **Database Concepts** aur **Consistent Hashing**. Yaha bhi bilkul scratch se explain kar raha hu, jaise Section A me kiya tha.

## B1. Why does scaling a database even become a problem?

Pehle ye samajhte hai ki ye saare concepts (replication, partitioning, sharding) exist hi kyu karte hai.

Ek single database server ki do fundamental limits hai:
- **Storage limit** — ek machine ke disk pe infinite data nahi rakh sakte
- **Throughput limit** — ek machine sirf itne hi reads/writes per second handle kar sakta hai (CPU, RAM, disk I/O sab finite hai)

Jab tumhara app grow karta hai (millions of users, jaise Instagram/Uber scale), single server ki capacity khatam ho jati hai. Do broad solutions hai:

1. **Vertical scaling** — bada machine le lo (more CPU/RAM/disk). Simple, but ek limit ke baad aur bada machine milta hi nahi (aur mehenga bohot hota hai).
2. **Horizontal scaling** — multiple machines use karo instead of one. Yahi se **replication** aur **sharding** dono born hote hai — dono horizontal scaling ke tools hai, but alag problems solve karte hai:
   - Replication → **read throughput + fault tolerance** ka solution
   - Sharding → **storage + write throughput** ka solution

## B2. What is Replication? (from scratch)

**Replication** matlab ek hi data ki **multiple copies** alag-alag machines (called "replicas" or "nodes") pe maintain karna.

```
                  writes
                    │
                    ▼
              [Leader / Primary]
              (holds the master copy)
                 /        \
        replicate          replicate
               /              \
    [Replica 1]              [Replica 2]
    (follower)                (follower)
```

### Why replicate at all?
1. **Fault tolerance** — agar ek server crash ho jaye, dusra server pe data already available hai (no data loss, no downtime)
2. **Read scaling** — reads ko multiple replicas pe distribute kar sakte ho (leader sirf writes handle kare, followers reads serve kare) — ek hi machine par saara read load nahi padta
3. **Geographic locality** — replicas ko different regions me rakh ke, users ko unke nearest server se serve kar sakte ho (lower latency)

**Important:** Replication storage capacity nahi badhata — har replica pe **poora hi data** hota hai, sirf copies badh rahi hai. (Isko sharding se confuse mat karna — wo agla topic hai.)

### Leader-Follower (Primary-Replica) Model — most common pattern
- **Leader (Primary)**: saare **writes** yahi accept karta hai
- **Followers (Replicas)**: leader se changes copy karte hai (replicate), aur **reads** serve karte hai

```
Client writes  → Leader
Client reads   → Leader OR any Follower
```

### Synchronous vs Asynchronous Replication

Ye decide karta hai ki leader, follower ko update hone ka wait karega ya nahi, before confirming a write to the client.

```
SYNCHRONOUS:
  Client write → Leader writes → waits for Follower to confirm → THEN success response to client
  ✅ Strong consistency (follower always up to date)
  ❌ Slower writes (network round-trip to follower required)
  ❌ Agar follower down ho, write bhi block ho sakta hai

ASYNCHRONOUS:
  Client write → Leader writes → success response to client IMMEDIATELY (follower update background me)
  ✅ Fast writes (no waiting)
  ❌ Follower thodi der ke liye purana data dikha sakta hai
  ❌ Agar leader crash ho jaye before replicating, unreplicated writes lose ho sakte hai
```

Most production systems (jaise MySQL default, Postgres default) **asynchronous** replication use karte hai — speed ke liye thoda consistency trade-off accept karte hai.

### Replication Lag (ye specifically tumne poocha tha)

**Replication lag** = time delay between jab leader pe koi write hota hai, aur jab wahi change follower pe visible hota hai.

```
t=0ms:    Client writes "balance = 500" to Leader
t=0ms:    Leader confirms write to client (async replication)
t=50ms:   Follower actually receives and applies the update

→ Replication lag = 50ms (in this window, Follower still shows OLD data)
```

**Ye ek real, important problem hai kyu?** Kyunki agar tum reads ko follower se serve kar rahe ho (read scaling ke liye), toh us lag window me client ko **stale (purana) data** mil sakta hai.

```
Real example:
  User posts a comment (write → Leader)
  User immediately refreshes page (read → Follower, but replication lag chal raha hai)
  → User apna hi comment nahi dekh pata for a few hundred ms!

  Yahi wo problem hai jiska naam hai "read-your-own-writes inconsistency"
```

**Replication lag increase kab hota hai?**
- Follower slow hardware pe ho ya overloaded ho
- Network issues leader-follower ke beech
- High write volume — follower catch-up nahi kar pa raha

**Common fixes:**
- Critical reads (jaise "apna khud ka data") ko hamesha leader se serve karo
- "Read-your-writes" consistency guarantee implement karo (recent writes wale user ko leader se hi serve karo, thodi der ke liye)
- Lag ko monitor karo, aur agar lag threshold cross kare toh us follower ko traffic serve karna band kar do

### Other replication topologies (just be aware of these — interview me naam aa sakta hai)
- **Single-Leader** — ek hi leader, jo humne abhi discuss kiya (most common)
- **Multi-Leader** — multiple leaders, each accepting writes (useful multi-datacenter setups me), but conflict resolution complex ho jata hai
- **Leaderless** — koi fixed leader nahi, client directly multiple replicas ko write karta hai, quorum-based reads/writes (e.g., Cassandra, DynamoDB use this)

## B3. What is Partitioning? (from scratch)

Ab dusra problem: replication se storage capacity nahi badhti — har replica pe poora data hota hai. Agar tumhara data itna bada ho jaye ki **ek machine ki disk pe fit hi na ho**, tab **partitioning** chahiye.

**Partitioning** matlab data ko **logical chunks (partitions)** me todna, based on some rule — e.g., user_id range, date range, geographic region, hash of a key, etc.

```
Example: 100 million users ka data
  Partition 1: user_id 1 - 25M
  Partition 2: user_id 25M - 50M
  Partition 3: user_id 50M - 75M
  Partition 4: user_id 75M - 100M
```

Ye abhi tak sirf ek **logical/conceptual split** hai — partitions physically kaha store honge, ye agla concept (sharding) decide karta hai.

### Partitioning strategies (high-level, sharding article me detail milega)
- **Range-based** — key ki value ke range ke hisaab se (e.g., dates, IDs). Simple, but "hot partition" ka risk — agar ek range me traffic zyada ho.
- **Hash-based** — key ko hash karke uske basis pe partition decide karna. Load evenly distribute hota hai, but range queries slow ho jaati hai (data scattered ho jata hai).
- **Directory-based** — ek separate lookup service maintain karta hai ki kaunsi key kaunse partition me hai. Flexible, but ye lookup service khud ek bottleneck/single-point-of-failure ban sakta hai agar sahi se design na ho.

## B4. What is Sharding? (from scratch)

**Sharding** = Partitioning ka wo version jaha har partition **alag physical server (machine)** pe rakha jata hai. Har shard apne aap me ek **independent, self-contained database** hota hai — apna storage, apna compute, apna I/O.

```
                    [ Router / Query Coordinator ]
                    (decides which shard to hit)
                   /          |            \
          [Shard 1]      [Shard 2]      [Shard 3]
          (Server A)     (Server B)     (Server C)
          user_id         user_id        user_id
          1-25M           25M-50M        50M-75M
```

### Sharding kya solve karta hai jo replication nahi karta?
- **Storage scaling** — total data N machines me split hai, so ek machine ki disk limit se bound nahi ho
- **Write throughput scaling** — writes bhi multiple machines me distribute ho jaate hai (replication me saare writes sirf ek leader pe jaate the!)

### Sharding ke apne challenges (jo actual "Database Sharding" article me detail me cover honge)
- **Cross-shard queries/joins** — agar data do alag shards me split ho, unke beech join karna expensive/complex hai
- **Rebalancing** — jab ek shard bohot bada ho jaye ya naya server add karna ho, data ko move karna padta hai (yahi exact jagah pe **Consistent Hashing** kaam aata hai — next section!)
- **Choosing the right shard key** — galat shard key choose karne se "hot shard" problem ho sakta hai (ek shard pe disproportionate traffic)

**Ek line me:** Replication = same data, multiple copies, for fault-tolerance + read-scaling. Sharding = different data, different machines, for storage + write-scaling. Real production systems dono use karte hai together — data ko shard karo, phir har shard ko replicate bhi karo.

## B5. ACID vs BASE

Ye do consistency philosophies hai jo batati hai database writes/transactions ko kaise handle karega.

### ACID (traditional RDBMS ka promise)
- **Atomicity** — Transaction ya toh pura hoga, ya bilkul nahi hoga (partial nahi)
- **Consistency** — Transaction database ko ek valid state se dusre valid state me le jayega
- **Isolation** — Concurrent transactions ek dusre ko interfere nahi karenge
- **Durability** — Ek baar commit ho gaya, toh data permanently safe hai (crash ke baad bhi)

### BASE (distributed NoSQL systems ka philosophy — scale ke liye practical trade-off)
- **Basically Available** — System mostly available rahega
- **Soft state** — State time ke saath change ho sakta hai even without input (yahi wo replication lag hai jo humne upar discuss kiya!)
- **Eventual consistency** — Agar naya write na ho, toh eventually saare replicas same value dikhayenge — turant nahi

**Remember:** ACID = correctness first, availability second. BASE = availability first, correctness eventually.

## B6. CAP Theorem

Distributed database sirf 2 out of 3 guarantee kar sakta hai, teeno simultaneously nahi:

- **Consistency (C)** — Har read latest write dekhega
- **Availability (A)** — Har request ko response milega (error nahi)
- **Partition Tolerance (P)** — System network split ke bawajood kaam karega (e.g., leader aur follower ke beech network toot jaye)

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

Notice karo — replication lag jo humne B2 me discuss kiya, wahi actually CAP theorem ka real-world manifestation hai: async replication choose karna essentially "availability + eventual consistency" (AP-leaning) choose karna hai.

## B7. Recap — Replication vs Partitioning vs Sharding

Ab teeno concepts detail me dekhne ke baad, ek quick side-by-side:

| | Replication | Partitioning | Sharding |
|---|---|---|---|
| What moves/splits | Nothing — data is copied | Data is logically split | Data is split AND physically distributed |
| Goal | Fault tolerance, read scaling | Organize large datasets logically | Storage + write scaling |
| Each node has | Full copy of all data | A logical subset (may still be same machine) | A physical subset on its own machine |

```
Replication:  [Server A: full data] → [Server B: full data copy] → [Server C: full data copy]
Partitioning: [Data: rows 1-1000] → [Chunk 1] , [rows 1001-2000] → [Chunk 2]
Sharding:     [Chunk 1 → Server A] , [Chunk 2 → Server B] , [Chunk 3 → Server C]
```

Real systems dono combine karte hai: data ko shard karo scaling ke liye, phir har shard ko replicate karo fault tolerance ke liye (so a "shard" itself might have a leader + followers internally).

## B8. Consistent Hashing (Sharding ka direct prerequisite — deep dive)

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
4. Replication kyu ki jaati hai, aur leader-follower model me writes/reads kaha jaate hai?
5. Replication lag kya hai, aur "read-your-own-writes" problem kaise hoti hai?
6. Partitioning aur Sharding me exact difference kya hai?
7. ACID vs BASE — trade-off kya hai?
8. CAP theorem me during a network partition, kya choice hoti hai — C ya A?
9. Naive `hash % N` approach me problem kya hai jab server count change hota hai?
10. Consistent hashing isse kaise solve karta hai, aur virtual nodes kyu zaroori hai?

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