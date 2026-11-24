# 09 — Connection Pooling

> **Is lesson me:** "Connection" asal me kya hai aur kab banta hai, pool kab banta hai, FastAPI + SQLAlchemy me `db.add()` / `db.query()` / `db.commit()` pe andar kya hota hai, connections mehenge kyu hai, pool kaise kaam karta hai, **pool sizing** (CPU cores, disk spindles, aur "zyada connections = better" kyu galat hai), pool config parameters, application-side vs proxy-side pooling (PgBouncer), connection leaks, aur serverless ka special problem. Topic 2 me isko chhua tha — ye full depth hai, aur infra→database ka bridge.

Prerequisites: `01-Networking-I` (TCP handshake, RTT, socket). Helpful: Topic 2 (Database), lesson 04 (concurrency, Little's Law).

---

## 1. Connection pooling kya hai

**Connection pooling** = pehle se bani hui (pre-established) database connections ka ek set maintain karna jo requests ke across **reuse** hota hai — har query ke liye nayi connection banane aur band karne ke bajaye.

**Analogy:** Ek busy office building with a security gate. Bina pooling, har employee poora security check (ID, bag scan, badge print) har baar karta hai — chahe wo sirf coffee ke liye bahar gaya ho. Pooling ke saath, employees apna badge (connection) poore din active rakhte hai — enter karne ke liye bas tap. Badges (connections) limited hai, toh sab busy ho toh naye employees line me wait karte hai.

---

## 2. Kaunsa "connection"? Kab banta hai? Pool kab banta hai? ⭐

### Connection = app process aur DB ke beech ek khuli, logged-in line
Yahan **browser ↔ server** wale HTTP connection ki baat **nahi** ho rahi. Ye tumhare **FastAPI process ↔ PostgreSQL** ke beech ka connection hai:

```
FastAPI worker process ══ TCP socket (lesson 01) ══▶ PostgreSQL :5432
                          + TLS (agar enabled)
                          + login (user/password)
                          + DB side pe ek dedicated "backend process" + memory
```
- App side: ek **socket** (OS-level, lesson 01) jo `IP:5432` se juda hai.
- DB side: PostgreSQL **har connection ke liye ek alag process** chalata hai (kuch MB memory). Isliye connections free nahi hai — DB ki side bhi kharch hota hai.
- Isi connection pe tumhara **transaction** chalta hai. `BEGIN … INSERT … COMMIT` sab **ek hi connection** pe hona chahiye — beech me connection badal nahi sakte.

### Humein connection chahiye hi kyu?
DB koi function nahi jo seedha call ho jaye — woh ek **alag server** hai jo network pe apni protocol (Postgres wire protocol) bolta hai. Usse baat karne ke liye pehle line jodni padti hai (TCP + TLS + login). Line ke bina SQL bhej hi nahi sakte.

### Pool kyu chahiye?
1. **Speed** — har query pe line jodna 10-50ms hai (section 3); pool me pehle se judi line 0.1ms me mil jaati hai.
2. **Limit / bouncer** — pool ek **max** rakhta hai, taaki tumhara app DB pe 1000 connections ka pahaad na gira de (section 5). Pool sirf cache nahi, ek **gatekeeper** bhi hai.

### Kab kya banta hai (SQLAlchemy me)
```python
# database.py — app start hote hi (har uvicorn WORKER process me ek baar import hota hai)
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    DATABASE_URL,
    pool_size=5,        # 5 connections pool me "rakhi" jaati hai (reuse ke liye)
    max_overflow=10,    # load pe 10 aur extra ban sakti hai → max 15 ek waqt pe
    pool_timeout=10,    # 15 busy? 10s tak wait, phir error (default 30s)
    pool_recycle=1800,  # 30 min purani connection band karke nayi (stale/LB idle-timeout se bacho)
    pool_pre_ping=True, # borrow se pehle "zinda ho?" check (mari hui conn pe query mat bhejo)
)
# NOTE: abhi tak EK BHI connection nahi bani. Pool object bana hai, connections LAZY bante hai.

SessionLocal = sessionmaker(bind=engine)

def get_db():
    db = SessionLocal()   # Session bana — abhi bhi connection NAHI liya
    try:
        yield db          # route handler isse use karega
    finally:
        db.close()        # connection POOL ME WAPAS (TCP band nahi hota). Exception me bhi chalega.
```

| Kab | Kya hota hai |
|---|---|
| `create_engine(...)` (app start) | **Pool object** banta hai. Connections abhi zero |
| Pehli query kabhi bhi | Pool khaali → **pehli asli connection** banti hai (DNS + TCP + TLS + login) |
| Baad ki requests | Pool se **purani connection reuse** |
| `pool_recycle` / DB restart / idle timeout | Purani band, zarurat pe nayi |

> ⚠️ **Har worker process ka apna pool hota hai.** `uvicorn --workers 4` = 4 alag processes = 4 alag pools. Ek instance ka max = 4 × 15 = **60 connections**. Ye math section 7 me bahut matter karega.

---

## 3. `db.add()`, `db.query()`, `db.commit()` — andar kya hota hai?

```python
@app.post("/users")
def create_user(data: UserIn, db: Session = Depends(get_db)):
    user = User(name=data.name)
    db.add(user)               # (1)
    existing = db.query(User).filter_by(email=data.email).first()  # (2)
    db.commit()                # (3)
    return user
```

| Step | DB se baat hui? | Connection ka kya hua |
|---|---|---|
| `SessionLocal()` | Nahi | Kuch nahi — session bas ek "notebook" hai |
| **(1) `db.add(user)`** | **Nahi** | Kuch nahi. Object sirf session ki memory me "pending" mark hua. Ye query **nahi** hai |
| **(2) `db.query(...).first()`** | **Haan** | Session pool se **connection borrow** karta hai → `BEGIN` (transaction shuru) → pending `INSERT` pehle bhejta hai (autoflush) → phir `SELECT`. Ab connection is session ke paas **pakdi hui** hai |
| **(3) `db.commit()`** | **Haan** | Bacha hua SQL bhejta hai → `COMMIT` → transaction khatam → connection **pool me wapas** |
| `db.close()` (get_db ka finally) | Shayad | Koi transaction khula ho toh `ROLLBACK`, phir connection pool me wapas |

- **Query** = jo actual SQL DB tak jaata hai (`SELECT`, `INSERT`, `UPDATE`, `COMMIT`). `db.add()` query nahi, `db.query()`/`execute()`/`commit()` queries bhejte hai.
- **`cursor()`** (raw psycopg/DB-API) = ek connection ke **andar** query chalane aur results padhne ka handle. Naya connection **nahi**. `conn.cursor().execute("SELECT ...")` → wahi connection.
- **`db.save()`** SQLAlchemy me nahi hota — woh Django ORM (`obj.save()`) ka hai; andar same idea: connection lo, `INSERT/UPDATE`, commit.
- Async (`create_async_engine` + asyncpg) me bhi bilkul same pool concept, bas `await`.

### Diagram me "App request" ka matlab?
Neeche wale flow me **"app request" = jab tumhara code ko DB connection chahiye** (jaise step (2) pe session ne connection maangi) — browser ki HTTP request nahi. Ek HTTP request me ye ek baar ho sakta hai (ek session), ya kai baar.

---

## 4. Connections mehenge kyu hai

Ek nayi database connection banane me:
| Step | Action | Time |
|---|---|---|
| 1 | DNS resolution (DB ka hostname → IP — lesson 03) | 1-5 ms |
| 2 | TCP handshake (SYN/ACK — lesson 01) | 1-3 ms |
| 3 | TLS handshake (if SSL — lesson 03) | 5-30 ms |
| 4 | Authentication (username/password) | 2-10 ms |
| 5 | DB pe us connection ke liye process + memory allocate | 5-10 MB per connection |
| 6 | Connection ready | **Total: 10-50 ms** |

Ek web request jo 2ms ki query kare, uspe 30ms connection setup = **15x overhead**. 1000 RPS pe = 1000 connections/sec banao-todo = massive waste (DB pe har baar process fork bhi).

```
Without pool (naya per request):        With pool (reuse):
Request 1 → create conn (30ms) →        Request 1 → borrow (0.1ms) →
  query (2ms) → close                     query (2ms) → return to pool
Request 2 → create conn (30ms) →        Request 2 → borrow SAME conn (0.1ms) →
  query (2ms) → close                     query (2ms) → return
```

> **Yaad hai lesson 01** — TCP + TLS handshake ka RTT tax aur keep-alive? Connection pooling exactly wahi tax bachata hai, database connections ke liye — pehle se khuli connections reuse karke. (Browser ka keep-alive = HTTP ke liye pooling; SQLAlchemy pool = DB ke liye.)

---

## 5. Pool kaise kaam karta hai

```
Code ko connection chahiye (session ki pehli query) → pool me idle connection hai?
   HAAN → borrow karo
   NAHI → pool max size pe hai? (pool_size + max_overflow)
            NAHI → nayi conn banao, pool me add
            HAAN → queue me wait karo
                     → pool_timeout exceed? → connection timeout error
→ query execute → commit/close → connection wapas pool me (idle, reuse ready)
```

**Pool lifecycle:**
1. **Initialization** — Java HikariCP startup pe `minIdle` connections pehle se bana leta hai; SQLAlchemy lazy hai (pehli zarurat pe)
2. **Borrow (checkout)** — code maange → pool ek de
3. **Use** — queries chalao
4. **Return (checkin)** — commit/close pe wapas (TCP band nahi hota)
5. **Validation** — pool check kare connection abhi zinda hai (`pool_pre_ping`)
6. **Eviction** — extra (overflow) ya bahut purani (`pool_recycle`) connections band

---

## 6. Pool Sizing — "zyada = better" GALAT hai ⭐

### Pehle do words
- **CPU core** = processor chip ke andar ek independent "dimaag" jo ek waqt pe ek thread ke instructions chala sakta hai. 8-core machine = ek instant pe 8 kaam **truly parallel** (lesson 04). (Hyper-threading se ek core 2 threads ko thoda overlap karwa leta hai.) **Yahan baat DB server ke cores ki hai**, tumhare app server ki nahi.
- **Disk spindle** = purani **hard disk (HDD)** ka ghoomta hua platter + read-head. Ek spindle ek waqt pe ek jagah se data padh sakta hai (head ko wahan "seek" karna padta hai ~5-10ms). Zyada disks (RAID) = zyada spindles = zyada parallel disk reads.

**Formula me ye dono kyu:** ek query kabhi CPU pe kaam kar rahi hoti hai, kabhi **disk ka wait** kar rahi hoti hai. Jab ek query disk ka wait kare, us core pe doosri query chal sakti hai. Isliye cores × 2 (wait ke time ka overlap) + spindles (har disk ek extra parallel I/O).

**Formula (PostgreSQL wiki / HikariCP "About Pool Sizing"):**
```
Pool Size = (CPU cores × 2) + effective disk spindles
SSD-backed ke liye roughly: Pool Size ≈ CPU cores × 2 + 1
```
SSD me spindle nahi hota aur woh ek saath bahut saare reads kar leta hai, toh formula sirf **starting point** hai — asli number load test se nikalo.

> ⚠️ **Ye number DB ke liye TOTAL hai** — saare app instances aur workers ko milake. "Har worker ka pool 17" nahi. Isi wajah se section 8 (PgBouncer) zaroori ban jaata hai.

**Kyu chhote pools better:**
| Pool Size | Behavior |
|---|---|
| Too small (5) | Requests queue up, high latency under load |
| **Optimal (10-20)** | Max throughput, CPU + disk fully utilized |
| Too large (200) | **Context switching overhead, lock contention, WORSE throughput** |

Ek 4-core PostgreSQL server usually **9-10 connections** pe best perform karta hai, **200 pe nahi**.

> **Kyu 200 connections 20 se slow (interview Q):** database ke paas limited CPU cores hai (say 8). 200 concurrent connections sab queries chala rahi ho → DB actual query execute karne se zyada time **context-switching** (ek process se doosre pe jaana) me lagata hai. Lock contention badhti hai, CPU cache misses badhte hai, disk I/O random ho jaata hai. 20 connections DB ko full CPU se 20 queries pe focus karne dete hai, baaki briefly pool queue me wait — higher overall throughput, lower average latency. (Lesson 04 ka Little's Law: 2ms queries pe 10K QPS ko bhi sirf ~20 in-flight chahiye.)

**Frontend analogy:** ek chef (CPU core) ek time pe 2-3 dishes achhe se bana sakta hai. Usse 200 orders ek saath do → wo har dish ke beech switch karta rahega, kuch achha nahi banega. 20 orders = focused, fast. Yehi context-switching overhead hai.

---

## 7. Pool config parameters

| Parameter (HikariCP / general) | SQLAlchemy me | Kya karta hai | Recommended |
|---|---|---|---|
| `maxPoolSize` | `pool_size + max_overflow` | Max connections (per pool) | 10-30 total across fleet (DB cores pe depend) |
| `minIdle` | ~`pool_size` (itni rakhi jaati hai) | Idle pe kitni alive rakho | 5-10 |
| `connectionTimeout` | `pool_timeout` | Connection ke liye max wait | 3-10 sec |
| `idleTimeout` | (overflow conns return pe band) | Idle conn kab band | 10-30 min |
| `maxLifetime` | `pool_recycle` | Force-close (stale rok) | 30-60 min (DB/LB ke idle timeout se kam) |
| `validationQuery` | `pool_pre_ping=True` | Conn alive test | `SELECT 1` |
| `leakDetectionThreshold` | (khud log/monitor karo) | Leak warn if not returned | 30-60 sec |

---

## 8. Application-side vs Proxy-side pooling (PgBouncer) ⭐

### Application-side — pool har process ke andar
SQLAlchemy engine pool / HikariCP / Prisma pool — har app process ka apna.
```
App Instance 1 (4 workers × 15) ─┐
App Instance 2 (4 workers × 15) ─┼─▶ PostgreSQL (max_connections default = 100)
App Instance 3 (4 workers × 15) ─┘
```
**Math karo:** 10 instances × 4 workers × 15 = **600 possible connections**. PostgreSQL ka default `max_connections` = **100**. Load aaya, auto-scaler (lesson 04) ne 10 aur instances jode → DB bolta hai *"FATAL: sorry, too many clients already"* → saari APIs fail. Aur agar `max_connections` 1000 kar do, toh section 6 wala context-switching + har connection ki memory DB ko maar deti hai.

**Asli baat:** zyada tar app connections **zyada tar time idle** hoti hai — app beech me JSON bana raha hai, doosri API call kar raha hai, LLM ka 8 second wait kar raha hai. Toh har app process ke liye ek **dedicated** DB connection rakhna waste hai.

### Proxy-side — PgBouncer beech me
**PgBouncer** = PostgreSQL ke liye ek halka **reverse proxy** (lesson 06 — bas HTTP ki jagah Postgres protocol bolta hai). App usse aise connect karta hai jaise woh hi Postgres ho (usually port 6432). PgBouncer andar sirf thodi si **asli** DB connections rakhta hai.
```
50 app instances (1000 app conns, mostly idle) → PgBouncer → 50-100 actual DB conns
```

**Connection multiplexing kaise:** jab ek app connection **transaction shuru** karti hai, PgBouncer use ek asli DB connection de deta hai; **transaction khatam (COMMIT)** hote hi woh asli connection kisi aur app connection ko de deta hai. App ko lagta hai uski apni connection hai; asal me 1000 app connections 50 real connections ko baari-baari share kar rahi hai.

**PgBouncer ke 3 modes:**
| Mode | Asli connection kab tak milti hai | Note |
|---|---|---|
| Session | Jab tak app connection khuli hai | Safe, but fayda kam |
| **Transaction** ⭐ | Sirf ek transaction ke liye | Sabse common, sabse zyada fayda |
| Statement | Sirf ek statement ke liye | Rare, multi-statement transactions nahi chalte |

> ⚠️ **Transaction mode ka catch:** agli transaction **kisi aur asli connection** pe ja sakti hai. Isliye "session-level" cheezein toot sakti hai: `SET` variables, advisory locks, `LISTEN/NOTIFY`, temp tables, aur (purane versions me) prepared statements — asyncpg users ko aksar statement cache band karna padta tha. Naye PgBouncer versions prepared statements support karte hai, phir bhi apne version ke docs check karo.

| Aspect | Application-side (SQLAlchemy/HikariCP) | Proxy-side (PgBouncer) |
|---|---|---|
| Kaha | Har app process ke andar | Alag process, app aur DB ke beech |
| Total DB conns | instances × workers × pool size | Shared, fewer |
| Use case | Kam, fixed instances | Bahut instances / auto-scaling / serverless |
| Complexity | None (library) | Extra infra (ek aur cheez jo down ho sakti hai) |

Real life me **dono saath** chalte hai: app me chhota pool (borrow fast ho) + PgBouncer (total DB connections bound rahe). Managed versions: **AWS RDS Proxy**, Supabase Supavisor, Neon pooler.

> **AI agents ke liye important:** LLM call (5-30s) ke dauraan **DB session/transaction khula mat rakho**. Pehle DB se data padho, session band karo, phir LLM call, phir naya session me save. Warna har chalta agent ek connection pakad ke baitha rahega aur 15 agents me pool khatam.

---

## 9. Connection Leaks ⭐

**Connection leak** = code connection borrow karta hai but kabhi **wapas pool me nahi deta**. Pool dheere-dheere khaali hota hai → phir har request `pool_timeout` tak wait karke fail hoti hai.

### Kaisa dikhta hai (symptom)
App deploy ke baad ghanton theek chalta hai, phir achanak **saari DB wali APIs ~10-30s latak ke fail** hone lagti hai, aur restart karte hi phir theek. SQLAlchemy ka error:
```
QueuePool limit of size 5 overflow 10 reached, connection timed out, timeout 30
```
Postgres pe dekho: `SELECT state, count(*) FROM pg_stat_activity GROUP BY state;` — bahut saari **`idle in transaction`** connections = kisi ne `BEGIN` kiya aur kabhi commit/rollback/close nahi kiya (aur ye locks bhi pakad ke baithi hoti hai).

### Common causes (FastAPI me)
```python
# ❌ LEAK: exception pe close kabhi nahi chalta
@app.get("/users/{id}")
def get_user(id: int):
    db = SessionLocal()
    user = db.query(User).get(id)       # connection borrow + BEGIN ho gaya
    if not user:
        raise HTTPException(404)        # yahan se nikal gaye → db.close() kabhi nahi
    db.close()
    return user
# Garbage collector shayad kabhi wapas kare — bharose ke laayak nahi.
```
- `try/finally` ya `with` ke bina session/connection kholna (upar jaisa).
- **Lambi transaction**: connection pakad ke LLM / external API / file upload ka wait (technically leak nahi, but effect same — pool khaali).
- **Background task** me request wala `db` use karna — response jaane ke baad `get_db` ne session band kar diya, ya task ne naya khola aur band karna bhool gaya.
- Async code me `await` ke beech exception aur cleanup na hona.

### Prevention
- **Hamesha `Depends(get_db)` (yield + finally)** ya `with SessionLocal() as db:` / `async with` — exception me bhi close.
- Session **chhoti rakho**: padho → band → slow kaam → naya session me likho.
- `pool_pre_ping` + `pool_recycle` (mari/purani connections ke liye).
- Postgres pe `idle_in_transaction_session_timeout` (e.g. 60s) — bhooli hui transactions ko DB khud maar de.
- Monitoring: pool ke **checked-out vs idle** counts, aur `pg_stat_activity`.
- Java me HikariCP `leakDetectionThreshold` stack trace ke saath batata hai kis line ne connection liya aur wapas nahi kiya.

> **Frontend analogy:** ye ek memory leak jaisa hai — resource acquire kiya but release nahi kiya (jaise `addEventListener` bina `removeEventListener`, ya `useEffect` me subscription bina cleanup). Dheere-dheere resources khatam.

---

## 10. Serverless ka special problem ⭐

**Kyu problem:** AWS Lambda me har **concurrent invocation** ek alag chhota container/process hai, apni memory ke saath. Unke beech **koi shared pool nahi** hota.
```
Normal server:  1 process, pool of 15 → 1000 requests baari-baari 15 connections pe
Lambda:         1000 concurrent invocations = 1000 alag processes = 1000 connections 💥
```
Aur upar se:
- **Handler ke andar** connection banaya → har invocation pe naya connect (30ms+ har baar).
- **Handler ke bahar** banaya → warm invocations me reuse ✅, but phir bhi **1 connection per container**. Aur idle container "freeze" ho jaata hai — uski connection DB pe khuli latki rehti hai jab tak DB timeout na kare (zombie connections).
- Traffic spike → Lambda 0 se 1000 containers turant → DB ka `max_connections` ek second me khatam.

| Solution | Kaise |
|---|---|
| **RDS Proxy** | AWS ka managed PgBouncer jaisa: saare Lambdas ↔ proxy ↔ thodi asli DB connections. IAM auth + DB failover bhi sambhalta hai |
| **PgBouncer / Supavisor** | External pooler saare Lambdas ke across shared |
| Connection reuse | Connection handler ke **bahar** init karo, pool size 1 (SQLAlchemy `NullPool` ya `pool_size=1`) — har container ek hi rakhe |
| Concurrency cap | Lambda ki reserved concurrency limit = connections ki upar ki limit |
| **HTTP-based DB access** | DynamoDB, ya Postgres ke HTTP drivers / Data APIs (Neon serverless driver, Aurora Data API) — persistent connection hi nahi |

> **Interesting:** DynamoDB (Topic 2 lesson 07) HTTP-based hai — har call ek HTTPS request (lesson 01-02 ka keep-alive wahan kaam aata hai), DB-side persistent "logged-in session" nahi. Isliye connection pooling ki problem hi nahi. Ye ek design trade-off hai (har call ka HTTP overhead vs no connection management).

---

## Quick Self-Check
1. Connection pooling kya hai — office badge analogy?
2. Yahan "connection" kis ke beech hai? DB side pe har connection ka kya kharch hai? Transaction ek connection pe kyu?
3. `create_engine` pe kitni connections banti hai? Pehli connection kab banti hai? 4 uvicorn workers = kitne pools?
4. `db.add()`, `db.query()`, `db.commit()`, `db.close()` — kis pe DB se baat hoti hai aur connection kab borrow/return hota hai? `cursor()` kya hai?
5. Connection banane ke steps (kya mehenga)? Lesson 01 ke kis concept se relate?
6. Pool kaise kaam karta hai — borrow/return/wait flow? `pool_size` vs `max_overflow`?
7. CPU core aur disk spindle kya hai? Pool sizing formula? "200 connections 20 se slow" kyu?
8. 10 instances × 4 workers × 15 pool — Postgres default 100 pe kya hoga? PgBouncer transaction mode kaise fix karta hai, aur uska catch kya hai?
9. Connection leak kaisa dikhta hai (symptom + error)? FastAPI me 2 common causes + prevention? `idle in transaction` kya hai?
10. LLM call ke time DB session khula kyu nahi rakhna?
11. Serverless me connection problem kyu? RDS Proxy + DynamoDB kaise avoid karte hai?

---

## Flow — FastAPI request se DB tak, pool ke saath
1. **App start:** har uvicorn worker `create_engine` chalata hai → ek **khaali pool** banta hai (connections abhi zero).
2. **HTTP request aayi** → `Depends(get_db)` ek Session banata hai (connection abhi nahi). `db.add()` sirf memory me.
3. **Pehli query / flush** pe session pool se **connection borrow** karta hai: idle mili toh 0.1ms me; nahi toh nayi (DNS → TCP → TLS → login, ~30ms); pool full toh `pool_timeout` tak wait. (PgBouncer ho toh ye connection PgBouncer tak jaati hai, woh transaction ke liye ek asli DB connection deta hai.)
4. **SQL us socket pe jaata hai** → DB ka backend process apne CPU cores (aur disk) pe query chalata hai → result wapas. `db.commit()` → `COMMIT`.
5. **`finally: db.close()`** → connection **pool me wapas** (TCP khula rehta hai), agli request turant reuse karti hai. Close bhool gaye = leak → pool khaali → sab requests timeout.

---
**Next:** `10-Life-Of-A-Request-Playbook.md` — sab kuch ek request ke safar me tie karo.