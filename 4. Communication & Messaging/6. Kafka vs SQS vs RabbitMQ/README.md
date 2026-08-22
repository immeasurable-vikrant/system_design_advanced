# 06 — Kafka vs SQS vs RabbitMQ (Choosing a Broker)

> **Is lesson me:** Teen sabse common message brokers ka comparison, unke core concepts (Kafka partitions/offsets, SQS visibility timeout), aur kaunsa kab. Ye lesson tumhe interview me "kaunsa message queue use karoge aur kyu" ka confident answer deta hai.

Prerequisites: `05-Delivery-Guarantees-And-Idempotency.md`

---

## 1. Teen alag philosophies

- **Kafka** — **log-based** (append-only). Messages ek durable log me rehte hai, consumers apni position (offset) se padhte hai. High throughput, replay possible.
- **SQS (AWS)** — **queue** (delete after read). Simple managed queue, message padhne ke baad gone. Zero-ops.
- **RabbitMQ** — **queue + exchange routing**. Complex routing logic, RPC patterns.

> **Recap (Topic 2 lesson 10 — WAL):** yaad hai "Kafka ka log ek WAL hai"? Kafka messages ko append-only log ki tarah store karta hai (jaise database WAL), isliye consumers purane messages **replay** kar sakte hai. SQS me message padha = gaya (no replay). Ye fundamental difference hai.

---

## 2. Head-to-head comparison

| Feature | Kafka | SQS | RabbitMQ |
|---|---|---|---|
| Model | Log-based (append-only) | Queue (delete after read) | Queue + exchange routing |
| Ordering | Within partition | No (Standard) / Yes (FIFO) | Within queue |
| Retention | Configurable (days/weeks/forever) | 4 days (max 14) | Until consumed |
| **Replay** | **Yes** (re-read old messages) | No (once read, gone) | No |
| Throughput | **1M+ msg/sec** | ~3000 msg/sec per queue | ~20K msg/sec |
| Scaling | Add partitions | Unlimited (managed) | Add queues/nodes |
| Consumer groups | **Built-in** | Not native | Not native |
| Managed option | Confluent, AWS MSK | AWS SQS (fully managed) | CloudAMQP, Amazon MQ |
| Best for | Event streaming, CDC, high throughput | Simple async tasks | Complex routing, RPC |

---

## 3. Kab kaunsa choose karo

### Kafka choose karo jab:
- **High throughput** (100K+ events/sec)
- **Replay** chahiye (historical data re-process)
- Multiple consumers ko same events chahiye (**consumer groups** — lesson 04)
- Event sourcing / CDC pipeline (lesson 08)
- Ordering matter kare per entity (partition key — lesson 05)

### SQS choose karo jab:
- Simple job/task queue
- **Fully managed, zero ops** chahiye
- Replay ki zarurat nahi
- Low-medium throughput theek hai
- AWS ecosystem me ho

### RabbitMQ choose karo jab:
- **Complex routing** (header/topic se alag queues me route)
- RPC pattern (request-reply)
- Message priority chahiye
- Smaller scale

**Interview default:** *"High-throughput event streaming, replay, ya multiple independent consumers → Kafka. Simple managed async tasks, AWS pe → SQS. Complex routing ya RPC → RabbitMQ. Zyadatar modern systems Kafka use karte hai event backbone ke liye."*

---

## 4. Kafka key concepts

### Partitions
Ek topic multiple **partitions** me split hota hai. Har partition ek **ordered, immutable log** hai.
```
Topic "payment_events" (3 partitions):
  Partition 0: [msg_0, msg_3, msg_6, msg_9, ...]
  Partition 1: [msg_1, msg_4, msg_7, msg_10, ...]
  Partition 2: [msg_2, msg_5, msg_8, msg_11, ...]

Har partition ek consumer (group ke andar) consume karta hai.
Zyada partitions = zyada parallelism.
```
**Rule of thumb:** partitions ≥ consumers in a group. 10 consumers chahiye toh kam se kam 10 partitions.

> **Connect (Topic 2 lesson 13 — sharding):** partitions basically sharding hai messages ki — data ko independent chunks me baant ke parallelism dete hai. Partition key = shard key.

### Offsets
Har message ka partition me ek sequential **offset** (position number) hota hai. Consumer track karta hai konsa offset process kiya.
```
Partition 0: [offset_0, offset_1, offset_2, offset_3, offset_4]
                                    ▲
                    consumer yahan tak process kiya (committed offset = 3)
```
Consumer restart ho → committed offset se resume (agar sahi commit kiya toh no duplicate — lesson 05).

**Frontend analogy:** offset ek bookmark hai — jahan tak padha wahan se continue. Kafka ka log ek book hai jise multiple readers apne-apne bookmark se padhte hai (consumer groups).

---

## 5. SQS key concept — Visibility Timeout

Jab consumer ek message padhta hai, wo doosre consumers ke liye **"invisible"** ho jaata hai ek timeout ke liye. Agar consumer us time me delete na kare, message **wapas visible** ho jaata hai (re-deliver).
```
Message queue me → Consumer A padhta hai → message 30s ke liye invisible
  - Consumer A success → message DELETE → done ✅
  - Consumer A crash → 30s timeout → message wapas visible → Consumer B uthata hai
```
Ye SQS ka at-least-once + crash-recovery mechanism hai. Agar processing 30s se zyada le, timeout badhao (warna message re-deliver ho jayega jab consumer abhi kaam kar raha ho).

**Frontend analogy:** ek shared to-do item jise koi "claim" karta hai — 30 min tak locked (doosre nahi dekhte), kaam na hua toh wapas available. Optimistic claiming.

---

## Quick Self-Check
1. Kafka (log) vs SQS (queue) ka fundamental difference — replay ke context me?
2. Ye Topic 2 ke WAL concept se kaise connect hota hai?
3. Kafka, SQS, RabbitMQ — ek-ek line me best-for?
4. High throughput + replay + multiple consumers → kaunsa? Simple managed AWS task → kaunsa?
5. Kafka partition kya hai, aur partitions ≥ consumers rule kyu?
6. Offset kya hai (bookmark analogy)? Consumer restart pe kaise resume?
7. SQS visibility timeout kaise crash-recovery deta hai?

---
**Next:** `07-Retries-Backoff-And-DLQ.md` — jab messages fail hote rahe.