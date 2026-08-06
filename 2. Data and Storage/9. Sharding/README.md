# 03 — Database Sharding: Complete Deep Dive (Merged: Article + Explanations)

> Original "Database Sharding - Complete Deep Dive" article, with **🔍 Explained** boxes dropped in right after every jargon-heavy part (cardinality, scatter-gather, denormalize, saga pattern, two-phase commit) — sab ek hi jagah, order me.

Prerequisites: `01-Hashing-Basics-Explained.md`, `02-Consistent-Hashing-Deep-Dive.md`
Used in: Key-Value Store, Chat System, URL Shortener, any system beyond single-DB scale

---

## What is Sharding?

Sharding splits a large database into smaller pieces (shards), each stored on a different server. Each shard holds a subset of the data.

**Real-world analogy:** A library with 1 million books. Instead of one giant room (single DB), you build 10 rooms (shards), each holding 100K books organized by author's last name initial (A-C in Room 1, D-F in Room 2, etc).

```
Without sharding:
  1 billion rows → 1 database server → CPU maxed, disk full, queries slow

With sharding (4 shards):
  Shard 1: rows where userId 0-249M      (Server 1)
  Shard 2: rows where userId 250M-499M   (Server 2)
  Shard 3: rows where userId 500M-749M   (Server 3)
  Shard 4: rows where userId 750M-1B     (Server 4)
```

---

## When Do You Need Sharding?

**NOT yet:** If your database is < 1TB and < 10K queries/sec, a single server with read replicas is enough. Don't shard prematurely.

Time to shard when:
- Single DB exceeds disk capacity (> 1-5 TB)
- Write throughput exceeds single server (> 10-50K writes/sec)
- Read replicas aren't enough (write-heavy workload)
- Query latency grows despite indexes

**Rule of thumb:** Scale reads with replicas first. Shard only when writes are the bottleneck.

---

## Sharding Strategies

### 1. Hash-Based Sharding

Hash the shard key and mod by number of shards.

```
shard_id = hash(userId) % num_shards

hash("user_001") % 4 = 2 → Shard 2
hash("user_002") % 4 = 0 → Shard 0
hash("user_003") % 4 = 3 → Shard 3
```

**Pros:** Even distribution (if hash function is good). No hotspots from sequential keys.
**Cons:** Range queries impossible (can't ask "all users with ID 100-200" — they're scattered). Adding shards requires rehashing (use consistent hashing to fix this).

**Best for:** Key-value lookups, user data, sessions.

> ### 🔍 Explained: "Hash the shard key and mod by number of shards" — full breakdown
> **Shard key** = wo column jise hash karke decide karte ho row kaunse shard pe jayega (yaha `userId`).
> ```
> Step 1: shard_key value lo         → userId = "user_101"
> Step 2: use hash karo                → hash("user_101") = 8234759102 (koi bhi bada number)
> Step 3: num_shards se mod karo       → 8234759102 % 4 = 2
> Step 4: result = shard index         → Shard 2
> ```
> **Hash kyu, direct `userId % 4` kyu nahi?** Kyunki shard key aksar string hoti hai (email, UUID) — strings pe `%` seedha kaam nahi karta, pehle hash karke number banate hai. Aur agar IDs sequential/non-uniform pattern me ho (jaise saare naye users consecutively aa rahe ho), hashing unhe achhe se scramble kar deta hai taaki distribution even rahe.
> **Poora detail + numeric worked example:** `01-Hashing-Basics-Explained.md` dekho.

### 2. Range-Based Sharding

Assign contiguous ranges to each shard.

```
Shard 0: userId 0 - 999,999
Shard 1: userId 1,000,000 - 1,999,999
Shard 2: userId 2,000,000 - 2,999,999
```

**Pros:** Range queries work naturally ("get all users from 1M to 1.5M"). Simple to understand.
**Cons:** Hotspots if data isn't evenly distributed (new users all hit the last shard). Uneven shard sizes over time.

**Best for:** Time-series data (shard by month), geographic data (shard by region).

### 3. Directory-Based Sharding

A lookup table maps each key to its shard.

```
Lookup table:
  user_001 → Shard 2
  user_002 → Shard 0
  user_003 → Shard 1

Query: look up shard in directory, then query that shard.
```

**Pros:** Flexible — can move any key to any shard. Rebalancing is easy (update directory).
**Cons:** Directory is a single point of failure. Extra hop for every query.

**Best for:** When you need fine-grained control over data placement.

### 4. Geographic Sharding

Shard by user's geographic region.

```
Shard "US": all US users (servers in us-east-1)
Shard "EU": all EU users (servers in eu-west-1)
Shard "APAC": all Asia-Pacific users (servers in ap-south-1)
```

**Pros:** Data locality (low latency for users near their shard). Compliance (EU data stays in EU for GDPR).
**Cons:** Uneven distribution (US shard might be 5x larger). Cross-region queries are expensive.

---

## Choosing a Shard Key

The shard key determines which shard a row goes to. It's the most important decision in sharding.

**Good shard key properties:**
- High cardinality (many unique values — userId is good, country is bad)
- Even distribution (no single value dominates)
- Matches query patterns (queries include the shard key → single-shard lookup)

| Shard Key | Good For | Bad For |
|---|---|---|
| userId | User-centric apps (get all data for one user) | Cross-user queries |
| orderId | Order lookups | "All orders for user X" (need to scan all shards) |
| timestamp | Time-series | Hot partition (all writes hit current time shard) |
| country | Geo-partitioning | Uneven (US has 60% of traffic) |
| hash(userId) | Even distribution | Range queries, debugging |

> ### 🔍 Explained: "High Cardinality" ka matlab kya hai?
> **Cardinality** = kitne **unique/distinct values** ek column me ho sakte hai.
> ```
> Column: country → ~195 possible values → LOW cardinality (bahut reuse hota hai)
> Column: userId  → millions of possible values → HIGH cardinality
> ```
> Agar `country` ko shard key banao (10 shards ke saath), aur 60% users sirf "India"+"USA" se hai — 2 shards overloaded ho jayenge, baaki 8 mostly empty. High-cardinality column (`userId`) se har unique value evenly spread ho sakta hai, especially hashing ke saath.
>
> **Frontend analogy:** `groupBy(user => user.country)` → sirf ~195 groups (kuch bahut bade) = low cardinality. `groupBy(user => user.id)` → har group me 1 item = high cardinality (but itna extreme bhi sharding ke liye useless hai — isliye `hash(userId) % N` use karte hai, taaki multiple users evenly ek shard me group ho, na ki 1-1 karke bikhar jaye).

---

## Hot Partition Problem

**The problem:** One shard gets disproportionately more traffic than others.

**Examples:**
- Celebrity user (10M followers → their shard gets hammered on every post)
- Time-based shard key (all current writes go to "today" shard)
- Popular product (flash sale → one product's shard overwhelmed)

**Solutions:**

| Solution | How |
|---|---|
| Add salt/suffix to key | `shard_key = userId + "_" + random(0,9)` → spreads across 10 sub-shards. Reads must fan-out. |
| Dedicated shard for hot keys | Detect hot keys → move to a dedicated high-capacity shard |
| Caching in front | Cache hot data aggressively → most reads don't hit the shard (see `08-Redis-Caching-Explained.md`) |
| Further split the hot shard | Break one shard into 4 smaller ones |

---

## Cross-Shard Queries (The Pain)

The biggest downside of sharding: Queries that span multiple shards are expensive.

```
"Get the top 10 orders across all users sorted by amount"
→ Query ALL shards → each returns its top 10 → merge → pick global top 10
→ Scatter-gather pattern (slow, expensive)
```

> ### 🔍 Explained: "Scatter-Gather" pattern kya hai?
> ```
> Step 1 (SCATTER): Query bhejo Shard1, Shard2, Shard3, Shard4 — sabko PARALLEL me
>   "give me YOUR top 10 orders"
> Step 2: Har shard apna local top 10 return karta hai
>   Shard1 → [A:$500, B:$450, ...]   Shard2 → [C:$600, D:$300, ...]
>   Shard3 → [E:$200, ...]           Shard4 → [F:$700, ...]
> Step 3 (GATHER): Application layer pe saare results combine + re-sort karo,
>   global top 10 nikaalo → [F:$700, C:$600, A:$500, B:$450, ...]
> ```
> **Frontend analogy:** Bilkul `Promise.all()` se 4 alag APIs parallel call karke, phir `.flat().sort()` se merge karne jaisa hai.
> **Problem:** Response time = **sabse slowest shard jitna time le**, plus merge overhead.

**How to handle:**

| Pattern | When | Trade-off |
|---|---|---|
| Denormalize | Store duplicate data to avoid cross-shard JOINs | Write amplification |
| Application-side join | Query both shards, merge in app code | Complex, higher latency |
| Scatter-gather | Fan out query to all shards, aggregate results | Latency = slowest shard |
| Secondary index (Elasticsearch) | Sync data to a search index that isn't sharded the same way | Extra infra, lag |

> ### 🔍 Explained: "Denormalize" + "Write amplification"
> Normal design: `orders` table has `userId` → JOIN with `users` table to get the name. Fine, **as long as both tables are on the same shard**. If `orders` is sharded by `orderId` and `users` is elsewhere — JOIN becomes cross-shard (expensive/impossible directly).
> ```
> Denormalized orders table:
> ┌────────┬─────────┬───────────┐
> │orderId │ userId  │ userName  │  ← "Bob" copied here directly, no JOIN needed
> ├────────┼─────────┼───────────┤
> │ 1      │ 101     │ Bob       │
> └────────┴─────────┴───────────┘
> ```
> **Write amplification:** if Bob renames himself to "Robert", you now have to update **every copy** of "Bob" (potentially thousands of order rows) — one logical update became many physical writes.
> **Frontend analogy:** Copying `user.name` into multiple components' local state instead of one Redux source of truth — fast to read, but now you must manually keep every copy in sync.

**Best practice in interviews:** "I'd shard by userId so all of a user's data is on one shard. For cross-user queries (leaderboards, analytics), I'd use a separate denormalized read store or Elasticsearch."

---

## Rebalancing

When shards become uneven (one grows too large), you need to move data between shards.

**Approaches:**

| Strategy | How | Downtime? |
|---|---|---|
| Fixed partitions | Pre-create many partitions (e.g., 1000), assign groups to nodes. Rebalance = move partition groups. | Minimal |
| Dynamic splitting | When a shard exceeds size threshold, split into two. | Zero (if background) |
| Consistent hashing | Add virtual nodes for new server, only ~K/N keys move. | Zero |

- **DynamoDB:** Auto-splits partitions when they exceed 10GB or 3000 RCU/1000 WCU. You don't manage this manually.
- **Cassandra:** Uses consistent hashing with virtual nodes. Adding a node = automatic rebalancing.

*(Full consistent hashing mechanics — ring, virtual nodes, worked example — in `02-Consistent-Hashing-Deep-Dive.md`.)*

---

## Sharding vs Replication

| | Sharding | Replication |
|---|---|---|
| Purpose | Scale writes + storage | Scale reads + availability |
| Data | Each shard has DIFFERENT data | Each replica has SAME data |
| Failure | Shard down = that data unavailable | Replica down = other replicas serve |
| Complexity | High (routing, cross-shard queries) | Medium (replication lag, failover) |
| When | Write-heavy, large data | Read-heavy, high availability |

**You usually need BOTH:** Shard for writes/storage, replicate each shard for reads/HA.

```
Shard 1: Primary → Replica 1A, Replica 1B
Shard 2: Primary → Replica 2A, Replica 2B
Shard 3: Primary → Replica 3A, Replica 3B
```

---

## Common Interview Questions

**Q: "How would you shard this database?"**
A: "I'd shard by [entity]Id using hash-based sharding for even distribution. All data for one [entity] lives on one shard, so most queries are single-shard. For cross-entity queries, I'd use a secondary index (Elasticsearch) synced via CDC."

**Q: "What's the risk of sharding by timestamp?"**
A: "Hot partition. All current writes go to the 'now' shard. Fix: compound key = timestamp + random suffix, or hash-based sharding with separate time-series index for time queries."

**Q: "When would you NOT shard?"**
A: "When your data fits on one server (< 1TB), when writes are < 10K/sec, or when your queries frequently need cross-entity JOINs (sharding makes JOINs very expensive)."

**Q: "How do you handle transactions across shards?"**
A: "You can't do ACID transactions across shards easily. Options: 1) Design so transactions are single-shard (shard by the transactional entity). 2) Use saga pattern for cross-shard operations. 3) Use two-phase commit (slow, avoid)."

> ### 🔍 Explained: "Saga Pattern" vs "Two-Phase Commit" — what's actually happening
> Single-DB transaction = "all or nothing" easily (`BEGIN...COMMIT`). Across 2 shards, this gets hard.
>
> **Two-Phase Commit (2PC):** coordinator asks both shards "ready to commit?", both lock and say yes, then coordinator says "commit now."
> ```
> Coordinator → Shard1: "Ready?" → Shard1: "Yes" (locks row)
> Coordinator → Shard2: "Ready?" → Shard2: "Yes" (locks row)
> Coordinator → both: "COMMIT NOW"
> ```
> Problem: both shards hold **locks** until the coordinator confirms — if the coordinator crashes mid-way, shards stay locked. Slow, fragile.
>
> **Saga Pattern:** break the transaction into small independent steps, each with a defined "undo/compensating action":
> ```
> Step 1: Deduct $100 from Account A → succeeds
> Step 2: Add $100 to Account B      → FAILS
> Compensation: undo Step 1 → refund $100 back to Account A
> ```
> This is "eventually consistent" — briefly inconsistent mid-flight, but every failure has a defined rollback path. More scalable than 2PC (no long locks), but more application-code complexity (you write the undo logic yourself).
>
> **Frontend analogy:** Multi-step form/checkout — if payment (step 3) fails, you programmatically release the reserved inventory (undo step 1-2), instead of trying one giant atomic transaction across services.

---

## Recap Table — all the "assumed knowledge" terms in this article

| Term | Simple meaning |
|---|---|
| Cardinality | Kitne unique values ek column me ho sakte hai |
| Scatter-gather | Sabhi shards ko parallel query bhejo, results merge karo |
| Denormalize | Duplicate data store karo taaki cross-shard JOIN avoid ho |
| Write amplification | Ek logical update, multiple physical writes me badal jata hai |
| Two-Phase Commit | Coordinator dono shards se confirm leta hai before committing (safe but slow) |
| Saga Pattern | Transaction ko chhote steps + rollback/compensating actions me todna |
| Hot partition | Ek shard pe disproportionate traffic (uneven load) |

---

## Quick Self-Check
1. `shard_id = hash(userId) % num_shards` — har step ka kaam kya hai?
2. High cardinality shard key ke liye kyu zaroori hai?
3. Scatter-gather pattern me response time kis cheez pe depend karta hai?
4. Denormalization se konsa problem solve hota hai, aur uska trade-off kya hai?
5. Two-Phase Commit slow kyu hota hai, aur Saga pattern isse kaise different hai?
6. Sharding aur Replication me core difference kya hai (data ke level pe)?