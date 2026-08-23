# 01 — Monolith vs Microservices

> **Is lesson me:** Do fundamental architecture styles — monolith vs microservices, unke trade-offs, services ko kaise decompose karo (bounded contexts / DDD), aur sabse important — **kab kaunsa** (aur "microservices hamesha better" kyu galat hai). Ye poore module ki foundation hai.

Prerequisites: Topics 1-5 (helpful). Ye Topic 6 ki pehli file hai.

---

## 1. Monolith kya hai

**Monolith** = poora application ek single, unified codebase + deployment ke roop me. Saare features (users, orders, payments, notifications) ek hi process me chalte hai, ek hi database share karte hai.

```
┌─────────────────────────────────────┐
│         MONOLITH (one app)          │
│  ┌────────┐ ┌────────┐ ┌─────────┐  │
│  │ Users  │ │ Orders │ │Payments │  │
│  └────────┘ └────────┘ └─────────┘  │
│         (ek process, ek deploy)      │
│              │                       │
│         ┌────▼────┐                  │
│         │ One DB  │                  │
│         └─────────┘                  │
└─────────────────────────────────────┘
```

**Frontend analogy:** ek monolith ek single-page app jaisa hai jaha saara code ek bundle me hai — sab kuch ek saath deploy hota hai, ek saath chalta hai. Simple to start, but bada hote-hote bundle unwieldy ho jaata hai.

---

## 2. Microservices kya hai

**Microservices** = application ko chhote, independent services me toDna, har ek apna specific business capability handle kare, apna database rakhe, aur independently deploy ho. Services network pe (API/messaging) baat karte hai.

```
┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────────┐
│  Users   │   │  Orders  │   │ Payments  │   │Notifications │
│ Service  │   │ Service  │   │ Service   │   │  Service     │
│    │     │   │    │     │   │    │      │   │     │        │
│ ┌──▼──┐  │   │ ┌──▼──┐  │   │ ┌──▼──┐   │   │  ┌──▼──┐     │
│ │ DB  │  │   │ │ DB  │  │   │ │ DB  │   │   │  │ DB  │     │
│ └─────┘  │   │ └─────┘  │   │ └─────┘   │   │  └─────┘     │
└──────────┘   └──────────┘   └───────────┘   └──────────────┘
   (each: independent deploy, own DB, own team)
     └──────────── API / messaging se baat ──────────────┘
```

**Key principle: har service apna database rakhta hai** (database-per-service). Ek service directly doosre ka DB access nahi karti — sirf uske API/events ke through. Ye loose coupling deta hai (Topic 4 lesson 01).

---

## 3. Head-to-head comparison

| Aspect | Monolith | Microservices |
|---|---|---|
| Codebase | Single | Multiple (per service) |
| Deployment | Ek saath sab | Independent per service |
| Database | Shared | Per-service |
| Scaling | Poora app scale karo | Sirf bottleneck service scale |
| Tech stack | Uniform | Per-service (polyglot) |
| Team | Coordinated | Independent per service |
| Failure blast radius | Ek bug = poora app down | Ek service down, baaki chale |
| Latency | In-process (fast) | Network calls (slower) |
| Complexity | Low (initially) | High (distributed systems!) |
| Debugging | Simple (one process) | Hard (distributed tracing) |
| Transactions | Easy (ACID, one DB) | Hard (saga — lesson 08) |

---

## 4. Microservices ke fayde (kab shine karte hai)

- **Independent scaling** — sirf payment service busy? Sirf use scale karo, poora app nahi (Topic 1 lesson 04).
- **Independent deployment** — ek team apni service deploy kare bina baaki ko block kiye (faster releases).
- **Fault isolation** — recommendation service crash → checkout still works (Topic 4 loose coupling).
- **Tech flexibility** — ek service Python me, doosri Go me, jo fit ho (polyglot — Topic 2 lesson 14).
- **Team autonomy** — chhoti teams apni services own karein (Conway's Law — org structure ↔ architecture).

---

## 5. Microservices ke costs (the hidden tax) ⚠️

Log microservices ko default samajh lete hai — but ye **distributed systems complexity** laata hai (poora Topic 5!):
- **Network calls** — in-process function call (nanoseconds) ab network call (milliseconds — Topic 1 lesson 01). Latency + failure modes.
- **Distributed transactions** — ek DB ki ACID transaction (Topic 2) ab multiple services me — saga chahiye (lesson 08), complex.
- **Data consistency** — har service apna DB → eventual consistency (Topic 5 lesson 03), no easy JOINs.
- **Operational overhead** — 50 services = 50 deployments, monitoring, logging, tracing.
- **Debugging** — ek request 10 services se guzarti hai → distributed tracing chahiye.
- **Testing** — integration testing across services mushkil.

**Interview line:** *"Microservices free nahi hai — ye distributed systems ki saari complexity (network failures, eventual consistency, distributed transactions) laate hai. Ye scaling/team-autonomy problems solve karte hai jo tumhe actually ho — pehle se nahi."*

---

## 6. Decomposition — services kaise baato (the hard part)

Galat decomposition = "distributed monolith" (worst of both — microservices ki complexity + monolith ki tight coupling). Sahi tarika:

### Bounded Contexts (Domain-Driven Design)
Services ko **business domain** ke around baato, na ki technical layers ke around.
```
GALAT (technical layers):        SAHI (business capabilities):
  UI Service                       Order Service (poora order domain)
  Business-Logic Service           Payment Service (poora payment domain)
  Database Service                 Inventory Service (poora inventory domain)
  (har request sab se guzre)       (har service ek capability, self-contained)
```

> **Bounded context kya hai:** ek business domain ki clear boundary jiske andar ek specific model consistent hai. "Order" ka matlab Order Service me ek cheez, Shipping Service me thoda alag — har context apna model rakhta hai. Services in boundaries pe split karo.

**Good service boundaries:**
- **High cohesion** — ek service ka saara code ek business capability ke liye
- **Loose coupling** — services minimal dependencies (Topic 4)
- **Independent data** — har service apna data own kare (no shared DB)
- **Single responsibility** — ek service ek cheez achhe se kare

**Frontend analogy:** ye React components ko feature ke around organize karne jaisa hai (`/features/checkout`, `/features/profile`) na ki technical type ke around (`/components`, `/hooks`, `/utils` sab mixed). Feature-based = high cohesion.

---

## 7. Kab kaunsa — the decision ⭐

| Monolith choose karo jab | Microservices choose karo jab |
|---|---|
| Early-stage / startup (fast iteration) | Large scale, clear domains |
| Small team (< 10-15 devs) | Multiple independent teams |
| Domain abhi unclear (evolving) | Domains well-understood |
| Simple app | Different scaling needs per component |
| Fast time-to-market | Independent deployment critical |

**Golden rule: "Monolith First."** Zyadatar successful systems ek monolith se shuru huye (Amazon, Netflix, Uber) aur **microservices me evolve** kiye jab scaling/team pain real hua. Premature microservices = complexity bina benefit ke.

> **Modular Monolith (the sweet spot):** ek monolith jo internally clean modules me organized ho (clear boundaries, but ek deployment). Best of both early on — simple deployment + clean boundaries jo baad me easily microservices me nikal sakte ho. Interview me ye mature answer hai: *"Modular monolith se shuru, extract services jab specific pain (scaling/team) justify kare."*

**Interview:** *"Main modular monolith se shuru karunga clean bounded contexts ke saath. Jab ek specific module ki alag scaling need ho ya alag team use own kare, use ek service me extract karunga (strangler fig — lesson 09). Microservices ek destination nahi, ek evolution hai."*

---

## Quick Self-Check
1. Monolith vs microservices — deployment, database, scaling ke terms me?
2. "Database-per-service" principle kya hai aur kyu (loose coupling)?
3. Microservices ke 5 fayde?
4. Microservices ka "hidden tax" — 4 costs (Topic 5 se connect)?
5. "Distributed monolith" kya hai aur kyu worst-case?
6. Bounded contexts se decompose karna kya hai (business vs technical)? React analogy?
7. Kab monolith, kab microservices? "Monolith First" + modular monolith kyu?

---
**Next:** `02-API-And-Communication-Styles.md` — services kaise baat karein.