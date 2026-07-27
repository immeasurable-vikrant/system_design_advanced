# Database Concepts + Consistent Hashing (Sharding ke prerequisites)

---

## Part 1: Types of Databases (Overview)

Frontend background se aane wale logo ke liye ye sabse pehla confusion hota hai — "database" bolte hi sabko MySQL/Postgres yaad aata hai, but interview me ye differentiate karna padta hai ki kaunsa DB kis use-case ke liye best hai.

### 1. Relational (SQL / RDBMS)
Data tables me store hota hai, rows aur columns ke saath, with **fixed schema**. Relationships foreign keys se define hote hai.

```
Examples: PostgreSQL, MySQL, Oracle, SQL Server
Best for: Structured data, complex joins, strong consistency requirements
  (e.g., banking, order management, anything with transactions)
```

### 2. Key-Value Store
Sabse simple model — bas ek key aur uske against ek value. No schema, no relationships.

```
Examples: Redis, DynamoDB, Memcached
Best for: Caching, session storage, super-fast lookups
Query pattern: GET key → value (that's it, mostly)
```

### 3. Document Store
JSON-like documents store karte hai — har document ka schema thoda flexible ho sakta hai (schema-less ya semi-structured).

```
Examples: MongoDB, Couchbase
Best for: Content management, catalogs, jab schema evolve hota rahe
Query pattern: Query by any field inside the document
```

### 4. Column-Family / Wide-Column Store
Data columns ke groups (families) me store hota hai instead of rows. Massive write throughput ke liye designed.

```
Examples: Cassandra, HBase, Bigtable
Best for: Time-series data, huge write-heavy workloads, IoT logs
Query pattern: Fast writes, range scans on row key
```

### 5. Graph Database
Data ko nodes (entities) aur edges (relationships) ke form me store karta hai. Relationship-heavy queries (jaise "friends of friends") yaha bohot fast hote hai — SQL me ye multiple joins lagte, graph DB me single traversal.

```
Examples: Neo4j, Amazon Neptune
Best for: Social networks, recommendation engines, fraud detection
```

### 6. Time-Series Database
Timestamp-indexed data ke liye optimized — writes mostly append-only hote hai, aur queries mostly "last N minutes/hours" type hote hai.

```
Examples: InfluxDB, TimescaleDB, Prometheus
Best for: Metrics, monitoring, sensor data
```

### Quick comparison table

| Type          | Schema        | Scaling style      | Best for                         |
|---------------|---------------|---------------------|-----------------------------------|
| Relational    | Fixed         | Vertical (mostly)   | Transactions, strong consistency  |
| Key-Value     | None          | Horizontal, easy    | Caching, fast lookups              |
| Document      | Flexible      | Horizontal          | Semi-structured, evolving data     |
| Column-Family | Flexible      | Horizontal, huge     | Write-heavy, big data              |
| Graph         | Relationship-based | Harder to shard | Connected data, relationships |
| Time-Series   | Fixed (mostly)| Horizontal (time-partitioned) | Metrics, logs        |

**Interview me ye important hai bolna:** "SQL vs NoSQL" ek binary choice nahi hai — real systems polyglot persistence use karte hai (e.g., Postgres for transactions + Redis for caching + Elasticsearch for search, sab ek hi system me).

---

## Part 2: ACID vs BASE

Ye do consistency philosophies hai jo batati hai ki database writes/transactions ko kaise handle karega.

### ACID (traditional RDBMS ka promise)
- **Atomicity**: Transaction ya toh pura hoga, ya bilkul nahi hoga (partial nahi)
- **Consistency**: Transaction database ko ek valid state se dusre valid state me le jayega
- **Isolation**: Concurrent transactions ek dusre ko interfere nahi karenge
- **Durability**: Ek baar commit ho gaya, toh data permanently safe hai (crash ke baad bhi)

### BASE (distributed NoSQL systems ka philosophy — jyada practical trade-off distributed scale ke liye)
- **Basically Available**: System mostly available rahega
- **Soft state**: State time ke saath change ho sakta hai even without input (replication lag ke wajah se)
- **Eventual consistency**: Agar koi naya write na ho, toh eventually saare replicas same value dikhayenge — but turant nahi

Simple way to remember: **ACID = correctness first, availability second. BASE = availability first, correctness eventually.**

---

## Part 3: CAP Theorem (crisp version — indexing doc me isko prerequisite bola tha, so quick recap)

Distributed database sirf 2 out of 3 guarantee kar sakta hai, teeno simultaneously nahi:

- **Consistency (C)**: Har read latest write dekhega
- **Availability (A)**: Har request ko response milega (error nahi)
- **Partition Tolerance (P)**: System network split ke bawajood kaam karega

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

---

## Part 4: Replication vs Partitioning vs Sharding — terminology jo mostly confuse hoti hai

Ye teeno alag concepts hai but log inko interchangeably use kar dete hai. Sharding article se pehle ye clear karna zaroori hai:

- **Replication**: Same data ki **multiple copies** different machines pe rakhna (fault tolerance + read scaling ke liye). Data same rehta hai har jagah.
- **Partitioning**: Data ko **logical chunks** me todna based on some rule (e.g., by date range, by category).
- **Sharding**: Partitioning ka ek specific type jaha alag partitions **alag physical machines/servers** pe rakhe jaate hai — horizontal scaling ke liye. Har shard independent database hota hai apne aap me.

```
Replication:  [Server A: full data] → [Server B: full data copy] → [Server C: full data copy]
Partitioning: [Data: rows 1-1000] → [Chunk 1] , [rows 1001-2000] → [Chunk 2]
Sharding:     [Chunk 1 → Server A] , [Chunk 2 → Server B] , [Chunk 3 → Server C]
```

Real systems dono combine karte hai: data ko shard karo scaling ke liye, phir har shard ko replicate karo fault tolerance ke liye.

---

## Part 5: Consistent Hashing (Sharding ka direct prerequisite — deep dive)

Ye woh concept hai jo actually sharding article me heavily use hoga, so isko thoda detail me samajhte hai.

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

Agar in questions ka answer clear hai, toh tum ready ho next article ke liye:

1. SQL vs NoSQL — kab kaunsa use karoge? (schema flexibility + scaling pattern ke basis pe)
2. ACID vs BASE — trade-off kya hai?
3. CAP theorem me during a network partition, kya choice hoti hai — C ya A?
4. Replication, Partitioning, aur Sharding — teeno me exact difference kya hai?
5. Naive `hash % N` approach me problem kya hai jab server count change hota hai?
6. Consistent hashing isse kaise solve karta hai, aur virtual nodes kyu zaroori hai?

---

## Suggested Reading Order (for your prep)

```
1. Database Concepts (this doc, Parts 1-4)          ✅ done
2. Consistent Hashing (this doc, Part 5)             ✅ done
3. Database Indexing — Complete Deep Dive            ✅ already read
4. Database Sharding — Complete Deep Dive            → next
5. Key-Value Store (HLD)
6. URL Shortener (HLD)
7. Chat System (HLD)
```

Note: Technically indexing aane se pehle ye foundational doc padhna better order hota — but ab tumhare paas dono hai, so sharding padhte waqt terminology friction nahi aayegi.