# 15 — Object Storage (S3) — Files, Images, Videos

> **Is lesson me:** Object storage kya hai aur file/block storage se kaise alag, S3 ki internal architecture, presigned URLs (backend ke bina direct upload/download), storage tiers + lifecycle, multipart upload, durability (11 nines), aur kab use karna/nahi. Ye relational databases ke bahar ka essential storage layer hai.

Prerequisites: `01-What-Is-A-Database.md` (bas basics); helpful: API/HTTP samajh

---

## 1. Object storage kya hai

**Object storage** = ek **flat-namespace** storage system jo **unstructured data** (files, images, videos, backups) ko discrete **objects** ki tarah store karta hai. Har object me:
- **Key** (unique identifier — jaise `images/user123/photo.jpg`)
- **Data** (actual bytes — the blob)
- **Metadata** (labels — content-type, size, custom tags)

HTTP API se access hota hai (GET/PUT), aur **exabytes** tak scale karta hai.

**Analogy:** Ek massive warehouse with numbered lockers. Har locker ka unique number (key), andar ka saman (data), aur front pe label (metadata). Koi folders/aisles nahi — bas locker number pata ho, seedha wahan jao. Warehouse infinitely expand ho sakta hai (aur lockers add karke).

> **"Unstructured data" kya hai:** jise rows/columns me neatly fit nahi kar sakte — ek 4K video, ek PDF, ek profile photo. Database ke liye ye awkward + expensive hai; object storage inke liye purpose-built hai.

---

## 2. Object vs File vs Block storage

Teen fundamental storage types:

| Aspect | Object Storage | File Storage | Block Storage |
|---|---|---|---|
| Access | HTTP API (GET/PUT) | File system (NFS/SMB) | Raw blocks (iSCSI) |
| Namespace | Flat (key → value) | Hierarchical (folders) | None (raw sectors) |
| Scale | Exabytes (unlimited) | Petabytes | Terabytes |
| Modification | Replace whole object | Edit in place | Edit any block |
| Latency | Higher, high throughput | Medium | Lowest |
| Use case | Images, videos, backups | Shared files, home dirs | **Databases**, VMs |
| Examples | S3, GCS, Azure Blob | EFS, NFS | EBS, SAN |

> **Interesting connection:** Tumhara Postgres database actually **block storage** (jaise AWS EBS) pe chalta hai — kyunki DB ko low-latency random reads/writes chahiye (index lookups, row updates — lesson 06). Object storage DB ke liye galat hai (high latency, whole-object replace only). But images/videos ke liye object storage perfect hai.

**Ek line me:** block = database-worthy raw disk; file = shared folders; object = internet-scale blobs via HTTP.

---

## 3. S3 architecture (andar kya hai)

```
Client
  │  PUT /bucket/key
  ▼
API Gateway          ← HTTP requests, authentication, routing
  │
  ▼
Metadata Service     ← key → physical location mapping, object metadata
  │                     (stored in a Metadata DB)
  ▼
Storage Service      ← actual data across distributed nodes manage
  │
  ├──▶ Data Node 1 (primary)
  ├──▶ Data Node 2 (replica)     ← typically 3 copies across
  └──▶ Data Node 3 (replica)        Availability Zones (AZs)
```

Components:
- **API Gateway:** HTTP requests handle, auth, routing
- **Metadata Service:** keys ko physical storage locations pe map karta hai
- **Storage Service:** actual data distributed nodes pe manage
- **Data Nodes:** object data with replication (3 copies, alag AZs — physically separate data centers)

---

## 4. Presigned URLs (backend ke bina direct upload/download)

**Problem:** Agar har file upload tumhare backend server se hoke jaye — 50MB video × 10,000 concurrent = 500GB bytes tumhare servers se guzar rahe. Massive bandwidth/CPU/memory waste.

**Solution:** **Presigned URL** — tumhara backend ek temporary, signed URL generate karta hai jisse client **directly S3 pe** upload/download kare, bytes backend ko touch kiye bina.

```
Client                Backend              S3
  │  1. "photo.jpg upload karna hai"       │
  │─────────────────▶│                     │
  │                  │ 2. presigned PUT URL │
  │                  │    generate (15min)  │
  │  3. presigned URL │                     │
  │◀─────────────────│                     │
  │  4. PUT directly to S3 with that URL    │
  │─────────────────────────────────────▶│
  │  5. 200 OK uploaded                     │
  │◀─────────────────────────────────────│
  │  6. "done"        │                     │
  │─────────────────▶│ 7. metadata DB update│
```

**Fayde:**
- Backend bytes handle nahi karta (bandwidth/CPU bachta hai)
- Client seedha S3 pe (faster, fewer hops)
- Time-limited (5-60 min expiry)
- Specific key + operation (PUT ya GET) tak scoped
- Storage credentials client ko expose nahi karne padte

**Frontend analogy:** Ye bilkul "signed upload URL" pattern hai jo tumne shayad Cloudinary/Firebase Storage me dekha ho — client ek short-lived permission-slip URL leke direct upload karta hai.

---

## 5. Storage tiers + lifecycle (cost optimization)

Sab data equally "hot" nahi hota. Tiers different access-frequency ke liye, different cost pe:

| Tier | Access | Latency | Cost/GB/mo | Use |
|---|---|---|---|---|
| Standard | Frequent | ms | ~$0.023 | Active app assets |
| Infrequent Access (IA) | Monthly | ms | ~$0.0125 | Backups, old logs |
| Glacier Instant | Quarterly | ms | ~$0.004 | Archives (instant) |
| Glacier Flexible | Yearly | 3-5 hrs | ~$0.0036 | Long-term archives |
| Glacier Deep Archive | Rarely | 12-48 hrs | ~$0.00099 | Compliance, legal hold |

**Lifecycle rules** automatically objects ko tiers ke beech move karti hai:
```
Day 0:    Standard
Day 30:   → IA
Day 90:   → Glacier Instant
Day 365:  → Deep Archive
Day 2555: Delete (7-year retention)
```
Isse purana data automatically sasta hota jaata hai bina manual kaam ke. (~$0.023 → ~$0.001 per GB = 23x cheaper for cold data.)

---

## 6. Multipart upload (bade files reliably)

Bade files (>100MB) ke liye:
```
1. Initiate multipart upload → upload ID milta hai
2. File ko parts me split (5MB-5GB each)
3. Parts PARALLEL upload karo
4. Complete → S3 parts ko assemble karke ek object banata hai
```
**Fayde:**
- Parallel uploads (faster)
- Ek part fail → sirf wahi part retry (poora file nahi)
- Stream upload (file fully generate hone se pehle shuru)

Mobile/unstable connection pe chhote chunks (5MB) + progress track + resume from last successful part. Incomplete multipart uploads ko 24hr baad abort karne ka lifecycle rule (storage cost bachao).

**Frontend analogy:** Bilkul chunked file upload jo tum resumable uploaders (tus, Uppy) me dekhte ho — file ko pieces me toDo, parallel bhejo, fail pe sirf ek piece retry.

---

## 7. Durability & consistency

**11 nines durability (99.999999999%):** S3 har object ko minimum **3 AZs** me redundantly store karta hai, continuous checksums se integrity monitor karta hai, corruption detect hote hi healthy copy se auto-repair. Matlab: 10 billion objects me se ~1 object 10,000 saal me lose. Practically unbreakable.

**Consistency (Dec 2020 ke baad):** S3 ab **strong read-after-write consistency** deta hai:
```
PUT (new object)     → strongly consistent (turant read kar sakte ho)
PUT (overwrite)      → strongly consistent
DELETE               → strongly consistent
LIST                 → strongly consistent
```
Matlab: successful PUT ke baad, koi bhi GET latest version dega. (Pehle eventual consistency thi — ab wo headache gaya.)

> **CAP connection (lesson 14):** S3 ab strong consistency deta hai read-after-write ke liye — ye ek deliberate design choice hai jo 2020 me possible hui. Distributed storage me strong consistency dena engineering-wise non-trivial hai.

---

## 8. Real systems + when to use

| System | Object storage kaise use |
|---|---|
| Instagram | Photos/videos store; CDN serve karta hai |
| Netflix | Encoded video segments (HLS/DASH) |
| Pastebin | Paste content (key = paste ID) |
| Google Docs | Document revisions + attachments |
| Data Lake | Raw analytics data landing zone |
| ML Pipeline | Training data + model artifacts |
| Backup/DR | Database dumps, point-in-time backups |

**✅ Use object storage when:**
- Unstructured data (images, videos, docs, logs)
- Write-once-read-many (WORM pattern)
- Virtually unlimited storage chahiye
- HTTP access kaafi hai (no random byte-edits)
- Built-in durability chahiye (11 nines)
- CDN ke peeche static content serve karna

**❌ Don't use when:**
- Low-latency random writes chahiye → block storage/database
- Files frequently in-place modify hote hai → file storage/database
- POSIX file system semantics chahiye → EFS/NFS
- Sub-ms access chahiye → in-memory cache (Redis)
- Chhote structured records → database

**The pattern to remember:** *"Metadata database me, blob object storage me."* E.g., Instagram: photo ka actual bytes → S3; photo ka `{id, user_id, caption, s3_key, created_at}` → Postgres/DynamoDB. Database chhota + fast rehta hai, bade files S3 me sste padte hai, CDN se fast serve hote hai.

---

## Quick Self-Check
1. Object storage kya hai — key/data/metadata, aur "flat namespace" ka matlab?
2. Object vs file vs block — teeno kab? Database kaunsa use karta hai aur kyu?
3. Presigned URL kya problem solve karta hai, aur flow kya hai?
4. Storage tiers + lifecycle rules cost kaise bachate hai?
5. Multipart upload ke 3 fayde?
6. S3 11 nines durability kaise achieve karta hai?
7. "Metadata DB me, blob object storage me" pattern — Instagram example se?

---
**Next:** `16-Putting-It-All-Together-Design-Playbook.md` — sab kuch combine, ek real system design karke.