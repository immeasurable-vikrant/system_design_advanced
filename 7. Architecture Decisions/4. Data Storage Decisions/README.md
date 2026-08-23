# 04 — Data Storage Decisions (+ FastAPI)

> **Is lesson me:** Data layer ke decisions ka framework — SQL vs NoSQL, kaunsa specific DB, consistency choice — Topics 2 & 5 ke gyan ko ek decision tree me distill karke. Plus FastAPI ke saath data layer kaise set karo (async SQLAlchemy, connection pooling, Pydantic). Ye dobara nahi sikhata; "kab kaunsa" pe focus.

Prerequisites: `03-Capacity-Estimation.md`, Topic 2 (databases), Topic 5 (consistency)

---

## 1. The first fork — SQL or NoSQL?

> **Recap (Topic 2 lesson 01, 14):** SQL = structured, relational, ACID, JOINs. NoSQL = flexible, horizontally-scalable, often eventual. Ye "old vs new" nahi — **access-pattern-based trade-off**.

**Decision tree:**
```
Data relational hai? (entities with relationships, JOINs chahiye?)
  HAAN → transactions/strong consistency chahiye? (money, orders)
           HAAN → SQL (Postgres/MySQL) ✅
           NAHI → SQL still fine, ya document DB
  NAHI → access pattern kya?
           Key-value lookups, massive scale → DynamoDB / Cassandra
           Flexible documents → MongoDB
           Full-text search → Elasticsearch
           Time-series → TimescaleDB / InfluxDB
           Graph/relationships → Neo4j
```

**Default: SQL (PostgreSQL).** Zyadatar apps relational hai, aur Postgres bahut door tak scale karta hai (read replicas + caching). NoSQL tab jab specific need ho (massive scale, flexible schema, special access pattern).

**Interview line:** *"Main Postgres se default karunga — relational data, ACID, flexible queries, aur ye caching + read replicas ke saath bahut scale karta hai. NoSQL tab switch karunga jab ek specific need ho: DynamoDB for key-value at massive scale, Elasticsearch for search, Redis for cache/sessions. Zyadatar systems polyglot hote hai — Postgres core + specialized stores."*

---

## 2. Which SQL / which NoSQL

> **Recap (Topic 2 lesson 14 — the master decision table):**

| Need | Store |
|---|---|
| Relational + transactions | **PostgreSQL** (default) |
| Key-value, massive scale, predictable latency | DynamoDB |
| Write-heavy, feeds, time-series | Cassandra |
| Flexible documents | MongoDB |
| Cache, sessions, leaderboards | Redis |
| Full-text search | Elasticsearch |
| Files/media | S3 (object storage) |
| Events/streaming | Kafka |

**Polyglot persistence — real systems mix:**
```
E-commerce:
  Orders/payments → PostgreSQL (ACID)
  Product search  → Elasticsearch
  Cart/sessions   → Redis
  Images          → S3 + CDN
  Activity feed   → Cassandra
```

---

## 3. Consistency decision (Topic 5 recap)

> **Recap (Topic 5 lesson 02-03):** per-component, not whole-system. CP (strong) for money/inventory; AP (eventual) for feeds/sessions.
```
Money / inventory / bookings → Strong consistency (SQL, single-leader) — CP
Feed / likes / analytics     → Eventual consistency (cache, replicas) — AP
User's own data              → Read-your-writes (write→primary, else replica)
```

---

## 4. FastAPI + data layer — concrete setup ⭐

Kyunki tum FastAPI use karoge, data layer aise set karo:

### Async SQLAlchemy + asyncpg (Postgres)
FastAPI async hai, toh **async database driver** use karo (warna async ka fayda nahi):
```python
# asyncpg = async Postgres driver (Topic 1 lesson 05 — driver)
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20,          # connection pool (Topic 1 lesson 09!)
    max_overflow=10,       # burst capacity
    pool_pre_ping=True,    # dead connection detect
)
AsyncSession = async_sessionmaker(engine, expire_on_commit=False)
```

> **Recap (Topic 1 lesson 09 — connection pooling):** `pool_size` DB cores ke hisaab se (~10-30). SQLAlchemy ka pool async me bhi kaam karta hai. Serverless me PgBouncer/RDS Proxy (Topic 1 lesson 09).

### Pydantic — validation + serialization (FastAPI ka core)
```python
from pydantic import BaseModel

class OrderCreate(BaseModel):     # request validation
    user_id: int
    items: list[int]
    total: float

class OrderResponse(BaseModel):   # response shape (CQRS read model jaisa — Topic 6!)
    id: int
    status: str
    total: float
```
Pydantic request ko validate karta hai (galat data → auto 422 error) aur response ko shape karta hai. **Ye ek natural "API contract" layer hai** (Topic 6 lesson 02).

### Dependency injection for DB sessions
```python
async def get_db():
    async with AsyncSession() as session:
        yield session      # request ke baad auto-close (connection wapas pool me)

@app.post("/orders")
async def create_order(order: OrderCreate, db=Depends(get_db)):
    # order already validated by Pydantic
    # db session pool se borrowed, request ke baad auto-returned
    ...
```

> ⚠️ **FastAPI async me sync DB driver mat use karo** (jaise plain psycopg2 sync mode) — wo event loop block karega aur async concurrency mar jaayegi. Async endpoints me async driver (asyncpg). Ya sync driver ke saath sync `def` endpoint (FastAPI use threadpool me chalayega). **Mix mat karo galat tarike se.** (Detail lesson 08.)

---

## 5. ORM choice for FastAPI

| Option | Kab |
|---|---|
| **SQLAlchemy (async)** | Most powerful, mature, complex queries — default choice |
| **SQLModel** | FastAPI author ka, SQLAlchemy + Pydantic combined, simpler | 
| **Tortoise ORM** | Django-like, async-native |
| Raw asyncpg | Max performance, hot paths (Topic 2 lesson 05 — raw SQL for critical) |

> **Recap (Topic 2 lesson 05):** ORM convenient but kabhi inefficient SQL. Hot queries raw SQL me. SQLModel FastAPI ke saath clean fit hai (ek model request/response/DB ke liye).

---

## 6. The data decision checklist

```
1. Relational + transactions?     → PostgreSQL (default)
2. Special access pattern?         → matching NoSQL (KV/doc/search/graph)
3. Consistency per component?      → CP (money) vs AP (feed) — Topic 5
4. Scale (from estimation)?        → replicas (reads) → shard (writes) — Topic 2
5. Media/files?                    → S3 + CDN — Topic 2, 1
6. FastAPI: async driver + pool + Pydantic models + DI sessions
```

**Interview line (FastAPI context):** *"Postgres via async SQLAlchemy + asyncpg (async driver, warna FastAPI ka async benefit nahi milega). Connection pool ~20, Pydantic for validation + response shaping. Reads scale karne ko read replicas + Redis cache; writes bahut badhe toh shard. Media S3 + CDN. Ek polyglot setup — Postgres core, Redis cache, S3 media."*

---

## Quick Self-Check
1. SQL vs NoSQL decision tree — pehla sawaal kya (relational + transactions)?
2. Default kyu Postgres, aur NoSQL kab?
3. Polyglot persistence — e-commerce me 4 alag stores kyu?
4. Consistency per-component kaise (money vs feed — Topic 5)?
5. FastAPI me async DB driver kyu zaroori (sync driver ka problem)?
6. Pydantic ka data layer me role (validation + response shape)? Topic 6 se connect?
7. FastAPI DB sessions dependency injection se kyu (pool borrow/return)?
8. SQLModel vs SQLAlchemy — kab kaunsa?

---
**Next:** `05-Communication-Decisions.md` — sync/async, REST/gRPC (+ FastAPI async).