# 09 — Zero-Downtime Migrations & Resilience

> **Is lesson me:** Do operational hard problems — zero-downtime schema migrations (expand-migrate-contract), aur runtime resilience (timeouts/retries/circuit breakers/load shedding/DR). Ye "how do you change the DB without downtime?" aur "what happens when a region dies?" ke answers hai.

Prerequisites: `08-Deployment-Strategies.md`, Topic 2 lesson 12 (replication), Topic 6 lesson 04 (resilience)

---

## 1. The migration problem

Rolling/blue-green deploys me (lesson 08), **purana aur naya code ek saath live** hote hai, ek hi database pe. Agar tum ek column rename kar do ek migration me, purana code (jo old name expect karta) turant toot jaata. Schema changes **backward-compatible** hone chahiye during the transition.

---

## 2. Expand-Migrate-Contract ⭐ (the pattern)

Ek breaking schema change ko teen safe deployable steps me toDo:
```
Goal: rename column `name` → `full_name`

EXPAND (additive only — safe):
  - Naya column `full_name` add karo (nullable, no drop)
  - Dono code versions abhi kaam karte (old uses `name`, new can use either)

MIGRATE (backfill + dual-write):
  - App dono columns me likhe (dual-write)
  - Purana data backfill karo (name → full_name), in batches
  - Naya code `full_name` se padhe

CONTRACT (cleanup — only after fully migrated):
  - Purana code gone (sab instances new) confirm karo
  - `name` column drop karo, dual-write hatao
```
Har step **independently deployable + backward-compatible**. Kisi bhi point pe rollback safe. Ye Topic 6 lesson 09 (strangler fig) ka database-schema version hai.

```
Timeline:
  Deploy 1: EXPAND (add column)      → old + new code both fine
  Deploy 2: MIGRATE (dual-write + backfill) → reading new, writing both
  Deploy 3: CONTRACT (drop old)      → only after 100% on new
```

> ⚠️ **Never in one shot:** `ALTER TABLE ... RENAME` in a single migration = old code instantly broken during the rolling deploy window. Aur bade tables pe, kuch ALTERs poori table lock kar dete hai (Topic 2) — downtime. Additive + batched changes safe hai.

**Say this:** *"Schema changes main expand-migrate-contract se karunga — additive change deploy, dual-write + batched backfill, phir purana drop only after all instances migrated. Har step backward-compatible taaki rolling deploy ke daoran dono code versions kaam karein aur rollback safe rahe."*

**Frontend analogy:** ye ek API contract evolve karne jaisa hai bina clients tode — naya field add karo (optional), clients migrate karo, phir purana field deprecate/remove. Kabhi ek saath breaking change nahi.

---

## 3. Runtime resilience (recap + operational depth)

> **Recap (Topic 6 lesson 04, Topic 4 lesson 07):** har inter-service call ke liye resilience stack. Yahan operational tuning:

| Pattern | Operational detail |
|---|---|
| **Timeout** | Downstream p99 se thoda upar. Har call pe (no infinite). Foundation — Little's Law (lesson 03) se W bound karo |
| **Retry** | Exponential backoff + jitter, idempotent only, retry budget (total retries cap — retry storm rok) |
| **Circuit breaker** | Repeated failures → open (fail fast) → half-open test → close. Cascade rok (lesson 03 pool exhaustion) |
| **Bulkhead** | Separate pools per dependency — ek slow dependency baaki ko na le doobe (Little's Law) |
| **Load shedding** | Overload pe low-priority requests **drop** karo (health check + critical path bachao) — partial > total failure |

### Load shedding (the overload survival move) ⭐
```
System capacity ke paas → naye requests reject karo (429) BEFORE collapse
  Priority: health checks + critical path (payments) > nice-to-have (recommendations)
  → Better: 80% requests served + 20% rejected cleanly
  → Than: 100% requests accepted → queue → everything times out → 0% served
```
Ye queueing wall (lesson 03) ka operational answer hai — jab tum wall ke paas ho, work reject karo taaki curve ke vertical part se bacho.

---

## 4. Disaster Recovery (region-level) ⭐

"What happens when a whole region dies?" — DR strategies (cost vs RTO/RPO — lesson 04):

| Strategy | RTO | RPO | Cost | How |
|---|---|---|---|---|
| **Backup & Restore** | Hours | Hours | $ | Backups another region me; restore on disaster |
| **Pilot Light** | ~10s of min | Minutes | $$ | Core (DB replica) warm; scale up app on failover |
| **Warm Standby** | Minutes | Seconds | $$$ | Full scaled-down copy running; scale up + shift traffic |
| **Active-Active** | ~0 | ~0 | $$$$ | Both regions serve live; one dies → other absorbs |

```
Choose by RTO/RTO (lesson 04):
  Internal tool → backup & restore (cheap, hours OK)
  Standard SaaS → warm standby (minutes, reasonable cost)
  Payments/critical → active-active (near-zero, expensive)
```

> ⚠️ **DR sirf real hai agar tum DRILL karo.** Ek untested DR plan = no DR plan. "We have backups" ≠ "we can restore in our RTO." Regular game days / failover drills — warna disaster ke din pata chalta ki restore 6 ghante leta hai jab RTO 1 ghanta tha.

> **Recap (Topic 2 lesson 12, Topic 5):** cross-region replication (async — RPO = lag window) + failover (promotion + fencing — Topic 5 lesson 05 taaki split-brain na ho). Active-active = multi-leader (Topic 2) with conflict resolution (Topic 5 vector clocks).

**Say this:** *"DR strategy RTO/RPO se choose karunga — payments active-active (near-zero), standard services warm standby (minutes). Cross-region async replication (RPO = lag), automated failover with fencing (no split-brain). Aur regularly drill karunga — untested DR ≠ DR."*

---

## 5. Interview application

```
Q: "How do you rename a column without downtime?"
A: "Expand-migrate-contract: add new column, dual-write + backfill, drop old only
    after all instances migrated. Never a single rename — rolling deploy me old code
    toot jaata."

Q: "What happens when a region fails?"
A: "Depends on RTO/RPO. Warm standby: async cross-region replica, automated failover
    with fencing, ~minutes RTO. Payments: active-active, near-zero. And I'd drill it."

Q: "The system is overloaded. What do you do?"
A: "Load shedding — reject low-priority requests (429) before collapse, protect
    critical path + health checks. Partial service beats total collapse (queueing wall)."
```

**Interview line:** *"Schema changes expand-migrate-contract (backward-compatible, rollback-safe). Runtime: timeouts (bound W), retries with budgets, circuit breakers (cascade rok), bulkheads, aur load shedding under overload. Region failure: DR by RTO/RPO (warm standby to active-active), async replication + fenced failover, drilled regularly."*

---

## Quick Self-Check
1. Rolling deploy me schema change kyu breaking ho sakta (old+new code live)?
2. Expand-migrate-contract ke 3 steps (column rename)?
3. Single-shot `RENAME` kyu khatarnak?
4. Load shedding kya hai — overload pe kya karo (queueing wall se relate)?
5. Retry budget kyu (retry storm)?
6. 4 DR strategies (backup→active-active) — RTO/RPO/cost trade-off?
7. "DR only real if you drill" — kyu?
8. Region failover me fencing kyu (Topic 5 — split-brain)?

---
**Next:** `10-Operations-Playbook.md` — sab tie together + FastAPI.