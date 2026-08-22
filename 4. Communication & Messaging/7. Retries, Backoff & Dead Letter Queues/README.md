# 07 — Retries, Backoff & Dead Letter Queues

> **Is lesson me:** Jab message processing fail ho — retry kaise karo (exponential backoff + jitter), poison messages kya hai, aur **Dead Letter Queue (DLQ)** jo failed messages ko isolate karti hai. Ye messaging systems ko resilient banata hai. Retry/backoff ek gap tha jo yahan fill kar raha hu.

Prerequisites: `05-Delivery-Guarantees-And-Idempotency.md`, `06-Kafka-vs-SQS-vs-RabbitMQ.md`

---

## 1. Failures inevitable hai — retry pehla defense

Message processing fail ho sakti hai — downstream service down, network glitch, temporary overload. Pehla response: **retry.** But blindly retry karna khatarnak hai.

### Naive retry ka problem
```
Service down → retry immediately → phir fail → retry immediately → ...
1000 consumers ye kare → down service pe hammering → aur bura (recover hi nahi paata)
```
Ye "retry storm" hai — failing service ko aur maar deta hai.

### Solution — Exponential Backoff ⭐
Har retry ke beech **badhta hua wait** karo:
```
Attempt 1 fail → wait 1s  → retry
Attempt 2 fail → wait 2s  → retry
Attempt 3 fail → wait 4s  → retry
Attempt 4 fail → wait 8s  → retry
(wait doubles: 1, 2, 4, 8, 16... — "exponential")
```
Isse failing service ko saans milti hai recover karne ke liye, aur load kam hota hai.

### Jitter — the crucial addition
Agar 1000 consumers exactly same backoff use kare, wo sab **ek saath** retry karenge (synchronized waves) → phir se spike. **Jitter** = backoff me thoda **randomness** add karo:
```
Bina jitter: sab consumers t=1s, 2s, 4s pe retry (synchronized spikes)
Jitter ke saath: har consumer random(0, backoff) wait kare → retries spread out
```
```python
wait = random(0, min(cap, base * 2**attempt))  # exponential backoff + jitter
```

> **Recap (Topic 2 lesson 09):** yaad hai SSI retry loop me "jitteredBackoff(attempt)"? Wahi ye hai. Aur Topic 1 (lesson 03) me WebSocket reconnection bhi exponential backoff use karta hai. **Ye pattern har jagah hai** — koi bhi retry hamesha backoff + jitter ke saath.

**Frontend analogy:** tumne API retry logic me `setTimeout(retry, delay * 2**attempt)` likha hoga — wahi exponential backoff. Jitter add karke thundering-herd rokte hai.

---

## 2. Retry Queue vs poison messages

Kuch failures **transient** hai (downstream temporarily down — retry se theek ho jayenge). Kuch failures **permanent** hai:

**Poison message** = ek message jo **kabhi successfully process nahi hoga**, chahe kitni baar retry karo:
| Cause | Example |
|---|---|
| Schema mismatch | Producer v2 schema bheje, consumer v1 expect kare |
| Invalid data | Null field jaha non-null chahiye, malformed JSON |
| Missing dependency | Message ek deleted entity ko reference kare |
| Consumer bug | Unhandled edge case exception de |
| Expired context | Message itna purana ki meaningless |

Poison message ko infinite retry karna = compute waste + **head-of-line blocking** (Topic 1 lesson 01 — ek atka message peeche walon ko rokta hai).

**Retry queue vs DLQ:**
- **Retry queue** = messages jo **eventually succeed karenge** (transient failure) — automatically delayed + re-attempted.
- **DLQ** = messages jinke **saare retries khatam** ho gaye aur likely intervention ke bina succeed nahi honge — humans ko escalate.

---

## 3. Dead Letter Queue (DLQ) ⭐

**Dead Letter Queue** = ek special queue jahan wo messages jaate hai jo **N attempts ke baad bhi process nahi hue**. Ye "poison" messages ko isolate karti hai taaki main pipeline block na ho, aur unhe investigation/reprocessing ke liye preserve karti hai.

**Analogy:** post office ki sorting machine. Zyadatar letters sort ho jaate hai. Kuch damaged/invalid letters machine process nahi kar paati — poori line jam karne ke bajaye, unhe ek **special bin** me daal deti hai jise human baad me dekhta hai. DLQ wahi special bin hai.

```
Producer → Main Queue → Consumer
                          │ fail 1 → retry
                          │ fail 2 → retry
                          │ fail 3 → retry
                          │ max retries exceeded
                          ▼
                    Dead Letter Queue → alert ops → manual review
                    (main queue chalti rehti hai baaki messages ke liye)
```

### DLQ ke bina vs saath
```
Bina DLQ:  poison message queue block kare (HOL), consumer infinitely retry (compute waste),
           valid messages peeche atke, koi visibility nahi kya fail ho raha

DLQ ke saath: poison message N attempts ke baad isolate, main queue chalti rehti,
              ops ko DLQ-depth alert, failed messages debug/reprocess ke liye preserved
```

---

## 4. DLQ implementation

### SQS — native support (redrive policy)
```
Main Queue (orders-queue)
  Redrive Policy: maxReceiveCount = 3
  → 3 failed receives ke baad → orders-dlq (retention 14 days)
  → CloudWatch alarm: DLQ depth > 0 → SNS alert to ops
```
SQS 2021 se **DLQ redrive** support karta hai — messages ko DLQ se wapas source queue me bhej sakte ho reprocessing ke liye (no custom code).

### Kafka — no native DLQ, pattern se banate hai
Kafka me built-in DLQ nahi — consumer level pe implement:
```
Consumer message process kare (try-catch me):
  N failures ke baad:
    1. Failed message ko "orders-topic.DLT" (dead letter topic) pe produce karo (error metadata ke saath)
    2. Original topic pe offset commit karo (taaki dobara retry na ho)
    3. Agle message pe badho
Ek alag monitoring consumer DLT ko padhe alerting ke liye.
```

**Tiered retry pattern (Uber-style):**
```
orders-topic → retry-1 (1min delay) → retry-2 (10min delay) → DLT (human review)
(graduated delays — pehle jaldi retry, phir zyada wait, phir give up)
```

---

## 5. Monitoring & reprocessing

**Alerts (critical):**
| Metric | Threshold | Action |
|---|---|---|
| DLQ depth | > 0 | Investigate |
| DLQ growth rate | > 10/min | Likely systemic bug (not isolated poison) |
| DLQ age | oldest > 24h | Messages aging out without review |
| DLQ as % of throughput | > 1% | Consumer me significant bug |

**Reprocessing (jab fix ho jaye):**
- **Automatic replay** — known transient issue, cooldown ke baad replay
- **Manual replay** — code fix deploy ke baad ops trigger kare
- **Transform and replay** — message format fix karke re-enqueue
- **Discard with logging** — truly unprocessable, audit ke liye log karke delete

> ⚠️ **DLQ ko band-aid mat banao:** agar messages DLQ me ja rahe hai **consumer bug** ki wajah se, bug fix karo — DLQ ko "ignore failures" ke tarah mat use karo. DLQ investigation ke liye hai, sweeping-under-rug ke liye nahi.

---

## 6. Ordering + DLQ ka conflict (interview depth)

Agar strict ordering matter kare (ek user ke events order me process hone chahiye — lesson 05), toh ek message DLQ me bhejne ka matlab uske baad wale messages bhi block ho sakte hai. Solutions:
- Us partition key ke saare messages ek saath DLQ me bhejo
- Us specific key ke liye processing pause karo jab tak DLQ resolve na ho
- Per-key retry counter — N failures ke baad us key ko skip karo, baaki continue

---

## Quick Self-Check
1. Naive immediate-retry ka problem (retry storm)? Exponential backoff kaise fix karta hai?
2. Jitter kya hai aur kyu zaroori (synchronized retries)?
3. Backoff+jitter kaunse pichle topics me bhi dikha (SSI, WebSocket reconnect)?
4. Poison message kya hai — 2 examples? Retry queue vs DLQ ka difference?
5. DLQ kya karti hai (post-office bin analogy)? DLQ ke bina 2 problems?
6. SQS (redrive) vs Kafka (DLT pattern) — DLQ kaise implement hoti hai?
7. Tiered retry pattern kya hai?
8. DLQ ko "band-aid" banane ka matlab kya, aur kyu galat?

---
**Next:** `08-Reliability-Patterns.md` — back-pressure aur outbox pattern.