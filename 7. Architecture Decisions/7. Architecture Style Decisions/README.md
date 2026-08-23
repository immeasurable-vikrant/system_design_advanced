# 07 — Architecture Style Decisions (+ FastAPI)

> **Is lesson me:** Bade-picture architecture decisions ka framework — monolith vs microservices, kaunse patterns (CQRS/event-driven/saga) kab, aur consistency-per-component — Topic 6 ko decision-form me. Plus FastAPI project ko kaise structure karo (modular monolith → microservices). "Kab kaunsa pattern" pe focus.

Prerequisites: `01-How-To-Make-Architecture-Decisions.md`, Topic 6 (patterns)

---

## 1. Monolith vs Microservices decision (Topic 6 recap)

> **Recap (Topic 6 lesson 01):** "Monolith first." Microservices ek evolution hai, starting point nahi.
```
Monolith (modular):     early-stage, small team, unclear domain, fast iteration
Microservices:          large scale, multiple teams, clear domains, independent scaling
```

**Decision:**
```
Start → Modular Monolith (clean bounded contexts, ek deployment)
Extract a service WHEN:
  - Ek module ki alag scaling need (e.g. video encoding CPU-heavy)
  - Ek alag team use own kare
  - Ek module ka release cycle alag ho
Via → Strangler Fig (Topic 6 lesson 09), incrementally
```

**Interview line:** *"Modular monolith se shuru — clean module boundaries, ek deployment, simple. Jab ek specific module independent scaling ya team ownership demand kare, use strangler fig se extract karunga. Microservices ki complexity (distributed systems, Topic 5) sirf tab jab wo actually problems solve kare jo mujhe hai."*

---

## 2. FastAPI structure — modular monolith ⭐

Kyunki tum FastAPI use karoge, ek clean modular monolith structure (jo baad me microservices me split ho sake):
```
app/
├── main.py                    # FastAPI app, router registration
├── core/                      # config, DB, shared (Topic 2 lesson 05 — env config)
│   ├── config.py              # settings (env vars — externalized, Topic 6 lesson 03)
│   ├── database.py            # async engine, session
│   └── security.py            # auth, JWT
├── modules/                   # BOUNDED CONTEXTS (Topic 6 lesson 01)
│   ├── orders/
│   │   ├── router.py          # API endpoints
│   │   ├── service.py         # business logic
│   │   ├── models.py          # SQLAlchemy models
│   │   ├── schemas.py         # Pydantic (request/response)
│   │   └── repository.py      # DB access
│   ├── payments/              # same structure, separate module
│   └── users/
└── workers/                   # Celery/Arq tasks (async work — lesson 05)
```

**Key principle:** har `module/` ek **bounded context** hai (Topic 6 lesson 01) — self-contained, clear boundary. Modules ek dusre ke `service.py` ko call karein (internal), na ki directly ek dusre ke DB models. Isse:
- **Clean boundaries** — modules loosely coupled
- **Easy to extract** — ek module ko alag FastAPI service me nikalna easy (already self-contained)
- **Team-friendly** — alag teams alag modules own karein

**Frontend analogy:** ye feature-based folder structure jaisa hai (`/features/orders`, `/features/payments`) — high cohesion per feature, easy to reason about, easy to extract. Same principle, backend.

---

## 3. Which patterns, when (Topic 6 recap)

> **Recap (Topic 6):** patterns solutions hai specific problems ke — problem pehle, pattern baad me.

| Pattern | Use WHEN | Topic 6 |
|---|---|---|
| **Event-Driven** | Multiple services react to events, decouple, fan-out | lesson 05 |
| **CQRS** | Read/write needs genuinely diverge, multiple read models | lesson 06 |
| **Event Sourcing** | Audit trail / temporal queries first-class (finance) | lesson 07 |
| **Saga** | Transaction spans multiple services | lesson 08 |
| **Circuit Breaker/Resilience** | Any inter-service call (essential) | lesson 04 |
| **Strangler Fig** | Migrating monolith → microservices | lesson 09 |

**⚠️ Default: don't use these** (unless justified). CQRS, event-sourcing, saga sab complexity laate hai. Simple CRUD → simple architecture. Add a pattern jab ek specific problem use demand kare.

**Decision:**
```
Simple CRUD app?                          → Modular monolith, REST, Postgres, done
Multiple services + shared events?         → Event-driven (Topic 6 lesson 05)
Read/write diverge hugely?                 → CQRS (Topic 6 lesson 06)
Audit critical (finance/healthcare)?       → Event sourcing (Topic 6 lesson 07)
Cross-service transaction?                 → Saga (Topic 6 lesson 08)
```

---

## 4. Consistency per component (Topic 5 recap)

> **Recap (Topic 5 lesson 02):** "Mera system CP hai" galat. Per-component.
```
Payment/wallet    → strong consistency (CP), single-leader Postgres
Feed/notifications → eventual (AP), cache + replicas + async
Inventory         → strong (oversell rok)
Analytics         → eventual (approximate ok)
```
Ye poore system ke liye ek choice nahi — har component ki apni consistency need.

---

## 5. The architecture decision checklist

```
1. Monolith or microservices? → Modular monolith first, extract when justified
2. FastAPI structure → modules = bounded contexts, self-contained
3. Which patterns? → default none; add per specific problem (CQRS/saga/ES/events)
4. Consistency per component → CP (money) vs AP (feed) — Topic 5
5. Resilience everywhere → timeout/circuit-breaker/retry (Topic 6 lesson 04)
6. Evolve via strangler fig (Topic 6 lesson 09), never big-bang
```

**Interview line (FastAPI):** *"Modular monolith in FastAPI — modules as bounded contexts (orders, payments, users), each self-contained (router/service/models/schemas). Simplicity by default. As scaling/team needs emerge, extract modules into separate FastAPI services via strangler fig. Patterns (CQRS/saga/event-sourcing) only where a specific problem demands — not preemptively. Consistency per component: strong for payments, eventual for feeds."*

---

## Quick Self-Check
1. Monolith vs microservices decision — "monolith first" kyu (Topic 6)?
2. Kab ek module ko service me extract karo (3 triggers)?
3. FastAPI modular structure — module = kya (bounded context)? Kyu extract-friendly?
4. Modules ek dusre se kaise baat karein (service, not DB models)?
5. CQRS/event-sourcing/saga — default use karo ya nahi? Kab?
6. Consistency per-component — payment vs feed?
7. Architecture evolve kaise (strangler fig, Topic 6)?

---
**Next:** `08-The-FastAPI-Stack.md` — FastAPI-specific decisions in depth.