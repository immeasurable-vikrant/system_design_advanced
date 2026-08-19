# 09 — Isolation Levels, Anomalies, MVCC & Locking

> **Is lesson me:** Concurrency ke saare classic bugs (dirty read, non-repeatable read, phantom, lost update, write skew), 4 isolation levels jo unhe control karte hai, MVCC (kaise modern DBs read locks avoid karte hai), aur practical techniques (atomic updates, optimistic/pessimistic locks) taaki tum double-spend/oversell rok sako. **Ye series ka sabse high-value lesson hai** — interviews me yahi differentiate karta hai.

Prerequisites: `08-Transactions-ACID.md`

---

## 1. Setup: kyu problems aate hai

Jab **kayi transactions ek saath** (concurrently) chalte hai, aur wo same data touch karte hai — subtle bugs aate hai. **Isolation levels** define karte hai kaunse bugs allow hai aur kaunse prevented.

Strategy: **pehle anomalies (bugs) seekho** — phir levels khud-ba-khud define ho jaate hai ("ye level ye bug rokta hai").

Notation: T1, T2 = do concurrent transactions.

---

## 2. The anomalies (severity order me)

### (a) Dirty Read — uncommitted data padhna
```
T1: UPDATE accounts SET balance = 0 WHERE id='A';   -- committed NAHI
T2: SELECT balance FROM accounts WHERE id='A';      -- reads 0  ← DIRTY
T1: ROLLBACK;                                       -- wo 0 kabhi tha hi nahi!
```
T2 ne aisa data padha jo kabhi exist nahi kiya. **Almost har DB by default ise rokta hai.**

### (b) Dirty Write — uncommitted data overwrite karna
Do transactions same row likhe jab doosra in-flight hai, aur haarne wali write gayab. **Row-level write locks se everywhere prevented.**

### (c) Non-Repeatable Read (read skew) — same row, alag values
```
T1: SELECT balance FROM accounts WHERE id='A';   -- 500
T2: UPDATE accounts SET balance=100 WHERE id='A'; COMMIT;
T1: SELECT balance FROM accounts WHERE id='A';   -- 100  ← beech me badal gaya!
```
Jo cheez ek row do baar padhti hai (ya consistent snapshot chahiye) wo toot jaati hai. E.g., ek backup jo A ko transfer-se-pehle aur B ko transfer-ke-baad capture kare → aisa paisa dikhata hai jo exist nahi karta.

### (d) Phantom Read — matching rows ka SET badal jaata hai
```
T1: SELECT COUNT(*) FROM bookings WHERE room=5 AND date='2026-08-01';  -- 0
T2: INSERT INTO bookings VALUES (room 5, '2026-08-01'); COMMIT;
T1: SELECT COUNT(*) ...;                                              -- 1  ← phantom!
```
T1 ne jo rows padhi thi wo nahi badli — **nayi rows** uske predicate se match kar gayi. Isliye row locks kaafi nahi — **predicate/range locks** chahiye.

### (e) Lost Update — read-modify-write race
```
Dono read counter = 10
T1: writes 11
T2: writes 11    ← 12 hona chahiye tha; ek increment gayab!
```
Fix: atomic write (`SET n = n + 1`), explicit lock (`SELECT FOR UPDATE`), ya version-based compare-and-set.

### (f) Write Skew — the subtle one (interviewers love this)
```
Rule: kam se kam 1 doctor hamesha on-call ho.
Abhi: Alice aur Bob dono on-call hai.

T1 (Alice): SELECT COUNT(*) FROM oncall WHERE shift='night';  -- 2, theek, main ja sakti hu
T2 (Bob):   SELECT COUNT(*) FROM oncall WHERE shift='night';  -- 2, theek, main ja sakta hu
T1: UPDATE oncall SET on_call=false WHERE dr='Alice'; COMMIT;
T2: UPDATE oncall SET on_call=false WHERE dr='Bob';   COMMIT;

→ ZERO doctors on call! Dono transactions individually valid the.
```
Dono ne **same set padha, alag rows likhe** — koi write-write conflict detect nahi hua, but shared invariant toot gaya. Ye snapshot isolation me bhi survive karta hai. **Sirf SERIALIZABLE (ya manual lock/materialized conflict row) rokta hai.**

**Write skew hi hai** double-spending wallet, overselling last ticket, aur duplicate-username races ke peeche ka mechanism. **Isko achhe se samjho — ye sabse important hai.**

---

## 3. The 4 isolation levels

| Level | Dirty read | Non-repeatable | Phantom | Lost update | Write skew |
|---|---|---|---|---|---|
| **Read Uncommitted** | ✅ possible | ✅ | ✅ | ✅ | ✅ |
| **Read Committed** | ❌ prevented | ✅ | ✅ | ✅ | ✅ |
| **Repeatable Read** | ❌ | ❌ | ✅* | ❌* | ✅ |
| **Serializable** | ❌ | ❌ | ❌ | ❌ | ❌ |

`*` = implementation-dependent (vendor table neeche). Higher level = zyada safe, but zyada cost (locks/aborts).

---

## 4. Defaults actually kya hai (the trap interviewers set)

| Database | Default | "Repeatable Read" wahan kya matlab |
|---|---|---|
| **PostgreSQL** | **Read Committed** | Snapshot isolation; phantoms bhi rokta hai (standard require nahi karta). Still allows write skew. |
| **MySQL/InnoDB** | **Repeatable Read** | Snapshot + gap locks → phantoms locking reads ke liye prevented |
| Oracle | Read Committed | True Repeatable Read nahi; "SERIALIZABLE" actually snapshot isolation |
| SQL Server | Read Committed | Lock-based default; `READ_COMMITTED_SNAPSHOT` → MVCC |
| SQLite | Serializable | Single writer, isliye nearly free |
| CockroachDB | Serializable | Only level |

> ⚠️ **Trap:** "Postgres defaults to Repeatable Read" is **WRONG** — it's **Read Committed**. Aur "Repeatable Read prevents phantoms" Postgres/MySQL me incidentally true hai but SQL standard require nahi karta. **Hamesha bolo kaunsa database.**

**Interview-ready line:** *"Postgres defaults to Read Committed — matlab har statement ek fresh snapshot dekhta hai, so application-code me read-modify-write ek update lose kar sakta hai. Balance check ke liye main `SELECT ... FOR UPDATE`, ya atomic `balance = balance - ?` with `CHECK (balance >= 0)`, ya SERIALIZABLE + retry use karunga."*

---

## 5. MVCC — kaise modern DBs read locks avoid karte hai

**MVCC (Multi-Version Concurrency Control):** writers **new row versions** banate hai overwrite karne ke bajaye — isliye **readers kabhi writers ko block nahi karte, aur writers readers ko nahi.**

```
Row 'A' ki physical versions (Postgres heap me):
 version | balance | created_by_txn | deleted_by_txn
---------|---------|----------------|----------------
   v1    |   500   |      100       |      105
   v2    |   100   |      105       |      NULL      ← current

Transaction 103 (snapshot txn 105 se pehle ka) reads v1 → 500
Transaction 110 reads v2 → 100
Dono me se koi read lock nahi leta, koi wait nahi karta.
```

**Consequences (interview me name karo):**
- **Readers don't block writers** — MVCC ka main win, read-heavy workloads pe excellent.
- **Old versions accumulate** → Postgres ko **VACUUM** chahiye purane versions clean karne ke liye; bina iske "table bloat".
- **Long-running transactions dangerous** — wo purane versions ko pin kar dete hai (vacuum horizon roK dete hai), ek forgotten analytics query millions dead rows pin kar sakti hai.
- **XID wraparound** — Postgres transaction IDs 32-bit hai; agar vacuum bahut peeche gir jaye, DB writes accept karna band kar deta hai (famous outage cause).

**MVCC vs locking:**
| | MVCC (Postgres, InnoDB, Oracle) | Two-Phase Locking (old SQL Server) |
|---|---|---|
| Reads | Never block | Shared locks lete hai, writers block |
| Writes | Sirf same-row writers block | Readers ko bhi block |
| Read-heavy | Excellent | Poor |
| Cost | Version storage + vacuum | Lock contention, deadlocks |

---

## 6. Serializable ko achieve karne ke 2 tarike

**(a) Actual serial execution** — transactions ek-ek karke single thread pe (Redis effectively). Perfect isolation, zero concurrency — sirf tab viable jab har transaction chhota + in-memory ho.

**(b) Two-Phase Locking (2PL)** — locks lete jao, sirf commit pe chhodo. Correct but slow: bahut blocking, deadlocks, phantoms ke liye predicate locks.

**(c) Serializable Snapshot Isolation (SSI) — jo Postgres karta hai** — optimistic: snapshot isolation pe chalao, read/write dependencies track karo, aur commit pe **abort** kar do agar interleaving serial nahi ho sakti thi.
```
Conflicts rare → near-snapshot performance (great)
Contention high → high abort rate, aur app ko RETRY karna PADEGA
```

> ⚠️ **Agar interview me "SERIALIZABLE" bolo, turant bolo "and the application retries on serialization failure (Postgres SQLSTATE 40001)."** Retry loop bhoolna #1 real-world SSI bug hai.
```java
for (int attempt = 0; attempt < 3; attempt++) {
    try {
        tx.begin();   // SERIALIZABLE
        transfer(from, to, amount);
        tx.commit();
        return;
    } catch (SerializationFailureException e) {   // 40001
        tx.rollback();
        sleep(jitteredBackoff(attempt));
    }
}
throw new TooMuchContentionException();
```

---

## 7. Lost update & write skew ko practically rokna (4 techniques)

Priority order me (jitni baar use karoge):

### (1) Atomic write — best jab fit ho
```sql
UPDATE accounts SET balance = balance - 500
WHERE id = 'A' AND balance >= 500;
-- 0 rows updated → insufficient funds. Koi race possible nahi.
```
Add `CHECK (balance >= 0)` — invariant database enforce karta hai, "hope" nahi. **Double-spend structurally impossible.**

### (2) Optimistic concurrency (version compare-and-set)
```sql
UPDATE docs SET content = ?, version = version + 1
WHERE id = ? AND version = ?;   -- 0 rows → koi aur jeet gaya, reload & retry
```
User-facing edits ke liye great (locks user ke "think time" me hold nahi hote). Tum "ye changed, reload?" dikha sakte ho.

### (3) Pessimistic lock (explicit)
```sql
BEGIN;
SELECT * FROM seats WHERE id = 42 FOR UPDATE;   -- dusre yaha block honge
-- decide karo, phir likho
COMMIT;
```
High contention (last seat, hot inventory) ke liye. **Warning:** `FOR UPDATE` ko network call ke across hold karna = deadlock factory. `FOR UPDATE SKIP LOCKED` = Postgres pe job queue banane ka standard tarika.

### (4) Materialize the conflict
Write skew isliye hota hai kyunki koi shared row nahi jispe conflict ho. Toh ek banao — ek `room_bookings_lock` row per (room, date) jise dono transactions update kare. Ab DB write-write conflict dekhta hai, ek abort hota hai. Ugly, but invisible anomaly ko detectable bana deta hai.

---

## 8. Deadlocks (practical reality)

**Deadlock** = T1 ne row X lock ki, T2 ne row Y, ab T1 ko Y chahiye aur T2 ko X — dono ek-dusre ka wait, forever.

Database ise **detect** karta hai aur ek "victim" ko abort kar deta hai (app ko retry karna padta hai). Consistently deadlock hona usually matlab transactions **inconsistent order** me locks lete hai.

**Fixes:**
- Locks hamesha same order me lo (e.g., hamesha lower account-ID pehle lock karo)
- Transactions short rakho
- Lock ke across kabhi network call mat karo

---

## 9. Decision framework — kab kya use karo

| Situation | Use |
|---|---|
| Counter / balance decrement | Atomic conditional `UPDATE` + `CHECK` constraint |
| User document edit | Optimistic version check, conflict UI |
| Last seat / high contention on one row | `SELECT ... FOR UPDATE` (short txn) |
| Job queue on Postgres | `FOR UPDATE SKIP LOCKED` |
| Complex invariant across multiple rows | `SERIALIZABLE` + retry loop |
| Long analytics query on hot table | A **replica** — never a long txn on primary |

---

## Quick Self-Check
1. 6 anomalies list karo (dirty read se write skew tak), ek line each.
2. Write skew kya hai — doctor example se, aur ise kaun sa level rokta hai?
3. Postgres ka default isolation level kya hai (trap!)? Wo kaunsa anomaly allow karta hai?
4. MVCC kya hai, aur uska main win + main cost (VACUUM) kya hai?
5. SSI me interview me "SERIALIZABLE" bolne ke baad turant kya bolna zaroori hai?
6. Double-spend ko structurally impossible kaise banate ho (atomic update)?
7. Optimistic vs pessimistic locking — kab kaunsa?
8. Deadlock kya hai aur 2 fixes?

---
**Next:** `10-WAL-Durability.md` — durability ka engine (kaise commits crash survive karte hai).