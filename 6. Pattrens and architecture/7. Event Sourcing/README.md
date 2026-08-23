# 07 — Event Sourcing

> **Is lesson me:** Event sourcing pattern — current state store karne ke bajaye **saare events** (state changes) ko store karna, aur state ko replay se derive karna. Benefits (audit, time-travel, replay), challenges, aur CQRS ke saath natural pairing. Ye Topic 2 (WAL) + Topic 4 (event streams) ko ek storage philosophy me le jaata hai.

Prerequisites: `06-CQRS.md`, Topic 2 lesson 10 (WAL)

---

## 1. Event sourcing kya hai

**Event sourcing** = current state ko directly store karne ke bajaye, tum **saare events** (jo changes hue) ko ek append-only log me store karte ho. Current state in events ko **replay** karke derive hota hai.

```
TRADITIONAL (current state store):
  Account balance = ₹500   (bas current value, history gone)
  Update → balance = ₹300  (purani value overwrite, ₹500 ka record nahi bacha)

EVENT SOURCING (events store):
  Event log (append-only):
    1. AccountOpened(balance=0)
    2. Deposited(₹1000)     → balance 1000
    3. Withdrew(₹500)       → balance 500
    4. Withdrew(₹200)       → balance 300
  Current state = replay all events = ₹300
  (poori history preserved, kabhi kuch delete/overwrite nahi hota)
```

**Key idea:** events **immutable facts** hai (ho chuka, badal nahi sakta). State ek **derived value** hai (events ka fold/reduce).

> **Recap (Topic 2 lesson 10 — WAL):** yaad hai WAL har change ko append-only log me likhta hai, aur crash pe replay karke state rebuild karta hai? Event sourcing wahi philosophy ko **application data model** bana deta hai — events source of truth hai, state derived. Database WAL internal tha; event sourcing use tumhara primary model banata hai.

**Frontend analogy:** ye Redux jaisa hai! Redux me state = reducer(actions) — tum current state directly store nahi karte, tum **actions** (events) dispatch karte ho aur state derive hota hai. Redux DevTools me time-travel debugging — wahi event sourcing ka replay hai. Git bhi event-sourced hai (commits = events, current code = replay of commits).

---

## 2. State kaise reconstruct hota hai

```
Current state chahiye → saare events replay karo (fold):
  state = events.reduce((s, event) => apply(s, event), initialState)

Account example:
  0 → Deposited(1000) → 1000 → Withdrew(500) → 500 → Withdrew(200) → 300
```

**Snapshots (performance optimization):** millions of events replay karna slow hai. Toh periodically ek **snapshot** (current state ka photo) save karo, aur sirf snapshot ke baad wale events replay karo:
```
Snapshot at event 10000: balance=₹5000
Naye events: 10001, 10002, 10003
Current state = snapshot(5000) + replay(10001..10003)  (sirf 3 events, not 10003)
```
> **Recap (Topic 2 lesson 10):** ye WAL ke **checkpoints** jaisa hai — periodically state flush karke, sirf checkpoint ke baad ka log replay karo. Same optimization.

---

## 3. Benefits ⭐

- **Complete audit trail** — har change ek immutable event. "Kisne kya kab kiya" — poora history, free. (Compliance, finance ke liye huge.)
- **Time-travel** — kisi bhi past point ka state reconstruct karo (events us point tak replay). "3 mahine pehle account kaisa tha?"
- **Debugging** — exact sequence of events se bug reproduce karo.
- **Replay / rebuild** — naya read model banana ho (CQRS lesson 06)? Bas events replay karke populate karo. Bug fix karke state recompute karo.
- **Event-driven natural fit** — events already hai, publish kar do (EDA — lesson 05).

**Killer use case — finance/banking:** har transaction ek event. Balance derived. Poora audit trail regulatory requirement ke liye built-in. Dispute? Events replay karo exact history dekhne ko.

---

## 4. Challenges ⚠️

- **Query complexity** — "current balance?" ke liye events replay/snapshot chahiye (isliye CQRS ke saath — read model current state rakhe).
- **Event schema evolution** — purane events ka format badle toh? Events immutable hai, toh versioning + upcasting chahiye (purane events ko naye format me translate).
- **Eventual consistency** — read models async update (Topic 5).
- **Storage growth** — events kabhi delete nahi hote, log badhta rehta hai (snapshots + archival se manage).
- **Learning curve** — mindset shift (state → events). Debugging different.
- **Deletes mushkil** — "delete" bhi ek event hai (immutability). GDPR "right to be forgotten" ke saath tension (crypto-shredding se handle).

---

## 5. Event Sourcing + CQRS = natural pair ⭐

Ye dono aksar saath aate hai:
```
Write side (Event Sourcing):
  Commands → events generate → append to event store (source of truth)

Read side (CQRS read models):
  Events consume → denormalized read models banao (current state, fast queries)

  Event Store (append-only)
       │ events
       ├──▶ Read Model: current balances (Redis — instant lookup)
       ├──▶ Read Model: transaction history (SQL — Topic 2)
       └──▶ Read Model: analytics (warehouse)
```
Event sourcing "write side" ko append-only events banata hai; CQRS "read side" un events se query-optimized views banata hai. **Perfect fit** — event sourcing raw truth, CQRS usable views.

---

## 6. Kab use karo

**✅ Use:** audit trail critical (finance, healthcare, legal), temporal queries chahiye ("us waqt kya tha"), complex domains with rich history, event-driven system already, debugging/replay valuable.
**❌ Avoid:** simple CRUD (massive over-engineering), audit/history ki zarurat nahi, team ready nahi (steep learning curve), current-state queries dominate aur history irrelevant.

**Interview line:** *"Event sourcing tab jab audit trail ya temporal queries first-class requirement ho — jaise banking (har transaction ek immutable event, balance derived). Main isse CQRS ke saath pair karunga — events source of truth, denormalized read models fast queries ke liye, snapshots replay optimize karne ke liye. But ye significant complexity hai — simple CRUD ke liye bilkul nahi."*

---

## Quick Self-Check
1. Event sourcing kya hai — state store vs events store (bank balance example)?
2. Events immutable facts, state derived — ye kaise reconstruct hota hai (replay/fold)?
3. Ye Topic 2 ke WAL se kaise relate karta hai (append-only, replay)?
4. Redux/Git analogy event sourcing ko kaise dikhati hai?
5. Snapshots kyu chahiye, aur ye WAL ke kis concept jaisa (checkpoints)?
6. Event sourcing ke 3 killer benefits (audit, time-travel, replay)?
7. Ye CQRS ke saath natural pair kyu hai?
8. Kab use karo, kab over-engineering?

---
**Next:** `08-Saga-Pattern.md` — microservices me distributed transactions.