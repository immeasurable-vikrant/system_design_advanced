# 05 — Communication Decisions (+ FastAPI)

> **Is lesson me:** Communication ke decisions ka framework — sync vs async, REST/GraphQL/gRPC, real-time (WebSocket/SSE), aur message queues — Topics 1 & 4 ko decision-form me. Plus FastAPI ke saath ye kaise implement karo (async endpoints, background tasks, WebSockets, Celery). "Kab kaunsa" pe focus.

Prerequisites: `04-Data-Storage-Decisions.md`, Topic 1 (API), Topic 4 (messaging/real-time)

---

## 1. Sync vs Async — the first decision

> **Recap (Topic 4 lesson 01):** user jo wait kare wo sync (request-response); background wala async (queue).
```
User result ka wait kar raha? (login, search, checkout confirm)  → SYNC
Background me ho sakta? (email, video encode, analytics)         → ASYNC (queue)
```

**Example — video upload:**
```
SYNC:  "Upload received" turant (user wait kar raha)
ASYNC: encoding, thumbnails, notifications → queue → workers (Topic 4)
```

⚠️ **Naming clarity:** "async" ke 2 matlab hai —
- **Architectural async** = message queues, decoupled services (Topic 4)
- **Code-level async** = FastAPI `async def`, concurrent I/O within one process (lesson 08)

Dono alag hai. Ye lesson **architectural** async pe (queues); code-level async lesson 08 me.

---

## 2. Sync API style — REST vs GraphQL vs gRPC

> **Recap (Topic 6 lesson 02):**
```
Public API / simple CRUD / caching → REST
Mobile / flexible nested data       → GraphQL
Internal service-to-service / perf  → gRPC
```

**FastAPI context:** FastAPI **REST-first** hai (aur excellent at it — auto OpenAPI/Swagger docs, Pydantic validation). GraphQL FastAPI ke saath possible (Strawberry/Ariadne libraries). gRPC alag hai (FastAPI HTTP-based; gRPC ke liye `grpcio` ya ek alag gRPC service).

```
FastAPI ke liye:
  External REST API  → FastAPI native (best fit) ✅
  GraphQL            → Strawberry + FastAPI (if needed)
  Internal gRPC      → separate grpcio service, ya FastAPI REST internally
```

**Interview line:** *"External API REST via FastAPI — auto-docs, validation, simple, cacheable. Internal high-throughput service-to-service gRPC ke liye main ek alag gRPC service consider karunga, ya agar simplicity chahiye toh REST internally bhi theek. GraphQL sirf tab jab clients ki data needs bahut vary karein."*

---

## 3. Real-time decision (Topic 4 recap)

> **Recap (Topic 4 lesson 02):**
```
One-way server push (notifications, live scores) → SSE
Bidirectional (chat, collab, games)             → WebSocket
Infrequent updates                              → polling
```

**FastAPI real-time:** FastAPI (Starlette pe) **WebSockets natively** support karta hai:
```python
@app.websocket("/ws/{room}")
async def websocket_endpoint(ws: WebSocket, room: str):
    await ws.accept()
    while True:
        msg = await ws.receive_text()
        # broadcast via Redis pub/sub (Topic 4 lesson 03 — scaling!)
        await ws.send_text(f"echo: {msg}")
```
> **Recap (Topic 4 lesson 03):** WebSockets scale karne ko Redis pub/sub backplane + connection registry chahiye. FastAPI ek instance me WebSockets handle karta hai; multiple instances ke across pub/sub se coordinate. SSE bhi FastAPI me easy (`StreamingResponse`).

---

## 4. Message queue decision (Topic 4 recap)

> **Recap (Topic 4 lesson 06):**
```
High throughput / replay / multiple consumers → Kafka
Simple managed tasks (AWS)                     → SQS
Complex routing / RPC                          → RabbitMQ
```

**FastAPI + async work — 3 levels:**

### Level 1: FastAPI BackgroundTasks (light, in-process)
```python
from fastapi import BackgroundTasks

@app.post("/signup")
async def signup(user: UserCreate, bg: BackgroundTasks):
    created = await create_user(user)
    bg.add_task(send_welcome_email, created.email)  # after response, same process
    return created   # user ko turant response, email background me
```
**Kab:** light, fire-and-forget, ok-if-lost (welcome email). **Limitation:** same process — server crash = task lost, no retries, no scaling. Heavy/critical kaam ke liye nahi.

### Level 2: Task queue — Celery / Arq / Dramatiq (heavy, durable)
```python
# Celery task (separate worker processes — Topic 4 durable queue)
@celery.task(bind=True, max_retries=3)
def process_video(self, video_id):
    # heavy work in a SEPARATE worker (not FastAPI process)
    # retries, DLQ, scaling — Topic 4 lesson 07
    ...

@app.post("/videos")
async def upload(video: Video):
    process_video.delay(video.id)   # enqueue → worker picks up
    return {"status": "processing"}
```
**Kab:** heavy work (video encoding), needs retries/durability/scaling, CPU-bound (workers = separate processes, GIL bypass — lesson 08). **Arq** = async-native alternative to Celery (Redis-based), FastAPI ke saath clean fit.

### Level 3: Full event streaming — Kafka (Topic 4)
Multiple services consume events, replay, high throughput → Kafka + FastAPI producer/consumer.

**Decision:**
```
Light, ok-if-lost, in-process     → BackgroundTasks
Heavy/critical, retries, scaling  → Celery/Arq (separate workers)
Event streaming, multi-consumer   → Kafka
```

> ⚠️ **Common mistake:** heavy/critical work FastAPI BackgroundTasks me daalna. Wo same process me chalta hai — crash pe lost, no retry, aur FastAPI workers ko block kar sakta hai. Critical async work = proper task queue (Celery/Arq).

---

## 5. The communication decision checklist

```
1. Sync ya async? → user waiting = sync; background = async queue
2. Sync style? → REST (FastAPI default), GraphQL (flexible), gRPC (internal perf)
3. Real-time? → SSE (one-way), WebSocket (bidirectional) — FastAPI native
4. Async work weight? → BackgroundTasks (light) | Celery/Arq (heavy) | Kafka (streaming)
5. Reliability? → at-least-once + idempotency + DLQ (Topic 4 lesson 05, 07)
```

**Interview line (FastAPI):** *"External REST via FastAPI (auto-docs, Pydantic). Real-time via native WebSockets + Redis pub/sub backplane for scaling. Light async (emails) via BackgroundTasks; heavy/critical async (video, batch) via Celery or Arq workers — separate processes, retries, DLQ. Event-driven cross-service via Kafka. Idempotency keys on critical operations."*

---

## Quick Self-Check
1. Sync vs async decision — user waiting? Video upload example?
2. "Architectural async" vs "code-level async" — difference?
3. FastAPI kis API style me native/best (REST)? gRPC ke liye kya?
4. FastAPI WebSockets scale karne ko kya chahiye (Topic 4 recap)?
5. BackgroundTasks vs Celery/Arq — kab kaunsa (light vs heavy)?
6. Heavy work BackgroundTasks me kyu nahi (crash, no retry, blocking)?
7. Async work ke 3 levels (in-process → task queue → streaming)?

---
**Next:** `06-Scaling-And-Performance-Decisions.md` — cache/replicate/shard, when (+ FastAPI).