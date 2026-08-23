# 02 — API & Communication Styles (REST vs GraphQL vs gRPC)

> **Is lesson me:** Services (aur clients) ek dusre se kaise baat karein — REST, GraphQL, gRPC ka comparison, sync vs async choice, aur API design basics (versioning, pagination, idempotency). Topic 1 (API gateway) + Topic 4 (async messaging) ke upar ye "kaunsa protocol/style" wala decision hai.

Prerequisites: `01-Monolith-vs-Microservices.md`, Topic 1 lesson 02 (HTTP), Topic 4 lesson 01 (sync/async)

---

## 1. Do fundamental choices

Service communication me do decisions hai:
```
1. SYNC ya ASYNC? (Topic 4 lesson 01)
   Sync (request-response): caller wait kare — REST, GraphQL, gRPC
   Async (messaging): fire-and-forget — Kafka, SQS (Topic 4)

2. Agar SYNC, kaunsa STYLE?
   REST, GraphQL, ya gRPC
```

Ye lesson mainly **sync styles** pe hai (async Topic 4 me covered). Rule of thumb: **user jo wait kare wo sync; background wala async.**

---

## 2. REST — the default

**REST (Representational State Transfer)** = resources ko URLs se represent karo, HTTP methods se operate karo.
```
GET    /users/123        → user 123 fetch
POST   /users            → naya user create
PUT    /users/123        → user 123 update (full)
PATCH  /users/123        → user 123 partial update
DELETE /users/123        → user 123 delete
```
**Principles:** stateless (har request self-contained — Topic 1 lesson 04), resource-based URLs, standard HTTP methods + status codes (Topic 1 lesson 02), cacheable (GET responses — Topic 3).

**Pros:** simple, universal, HTTP caching se kaam karta hai (CDN — Topic 1 lesson 07), har language support, easy debugging (curl).
**Cons:** **over-fetching** (endpoint zyada data deta hai jitna chahiye), **under-fetching** (ek screen ke liye multiple calls — N+1), fixed response shape.

**Frontend analogy:** tumne 1000 baar REST APIs `fetch` ki hai. `GET /api/users` — yehi REST hai.

---

## 3. GraphQL — client-driven queries

**GraphQL** = client **exactly** batata hai use kaunsa data chahiye, ek single query me. Server wahi return karta hai — na zyada, na kam.
```graphql
query {
  user(id: 123) {
    name
    posts(last: 3) { title }     # sirf jo chahiye
  }
}
```
Ek request me user + uske last 3 posts ke titles — REST me ye 2+ calls hoti (user, phir posts).

**Pros:** **no over/under-fetching** (client exact data maange), single request for complex data (mobile-friendly — Topic 1 RTT), strongly-typed schema, evolve without versioning.
**Cons:** caching mushkil (POST-based, har query alag — Topic 3 ke HTTP caching se kaam nahi karta), complex server, query cost/depth control chahiye (malicious deep queries), **N+1 problem** backend pe (DataLoader se fix).

**Kab:** mobile apps (bandwidth-sensitive), complex/nested data, kai clients ke alag data needs (BFF alternative — Topic 1 lesson 08).

---

## 4. gRPC — high-performance internal

**gRPC** = Google ka RPC framework, HTTP/2 pe, binary protocol (Protocol Buffers) use karta hai.
```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc StreamUsers(Filter) returns (stream User);   // streaming!
}
```
> **Recap (Topic 1 lesson 02):** gRPC HTTP/2 pe chalta hai (multiplexing), binary "protobuf" use karta hai (JSON se compact + fast), aur streaming support karta hai. Internal microservices ke beech popular.

**Pros:** **very fast** (binary, HTTP/2 multiplexing), streaming (bidirectional), strongly-typed contracts (protobuf), code generation (client/server stubs auto).
**Cons:** browsers directly support nahi karte (gRPC-Web chahiye), binary = harder debugging, learning curve.

**Kab:** **internal service-to-service** communication (high throughput, low latency), streaming needs, polyglot microservices (contract-first).

---

## 5. Comparison + decision

| Feature | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1+ | HTTP (usually POST) | HTTP/2 |
| Format | JSON | JSON | Binary (protobuf) |
| Data fetching | Fixed per endpoint | Client-specified | Fixed per method |
| Performance | Good | Good | **Best** |
| Caching | **Easy (HTTP)** | Hard | Hard |
| Browser support | **Native** | Native | Needs gRPC-Web |
| Streaming | Limited (SSE) | Subscriptions | **Native** |
| Best for | Public APIs, CRUD | Flexible clients, mobile | Internal microservices |

**Decision:**
```
Public API / simple CRUD / caching matters   → REST
Mobile / complex nested data / many clients   → GraphQL
Internal service-to-service / performance     → gRPC
```

**Interview line:** *"External-facing public API → REST (universal, cacheable, simple). Mobile clients with varied data needs → GraphQL (no over-fetching). Internal microservices → gRPC (binary, HTTP/2, streaming, typed contracts). Ek system me teeno mix ho sakte hai — REST/GraphQL at the edge, gRPC internally."*

---

## 6. API design basics (jo interview me matter karte hai)

Chahe koi bhi style ho, ye principles apply:

### Versioning
```
/v1/users, /v2/users     (URL-based — most common, cacheable, gateway routable — Topic 1 lesson 08)
Accept: version=2         (header-based — cleaner URLs)
```
Kabhi breaking changes bina versioning mat karo — existing clients toot jaayenge.

### Pagination (large lists)
```
Offset:  GET /users?limit=20&offset=40     (simple, but deep offset slow — Topic 2)
Cursor:  GET /users?limit=20&after=abc123  (scalable, stable — large datasets ke liye)
```
> **Recap (Topic 2 lesson 03):** offset pagination `LIMIT/OFFSET` — deep pages slow. Cursor pagination stable + fast at scale.

### Idempotency (safe retries)
> **Recap (Topic 4 lesson 05):** POST idempotent nahi. Client ek `Idempotency-Key` header bheje → server duplicate request pe pehla result return kare (double-charge na ho). Payment APIs me zaroori.

### Rate limiting
> **Recap (Topic 3 lesson 07-08):** per-user/API-key limits, `429` + `Retry-After`. API gateway pe enforce (Topic 1 lesson 08).

---

## Quick Self-Check
1. Sync vs async communication choice — kab kaunsa (Topic 4 recap)?
2. REST ka over-fetching aur under-fetching problem kya?
3. GraphQL ye kaise solve karta hai? Uska caching downside kya?
4. gRPC kis pe chalta hai, kya use karta hai (protobuf), aur kab best?
5. REST vs GraphQL vs gRPC — ek-ek line me best-for?
6. API versioning kyu zaroori, aur 2 approaches?
7. Offset vs cursor pagination — kaunsa scale pe better (Topic 2)?
8. Idempotency key POST me kyu (Topic 4 recap)?

---
**Next:** `03-Service-Discovery-And-Config.md` — services ek dusre ko dhundhte kaise hai.