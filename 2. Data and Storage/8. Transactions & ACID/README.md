# 08 — Transactions & ACID (Correctness When It Matters)

> **Is lesson me:** Transaction kya hai, ACID ke har letter ka **exact** matlab (aur wo kis cheez se protect karta hai), atomicity vs isolation ka confusion, aur kyu money/inventory/seats wale systems isse zaroori hai. Ye lesson 09 (isolation levels) ki foundation hai.

Prerequisites: `03-SQL-Language-Basics.md`

---

## 1. Transaction kya hai

**Transaction** = multiple reads/writes ko ek single unit me group karna, jo **ya toh poora hota hai, ya bilkul nahi**.

**Classic example — paisa transfer:** A se B ko ₹500 bhejna = 2 writes:
```sql
BEGIN;                                                    -- transaction start
  UPDATE accounts SET balance = balance - 500 WHERE id = 'A';   -- debit A
  UPDATE accounts SET balance = balance + 500 WHERE id = 'B';   -- credit B
COMMIT;   -- dono ek saath visible ho gaye
-- ya ROLLBACK; -- dono me se kuch nahi hua
```

**Bina transaction, kya bura ho sakta hai:** Pehle UPDATE (A se 500 kate) ke baad, doosre UPDATE se pehle **crash** ho jaye — ₹500 gayab! A se kat gaya, B ko mila hi nahi. Transaction ye rokta hai: **dono, ya koi nahi** (atomicity).

- `BEGIN` (ya `BEGIN TRANSACTION`) — transaction shuru
- `COMMIT` — saare changes permanently apply karo (ek saath visible)
- `ROLLBACK` — saare changes cancel karo (jaise kuch hua hi nahi)

**Frontend analogy:** Socho ek multi-step form/checkout — tum chahte ho ya toh poora order place ho (payment + inventory reserve + confirmation), ya kuch bhi na ho. Aadha-adhura (payment ho gaya but order nahi bana) = disaster. Transaction database level pe yehi "all-or-nothing" deta hai.

---

## 2. ACID — precisely (har letter kis cheez se bachata hai)

Sab acronym expand kar lete hai. Asli samajh = **har letter EXACTLY kis problem se protect karta hai:**

| Letter | Guarantee | Kis se bachata hai | Kaise implement |
|---|---|---|---|
| **Atomicity** | All-or-nothing | Crash/abort beech me, aadha kaam apply ho jaana | Undo log / rollback |
| **Consistency** | Invariants hold (before & after) | Constraint violations (negative balance, orphan FK) | Constraints + app logic |
| **Isolation** | Concurrent transactions ek-dusre ko corrupt na kare | Simultaneous transactions ke beech race conditions | Locks ya MVCC |
| **Durability** | Committed = crash ke baad bhi safe | Acknowledge kiya hua commit power-cut me lose hona | WAL + fsync (lesson 10) |

### ⚠️ Sabse bada confusion: Atomicity ≠ Isolation
- **Atomicity** = **crashes aur aborts** ke baare me. "Kya aadha kaam bacha reh sakta hai?" → No.
- **Isolation** = **doosre transactions** ke baare me. "Kya ek aur transaction meri beech me interfere kar sakta hai?" → No.

Log inhe constantly blur karte hai. Yaad rakho: **Atomicity = crash-safety. Isolation = concurrency-safety.**

### Ek aur trap: "C" in ACID ≠ "C" in CAP
ACID ka Consistency = "database invariants/constraints valid rehte hai". CAP ka Consistency (lesson 14) = "har read latest write dekhega" — bilkul alag cheez. Interview me clarify karo kaunsa "C".

---

## 3. Durability ki ek caveat (important nuance)

"Committed" ka matlab technically = "WAL me likha aur fsync ho gaya" (lesson 10 me WAL). But durability ek **spectrum** hai:
- Agar `synchronous_commit = off` ho, ya disk jhooth bole flush ke baare me, ya replication async ho aur primary mar jaye — tum ek acknowledged commit **lose** kar sakte ho.
- Iska "knob" hai `synchronous_commit` (speed vs safety trade-off).

Abhi bas itna: durability absolute nahi, configurable hai. Detail lesson 10 (WAL) me.

---

## 4. Consistency (ACID wala) ko thoda aur samjho

Ye letter thoda philosophical hai. Iska matlab: transaction database ko ek **valid state** se doosre **valid state** me le jaata hai — koi "rule" (constraint) na toote.

Examples of rules (constraints):
```sql
CHECK (balance >= 0)          -- balance kabhi negative na ho
FOREIGN KEY (user_id) ...      -- orphan record na bane
UNIQUE (email)                 -- duplicate email na ho
NOT NULL                       -- required field khali na ho
```
Agar ek transaction in rules ko todne ki koshish kare (jaise balance ko -100 karna), database use **reject** kar deta hai (transaction fail/rollback). Ye "consistency" hai — database khud invariants enforce karta hai, tumhare app logic + declared constraints ke through.

---

## 5. Kyu ye sab itna matter karta hai — real stakes

Transactions "nice to have" nahi hai jab paisa/inventory/seats involved ho:

- **Digital wallet:** ₹500 transfer beech me crash → paisa gayab, ya duplicate. ACID ke bina impossible to guarantee.
- **Ticket booking:** last seat 2 log ek saath book kare → dono ko mil gaya (oversell). Isolation ke bina hoga.
- **Inventory:** flash sale me 1 item, 1000 buyers → oversell. Atomicity + isolation dono chahiye.
- **E-commerce order:** payment liya but order record nahi bana → customer ne paisa diya, saman nahi milega.

Ye saare "correctness under concurrency + crashes" problems hai — exactly jo ACID solve karta hai.

> **Ek preview (lesson 09):** Simple `BEGIN...COMMIT` kaafi nahi hota kayi baar. E.g., "balance check karke phir deduct" — do transactions ek saath check kare, dono ko lage "paisa hai", dono deduct kare → **double-spend**. Isse rokne ke liye tumhe **isolation levels** aur specific techniques (atomic updates, locks) chahiye — jo agla lesson hai.

---

## 6. Transactions kaise likhte hai (practical)

Raw SQL:
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 'A' AND balance >= 500;
  UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;
```

ORM (SQLAlchemy) me:
```python
try:
    with session.begin():           # transaction start
        account_a.balance -= 500
        account_b.balance += 500
    # block khatam hote hi auto-COMMIT
except Exception:
    # koi bhi error → auto-ROLLBACK, dono changes cancel
    ...
```

**Best practices (abhi ke liye):**
- Transactions **short** rakho — jitni jaldi commit/rollback ho utna acha (locks kam der hold honge — lesson 09).
- Transaction ke andar **external network calls mat karo** (jaise ek API call) — wo transaction (aur uske locks/connection) ko lamba block karta hai.
- Money wali cheezon me hamesha transaction + constraints (`CHECK balance >= 0`) use karo — "hope" pe mat chhodo.

---

## Quick Self-Check
1. Transaction kya hai — paisa-transfer example se explain karo.
2. `BEGIN`, `COMMIT`, `ROLLBACK` — teeno kya karte hai?
3. ACID ke har letter ka guarantee aur wo kis se bachata hai?
4. Atomicity aur Isolation me exact difference kya hai (crash vs concurrency)?
5. ACID ka "C" aur CAP ka "C" alag kaise hai?
6. Durability "spectrum" kyu hai, absolute kyu nahi?
7. Transactions ke andar external API call kyu nahi karni chahiye?

---
**Next:** `09-Isolation-Levels-And-Anomalies.md` — concurrency ke saare bugs aur unse bachne ke tarike (the hard, high-value lesson).