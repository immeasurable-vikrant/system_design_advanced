# 05 — Fencing Tokens

> **Is lesson me:** Fencing token kya hai aur kaise ek expired lock/leader ko data corrupt karne se rokta hai — GC-pause problem, monotonic token, storage-layer enforcement, implementation (ZooKeeper/Redis/DB), aur fencing vs CAS. Ye leader election (lesson 04) ki safety net hai. Topic 2 lesson 12 aur Topic 4 lesson 05 (Kafka epoch) me chhua tha; yahan full.

Prerequisites: `04-Leader-Election-And-Consensus.md`

---

## 1. Fencing token kya hai

**Fencing token** = ek **monotonically increasing number** jo har lock acquisition ke saath issue hota hai. Storage layer (DB, file system, queue) **kisi bhi write ko reject** karta hai jo already-seen highest token se **chhota** token laaye. Ye stale lock holders ko data corrupt karne se rokta hai — jab unka lock expire ho chuka aur kisi aur ko re-issue ho chuka ho.

**Analogy:** ek relay race jaha baton pe number hai. Runner 1 ko baton #5 mila. Wo lad-khadaake slow ho gaya, toh organizer Runner 2 ko baton #6 de deta hai. Agar Runner 1 eventually finish line pe #5 ke saath pahunche, judges reject kar dete hai — #6 already cross kar chuka. Baton number = fencing token; judges = storage layer jo ordering enforce karta hai.

---

## 2. The problem — expired lock holders ⭐

Distributed locks ka ek TTL hota hai (deadlock avoid karne ko — lesson 04). Agar lock holder **pause** ho jaye (GC pause, network delay, CPU saturation) TTL se **zyada** der, lock **expire** hoke doosre client ko mil jaata hai.

```
Client 1: acquire lock → granted (TTL=30s)
Client 1: LONG GC PAUSE (35s) ...
  T=30s: lock expire ho gaya
Client 2: acquire lock → granted (naya holder)
Client 2: Write X=200
Client 1: GC pause khatam, "mera lock abhi bhi hai" (galat!) → Write X=100
  → DATA CORRUPTED! Client 1 ne Client 2 ka valid write overwrite kiya
```

> **GC pause kya hai:** garbage-collected languages (Java, Go, JS) me runtime kabhi-kabhi execution **pause** kar deta hai memory clean karne ko — kuch ms se kai seconds tak. Is pause me program "freeze" hota hai, but crash nahi. Lock TTL is beech expire ho sakta hai, aur program ko pata bhi nahi.

**Root cause (lesson 01):** partial failure — Client 1 "alive but paused" tha, lock service ne use "dead" maan liya. Lock ne **false sense of mutual exclusion** diya.

---

## 3. Fencing solves it — safety storage layer pe move karo

Har lock grant ke saath ek **incrementing token** do. Storage har write ke saath token check kare:
```
Client 1: lock granted with token=33
Client 1: LONG GC PAUSE ...
  lock expire
Client 2: lock granted with token=34
Client 2: Write X=200 with token=34 → storage: "34 >= max_seen, ACCEPT. max=34"
Client 1: (pause khatam) Write X=100 with token=33
  → storage: "33 < max_seen 34, REJECT!" ✅
```

**Key insight:** safety guarantee **lock service se storage layer pe** move ho gaya. Lock service GC pauses nahi rok sakta — but storage layer **har write dekhta hai** aur ordering enforce kar sakta hai. Ye distributed systems ka ek beautiful pattern hai: *jaha guarantee nahi de sakte (lock service), wahan se hata ke jaha de sakte hai (storage) le jao.*

---

## 4. How it works (requirements)

```
Client lock acquire kare → lock service token N deta hai
Client computation kare → storage ko write kare token N ke saath
Storage: N >= max_seen_token?
  HAAN → accept, max_seen = N
  NAHI → REJECT (stale token)
```
Requirements:
1. Lock service **monotonically increasing** tokens issue kare (kabhi reuse/decrease nahi)
2. Storage layer **max token per resource** track kare
3. Storage **stale-token writes reject** kare
4. Client **har write ke saath token** include kare

---

## 5. Implementation patterns

### ZooKeeper
Sequential ephemeral znodes naturally fencing tokens dete hai:
- Sequential znode number = monotonic token
- `zxid` (transaction ID) = global ordering

### Redis (Redlock) — token ALAG se chahiye
Redis locks **natively fencing nahi dete** (Martin Kleppmann ki famous critique). Alag se add karo:
```python
LOCK = redis.lock("resource-X")
TOKEN = redis.incr("fence:resource-X")   # separate atomic counter
# ... kaam ...
storage.write(data, fence_token=TOKEN)   # storage token validate kare
```

### Database (advisory lock + sequence)
```sql
SELECT pg_advisory_lock(resource_id);
INSERT INTO fence_tokens (resource_id, token)
  VALUES ('job-123', nextval('fence_seq')) RETURNING token;
-- token ke saath kaam
```

### Storage enforcement (yahi asli power hai)
Token tabhi meaningful hai jab storage use enforce kare:
| Storage | Kaise enforce |
|---|---|
| PostgreSQL | `UPDATE ... SET ... WHERE fence_token < :new_token` (conditional) |
| DynamoDB | `ConditionExpression: "fence_token < :new"` |
| Kafka | Producer epoch (broker zombie producers reject kare) |

> **Recap (Topic 4 lesson 05):** Kafka ka "producer epoch" ek fencing token hai — zombie producer (purana, hung) jab commit kare, broker uska epoch dekh ke reject karta hai. Wahi mechanism.

---

## 6. Fencing vs CAS (Compare-and-Swap)

Dono stale writes rokte hai, but alag:
- **CAS** — **current value/version** compare kare ("update if version = 5"). Kisi bhi concurrency ke liye.
- **Fencing token** — **lock-issued number** compare kare ("reject if token < max"). Specifically "lock expired but holder ko pata nahi" problem ke liye.

> **Recap (Topic 2 lesson 09):** optimistic concurrency (compare-and-set on version) — wo CAS hai. Fencing token lock-context ke liye specialized hai. Dono combine bhi kar sakte ho.

**Raft term = fencing token:** yaad hai Raft ka term number (lesson 04)? Jab naya leader term T+1 se elect hota hai, followers purane leader (term T) ke messages reject karte hai. **Term ek fencing token hai** — consensus protocol me built-in.

---

## 7. Fencing kya nahi rok sakta (honest limitation)

Fencing tokens **cooperating storage** pe stale **writes** rokte hai. But:
- Stale **reads** nahi rokte
- **Non-cooperating external systems** pe actions nahi rokte — agar stale holder ek email bhej de ya third-party API call kare, fencing undo nahi kar sakta.

Un cases ke liye tumhe **idempotency keys** (Topic 4 lesson 05) + compensating transactions (Topic 2 saga) chahiye, fencing ke saath.

---

## Quick Self-Check
1. Fencing token kya hai — relay-race baton analogy?
2. Expired-lock-holder problem — GC pause se data corruption kaise (Client 1/2)?
3. Fencing "safety ko storage layer pe move" — ye kyu clever hai (lock service GC nahi rok sakta)?
4. Fencing ke 4 requirements?
5. Redis (Redlock) ko fencing ke liye extra kya chahiye, aur kyu?
6. Storage enforcement kyu essential (token = "just a number" bina iske)?
7. Fencing vs CAS — difference? Raft term fencing token kaise hai?
8. Fencing kya NAHI rok sakta (reads, external actions)?

---
**Next:** `06-Consistent-Hashing.md` — data ko nodes me distribute karna (recap + depth).