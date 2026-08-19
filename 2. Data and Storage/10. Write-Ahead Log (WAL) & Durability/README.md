# 10 — Write-Ahead Log (WAL) & Durability

> **Is lesson me:** WAL kya hai aur kaise ye "committed = crash-proof" guarantee deta hai, fsync ka role, checkpoints, crash recovery process, aur kaise wahi WAL replication (lesson 12) aur LSM-trees/Kafka ko power karta hai. Ye durability (ACID ka "D") ka actual engine hai.

Prerequisites: `08-Transactions-ACID.md`, `06-Indexing-Deep-Dive.md` (disk concepts)

---

## 1. Core problem: memory volatile hai

Database performance ke liye data ko **memory (buffer pool/RAM)** me rakhta hai. But RAM volatile hai — crash pe uncommitted in-memory data gayab.

Sawaal: agar database ne client ko bola "tumhara write ho gaya" (acknowledged), aur turant crash ho jaye — kya wo write bacha? **Durability guarantee: haan, bachna CHAHIYE.** WAL yehi ensure karta hai.

---

## 2. WAL kya hai

**Write-Ahead Log (WAL)** = ek **sequential, append-only file** jaha har modification **PEHLE** likhi jaati hai, main data structure me apply karne se pehle. Crash pe, WAL ek complete record hai jise **replay** karke correct state restore kar sakte hai.

**Analogy:** Ek chef restaurant me. Cooking se pehle, order ko ek paper ticket (WAL) pe likh ke board pe pin karta hai. Agar beech me interrupt ho (crash), wo tickets dekh ke pata karta hai kya order tha aur kya banana baaki hai. **Ticket source of truth hai, aadha-paka khaana nahi.**

---

## 3. WAL kaise kaam karta hai — the sequence

```
Client: "INSERT INTO orders VALUES(...)"

Step 1: Operation ko WAL file me APPEND karo
        [LOG] "insert order 123: ..."
Step 2: fsync() — ensure WAL record physically disk pe pahunch gaya
Step 3: Client ko ACK bhejo (durability ab guaranteed!) ✅
Step 4: (baad me) change ko in-memory structures me apply karo
Step 5: (periodically) in-memory data ko main data files me flush karo (CHECKPOINT)
Step 6: Checkpoint ke pehle wale WAL entries safely delete ho sakte hai
```

**The guarantee:** Agar system ne write **acknowledge** kiya, wo write **kisi bhi crash ko survive karega** — kyunki wo ACK bhejne se **pehle** hi WAL me disk pe likha ja chuka tha.

> **Kyu step 1 (WAL append) fast hai, but direct data-file update slow?**
> WAL **sequential append** hai (file ke end me likhna) — disk ke liye ye super fast hai (no seeking around). Actual data files me row update karna = **random I/O** (disk pe specific jagah dhundh ke likhna) — 100-1000x slower on HDD. Isliye WAL trick: pehle fast sequential log likho (durability secure), phir aaram se background me actual data files update karo. (Ye lesson 06 ka "disk reads/random I/O" concept ka extension hai.)

---

## 4. fsync — the critical piece

**`fsync()`** OS ko force karta hai ki uska write-buffer **physically disk** pe flush ho. Iske bina, WAL write OS ke page cache (RAM) me baith sakta hai aur crash pe lose ho sakta hai.

Critical sequence: **`write()` to WAL → `fsync()` → THEN ACK to client.**

Kuch systems ek "relaxed durability" mode dete hai jo fsync skip karta hai higher throughput ke liye — but tab crash pe last few milliseconds ke writes lose ho sakte hai. **Yahi hai wo `synchronous_commit` knob jo lesson 08 me mention kiya tha** — speed vs safety.

**Group commit optimization:** Har single write pe fsync karne ke bajaye, kayi writes batch karke ek fsync karo. Thodi latency (few ms) trade karke dramatically zyada throughput.

---

## 5. Checkpoints & crash recovery

**Checkpoint** = periodically in-memory dirty data ko main data files me flush karna, aur WAL me ek "checkpoint marker" lagana. Iske pehle wali WAL entries phir delete ho sakti hai (data files me already safe hai).

**Crash recovery process:**
```
1. System crash ke baad restart hota hai
2. WAL me last checkpoint marker dhundho
3. Checkpoint ke BAAD wali saari WAL entries REPLAY karo
4. Committed transactions re-apply karo
5. Uncommitted (incomplete) transactions discard karo
6. System ab consistent hai, ready to serve
```

Recovery time checkpoints ke interval pe depend karta hai. Frequent checkpoints = faster recovery, but normal operation me zyada I/O. Trade-off.

**WAL corruption?** WAL entries me CRC checksums hote hai. Recovery pe agar checksum fail ho (mid-write crash), wo entry (aur uske baad wali) discard ho jaati hai — wo transaction client ko acknowledge hi nahi hua tha, so client ke perspective se koi data loss nahi.

---

## 6. WAL har jagah hai (real systems)

### PostgreSQL WAL
16MB WAL segments, data pages modify karne se pehle. Used for: crash recovery, **replication** (WAL replicas ko stream karo — lesson 12), point-in-time recovery (kisi bhi timestamp tak replay).

### MySQL InnoDB Redo Log
Functionally identical WAL — fixed-size circular buffer, buffer pool pages modify karne se pehle redo entries likhta hai.

### Kafka — poora storage model hi WAL hai
Kafka ka har partition ek append-only log on disk hai:
```
Producer → append → [Segment 0: offsets 0-999][Segment 1: 1000-1999]...
Consumers apni position (offset) se padhte hai
```
- Messages sirf append hote hai, kabhi modify nahi
- Har message ka sequential offset
- Followers leader ka log padh ke replicate karte hai

> **Kafka log vs database WAL — key difference:** Database WAL ek **temporary recovery mechanism** hai — checkpoint ke baad entries discard. Kafka ka log **primary data store hai** — messages retention period tak jeete hai, consumers apni pace pe padhte hai. Isliye Kafka ek message broker + durable store dono hai.

### LSM Trees (Cassandra, RocksDB, LevelDB, HBase)
LSM trees WAL ko write path ke first step ki tarah use karte hai:
```
Write → WAL me likho → MemTable (in-memory) me insert
      → MemTable full → disk pe SSTable me flush → WAL segment delete
```

---

## 7. WAL vs other durability approaches

| Approach | Durability | Write perf | Recovery | Use case |
|---|---|---|---|---|
| **WAL + Checkpoint** | Strong | Fast (sequential) | Medium | Postgres, MySQL |
| Write directly to data file | Strong | Slow (random I/O) | Instant | Simple systems |
| In-memory only | None | Fastest | N/A (lost) | Redis without persistence |
| Append-only file (AOF) | Strong | Fast | Slow (replay all) | Redis AOF |

> **Redis connection (lesson 14 preview):** Redis by default in-memory (no durability). But **AOF** (append-only file — WAL jaisa) ya **RDB snapshots** enable karke Redis ko crash-resistant bana sakte ho — tabhi Redis ko "primary database" ki tarah use kar sakte ho, na ki sirf cache.

---

## 8. Performance characteristics (why WAL is fast)

| Aspect | WAL advantage |
|---|---|
| Write pattern | Sequential append (100-1000x faster than random on HDD) |
| Disk I/O | Ek fsync per batch (group commit) |
| Throughput | Multiple writes ko ek WAL flush me batch |

---

## Quick Self-Check
1. WAL kis core problem ko solve karta hai (memory volatile hone ka)?
2. "Committed = crash-proof" guarantee WAL kaise deta hai — sequence kya hai?
3. Sequential WAL append, direct data-file update se fast kyu hai?
4. `fsync()` kya karta hai, aur wo critical path me kaha aata hai?
5. Crash recovery ke steps kya hai (checkpoint marker se)?
6. Kafka ka log aur database WAL me core difference kya hai?
7. Redis ko durable (primary-store-worthy) kaise banate hai?

---
**Next:** `11-Hashing-And-Consistent-Hashing.md` — ab scaling out shuru (distributing data across machines).