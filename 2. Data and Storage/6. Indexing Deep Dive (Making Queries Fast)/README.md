# 06 — Indexing Deep Dive (Making Queries Fast)
 
> **Is lesson me:** Index kya hai, kaun banata hai aur kaise "on" hota hai, full table scan andar se kaise chalta hai, **B-Tree ek 8-row table se** (node, leaf, key, pointer, hop — sab concrete), range/sort/prefix kaise, Hash index ke bucket ke andar kya hota hai, Composite index aur **left-prefix rule kahan se aata hai**, Covering index, GIN/GiST/BRIN (+ vector indexes), write overhead, **cardinality**, aur clustered vs secondary index. Ye poore performance ka core hai.
 
Prerequisites: `04-How-A-Query-Travels.md` (disk vs RAM, query planner samajh liya ho)
 
---
 
## 1. Index kya hai — aur kyu
 
**Index** = ek extra data structure (table ke bagal me, alag file) jo queries ko fast banata hai, **full table scan** avoid karke. Bina index, database har row check karta hai; index se seedha matching rows pe jump karta hai.
 
**Analogy:** Ek textbook ka index (peeche wala). "Binary Search" dhundhna hai? Bina index → har page palto. Index se → "Binary Search → page 142" dekho, seedha wahan jao.
 
```
Bina index (full table scan):
  Table: 10 million rows
  Query: SELECT * FROM users WHERE email = 'bob@mail.com'
  → saari 10M rows check karo → O(N) → SLOW (seconds)
 
Index ke saath (email pe):
  → index tree me neeche utro → O(log N) → FAST (milliseconds)
```
 
### Full table scan andar se kaise hota hai (1 line)
DB table ke **saare pages** (disk ke chunks, section 2.1) ek-ek karke padhta hai aur har page ki har row pe WHERE check karta hai:
```
for page in table_pages:          # 10M rows ≈ lakhs pages
    for row in page:
        if row.email == 'bob@mail.com': output(row)
```
Rows ek-ek nahi, **page-by-page** padhi jaati hai (ek page me kai rows) — but phir bhi poori table padhni padti hai. `EXPLAIN` me isko **`Seq Scan`** (sequential scan) likha aata hai.
 
### Index kaun banata hai? "On" kaise hota hai?
- **Tum (developer/DBA) banate ho** — SQL se:
```sql
CREATE INDEX idx_users_email ON users (email);
```
  Usually migration me (Alembic `op.create_index(...)`, SQLAlchemy `Column(..., index=True)`, Django `db_index=True`).
- **DB khud banata hai** sirf **PRIMARY KEY** aur **UNIQUE** constraints ke liye (uniqueness check karne ke liye index chahiye hi).
  > ⚠️ Postgres **foreign key** column pe khud index **nahi** banata (MySQL InnoDB banata hai). `orders.user_id` pe index bhool gaye = har "user ke orders" query slow.
- **"On" karne ka koi switch nahi.** Index bante hi DB use **har write pe khud update** karta hai, aur har query pe **query planner khud decide** karta hai ki index use kare ya scan (statistics dekh ke — section 8). Tum query me index ka naam nahi likhte.
- **Check kaise karo:** `EXPLAIN ANALYZE SELECT ...` → `Seq Scan` (index nahi laga) vs `Index Scan` / `Index Only Scan` / `Bitmap Index Scan` (laga).
- Badi live table pe Postgres me `CREATE INDEX CONCURRENTLY` use karo — warna index bante waqt table ke writes ruk jaate hai.
 
> **Disk reads yaad hai? (lesson 04):** Disk RAM se ~1000x slow hai. Index ka poora point = **padhne wale pages kam karna**. `O(N)` (slow) vs `O(log N)` (fast) directly pages-read-count se correlate karta hai.
 
---
 
## 2. B-Tree Index — the default (Postgres/MySQL)
 
### 2.1 Pehle teen words — page, key, pointer
 
**Disk block / page:** DB disk se data **byte-byte nahi**, fixed size ke **chunks** me padhta-likhta hai — inko **page** (ya block) kehte hai. Postgres me **8 KB**, MySQL InnoDB me **16 KB**. Ye pages **SSD/HDD pe DB ki files** ke hisse hai; jo pages abhi use ho rahe hai unki copy **RAM** me cache hoti hai (Postgres `shared_buffers`, MySQL buffer pool). "1 disk read" = 1 page padhna.
 
**Key:** indexed column ki **value** (jaise `age = 25`). Index me ye values **sorted order** me rakhi jaati hai — yahi "sorted keys" hai.
 
**Pointer:** batata hai ki us value wali **asli row table me kahan** hai. Postgres me ye `(page number, slot)` hota hai, jaise `(page 512, slot 3)`.
 
> **Key + pointer = ek index entry.** Library catalog card jaisa: *"Harry Potter → shelf 12, row 3"*. Card khud kitaab nahi, bas pata batata hai. Index entry khud poori row nahi — sirf value + uska address.
 
Ek **node = ek page** jisme aisi bahut saari entries sorted line me hoti hai. 8 KB / ~20-40 bytes per entry ≈ **100-400 entries per node**.
 
### 2.2 Ek 8-row table se B-Tree
 
**Table `users`** (rows disk pe jis order me insert hui, waise hi padi hai — sorted nahi):
| Row address | id | name | age |
|---|---|---|---|
| R1 | 1 | Bob | 25 |
| R2 | 2 | Alice | 30 |
| R3 | 3 | Eve | 25 |
| R4 | 4 | Raj | 19 |
| R5 | 5 | Meera | 42 |
| R6 | 6 | Kabir | 35 |
| R7 | 7 | Zoya | 22 |
| R8 | 8 | Dev | 30 |
 
`CREATE INDEX idx_age ON users (age);` chalaya → DB saari ages **sort** karta hai: `19, 22, 25, 25, 30, 30, 35, 42` aur unse ek tree banata hai. Samjhane ke liye maan lo ek node me sirf **2 entries** fit hoti hai (asli me 100s):
 
```
                       ROOT NODE
                  [ 25 | 30 | 35 ]            ← sirf "signboards": kis raaste jaana hai
              /       |        |       \
             /        |        |        \
  LEAF 1           LEAF 2           LEAF 3           LEAF 4
 [19→R4, 22→R7] ⇄ [25→R1, 25→R3] ⇄ [30→R2, 30→R8] ⇄ [35→R6, 42→R5]
   (age < 25)      (25 ≤ age < 30)  (30 ≤ age < 35)   (age ≥ 35)
```
- **Node** = har ek box (har box = ek page).
- **Root node** = sabse upar wala box. Isme sirf **separator keys** hai — "25 se chhota left jao, 25-30 beech wale me…".
- **Leaf nodes** = sabse neeche ke boxes. Inme **asli entries** hai: `age → row address`. Leaves aapas me **side-by-side linked** hai (⇄) — range ke liye kaam aayega.
- Bade tree me root aur leaves ke beech **internal (branch) nodes** bhi hote hai — woh bhi sirf signboards.
- *(Technically Postgres/MySQL **B+Tree** use karte hai — saari entries leaves me, leaves linked. Log isko bhi "B-Tree" hi bolte hai.)*
 
### 2.3 `WHERE age = 25` — step by step (aur "hop" kya hai)
```
Hop 1: ROOT page padho → [25 | 30 | 35] me binary search → 25 ≥ 25 aur < 30 → LEAF 2 ka raasta
Hop 2: LEAF 2 page padho → entries: 25→R1, 25→R3 ✅
Phir:  table se R1 (Bob) aur R3 (Eve) ke pages padho → rows return
```
- **Hop = tree me ek level neeche jaana = ek node (page) padhna.** Root → leaf tak jitne levels, utne hops.
- **Node ke andar** keys sorted hai, toh DB us page ke andar **binary search** karta hai — ye RAM me hota hai (page load ho chuka), bahut sasta. Mehenga part sirf page ko disk se laana hai.
- Alice, Raj, Meera… ke pages **chhue bhi nahi**.
 
### 2.4 "1 million rows → depth ~3" ka matlab
Matlab: **1 million rows me se tumhari matching row(s) dhundhne ke liye sirf ~3 index pages padhne padte hai** (root → internal → leaf), phir table se woh row.
 
**"Left half / right half" nahi — "300 me se ek raasta":** binary tree me har node 2 raaste deta hai (left/right). B-Tree ka ek node 300 keys rakhta hai → **~300 raaste**. Isliye tree bahut **chauda (wide)** aur **chhota (shallow)**:
```
Level 1 (root):       1 node     → 300 raaste
Level 2:            300 nodes    → 300 × 300 = 90,000 raaste
Level 3 (leaves): 90,000 nodes   × ~300 entries = ~27 million entries
```
Toh 3 levels me ~27M rows aa jaati hai; 1 billion ke liye 4 levels. **Data 1000x badhe, depth sirf +1.** Upar ke levels (root, branches) itne chhote hai ki aksar **RAM me hi cached** rehte hai — asli disk read aksar 1-2 hi hota hai.
 
Lookup har level pe: *"main 25 dhundh raha hu; is node ki sorted keys me 25 kis do keys ke beech hai? → us beech wale child pe jao"*. Leaf tak pahunch ke entries mil jaati hai.
 
### 2.5 Range, Sorting, Prefix kaise?
Sab ek hi wajah se kaam karta hai — **leaves sorted hai aur side-by-side linked hai**:
 
- **Range** `WHERE age BETWEEN 22 AND 35`: tree me neeche utar ke **22** wali leaf pe pahuncho (start point), phir leaves pe **daayein chalte jao** (⇄ links) jab tak value > 35 na ho jaye. Beech ka sab mil gaya, baaki tree chhuna hi nahi.
```
LEAF 1 [19, 22✓] → LEAF 2 [25✓, 25✓] → LEAF 3 [30✓, 30✓] → LEAF 4 [35✓, 42✗ STOP]
```
- **Sorting** `ORDER BY age`: leaves already sorted hai → bas left se right padh lo, **alag se sort karne ki zarurat hi nahi**. `DESC` = right se left. `ORDER BY age LIMIT 10` = pehli 10 entries padhke ruk jao — super fast.
- **Prefix** `WHERE name LIKE 'Bob%'`: strings alphabetical sorted hai, toh "Bob" se shuru hone wale saare naam **ek saath, consecutive** pade hai (`Bob`, `Bobby`, `Boby`…). Isko DB range bana deta hai: `'Bob' ≤ name < 'Boc'`.
  - `LIKE '%bob'` (end me match) **nahi chalta** — "…bob" pe khatam hone wale naam sorted list me poore bikhre hai, consecutive nahi. (Iske liye GIN + trigram, section 6.)
  - *(Postgres me non-C collation pe prefix LIKE ke liye `text_pattern_ops` wala index chahiye hota hai — detail me mat jao, bas yaad rakho.)*
 
**B-Tree supports:**
```
Equality:  WHERE age = 25
Range:     WHERE age BETWEEN 20 AND 40
Prefix:    WHERE name LIKE 'Bob%'
Sorting:   ORDER BY age  (tree already sorted!)
```
 
---
 
## 3. Hash Index — O(1) but limited
 
Key ko hash function se ek **bucket number** me badla jaata hai; bucket me row ka address.
 
```
Hash Index on email:
  hash('bob@mail.com')   → bucket 7 → row 42
  hash('alice@mail.com') → bucket 3 → row 18
 
Query WHERE email = 'bob@mail.com':
  hash('bob@mail.com') = bucket 7 → row 42 found → O(1)!
```
 
> **Hash function recap (detail lesson 11 me):** hash = koi bhi input ko ek fixed-size number me convert karta hai, deterministically (same input → hamesha same output). Similar inputs → completely unrelated outputs.
 
### Tumhara sawaal: bucket 7 me sirf vikrant@gmail.com hi hoga?
**Nahi** — tumne flow sahi samjha (email ka hash = bucket = row), bas ek detail: buckets **limited** hai aur emails **crore**, toh ek bucket me **kai emails** aa sakti hai (**collision**).
```
hash('vikrant@gmail.com') % 1024 = 7
hash('neha@yahoo.com')    % 1024 = 7    ← same bucket!
 
Bucket 7 (ek page): [hash=0x9A1F → R42, hash=0x33C0 → R88, hash=0x9A1F → R105?]
```
Lookup: (1) email ka hash nikalo → bucket 7, (2) bucket 7 ka page padho, (3) uske andar sirf **chhoti si list** check karo — jinka hash match kare, (4) asli row khol ke confirm karo ki email sach me `vikrant@gmail.com` hai (do alag emails ka hash kabhi same bhi ho sakta hai).
 
**Phir O(1) kaise?** DB data badhne ke saath **buckets badhata rehta hai** (bucket split), taaki har bucket me average **thodi si** entries rahe. Bucket ki size table size (N) pe depend nahi karti → **average O(1)**. Worst case (bahut collisions) me slow ho sakta hai — isliye "average".
 
### Hash index kaun banata hai?
Tum khud: `CREATE INDEX idx_email_hash ON users USING HASH (email);` (Postgres). Default hamesha B-Tree hota hai — hash tabhi jab tum `USING HASH` likho. MySQL InnoDB me tum hash index nahi bana sakte (woh andar khud ek "adaptive hash index" banata hai). **Practically:** zyada tar log equality ke liye bhi B-Tree hi lagate hai — 3-4 hops vs 1 ka farak chhota hai, aur B-Tree range/sort bhi deta hai.
 
**Hash index supports:** sirf **Equality** (`WHERE email = 'x'`)
**Does NOT support:** Range (`age > 25`), Sorting, Prefix
 
**Kyu range nahi?** Hash "avalanche effect" — `hash(24), hash(25), hash(26)` bilkul unrelated random numbers. Koi order nahi bacha, isliye "range" nikalna impossible. B-Tree range support karta hai kyunki wo data **sorted** rakhta hai (hash nahi karta):
```
B-Tree (sorted):     [10,15,20,25,30,35,40]  → range easy, consecutive
Hash (scattered):    bucket7, bucket2, bucket9  → koi order nahi, range impossible
```
 
---
 
## 4. Composite Index — multiple columns
 
**Composite index = ek hi B-Tree jiski key ek column nahi, columns ka combination (tuple) hai.** Haan — `CREATE INDEX` chalake **tum khud** isko banate ho:
```sql
CREATE INDEX idx_country_age ON users (country, age);
```
Ab har index entry ki key = `(country, age)`, aur sorting **dictionary jaisi** hoti hai: pehle `country` se; jahan country same ho, wahan `age` se.
 
Table me ek `country` column jod do:
| Row | name | country | age |
|---|---|---|---|
| R1 | Bob | USA | 25 |
| R2 | Alice | India | 30 |
| R3 | Eve | USA | 18 |
| R4 | Raj | India | 20 |
| R5 | Meera | India | 25 |
| R6 | Kabir | USA | 35 |
 
Index ki leaves me entries is order me:
```
('India', 20)→R4, ('India', 25)→R5, ('India', 30)→R2,   ← India block, andar age sorted
('USA', 18)→R3,   ('USA', 25)→R1,   ('USA', 35)→R6      ← USA block, andar age sorted
```
Yahi **column order matter karta hai** — `(age, country)` banate toh pehle age se sort hota, bilkul alag list.
 
### 4.1 Left-Prefix Rule — kahan se aaya?
Ye koi upar se banaya rule nahi — ye **seedha sorting ka natija** hai. Index tabhi madad karta hai jab tumhari matching entries list me **ek jagah, consecutive** pade ho (taaki tree me jump karke ek block padh lo). Upar ki list dekho:
 
| Query | Matching entries list me kahan? | Index use? |
|---|---|---|
| `WHERE country = 'India'` | Pehli 3, **ek saath** | ✅ jump karo |
| `WHERE country = 'India' AND age = 25` | India block ke andar ek jagah | ✅ aur bhi precise |
| `WHERE country = 'India' AND age > 22` | India block ke andar, 25 se aage consecutive | ✅ |
| `WHERE age = 25` | `('India',25)` aur `('USA',25)` — **bikhre hue**, har country block me | ❌ jump nahi ho sakta |
| `WHERE country='India' AND city='Delhi'` (index `(country, age, city)` pe) | India block mila, but age skip ki → city India block me bikhri | ⚠️ partial: India tak jump, phir har entry check |
 
Table ki notation ka matlab: index `(A, B, C)` — **A = pehla column, B = doosra, C = teesra**; `WHERE A = x` matlab "query me pehle column pe condition hai". **"Left prefix"** = index ke columns ki list me **left se shuru hone wala hissa**: `(A)`, `(A, B)`, `(A, B, C)`. Query ki conditions inme se kisi ek se match karein toh index poora kaam karta hai. Beech ka column chhoda (`A` aur `C`, bina `B`) toh sirf `A` tak kaam karega.
 
**Analogy:** Phone book surname phir first name se sorted hai. "Sharma" dhundho → seedha jump. "Sharma, Rahul" → aur precise. Sirf "Rahul" (first name) dhundho → poori book palto, kyunki Rahul har surname ke andar bikhre hai.
 
> *Naye DBs (PostgreSQL 18, MySQL 8) kabhi "skip scan" kar lete hai — agar pehle column me bahut kam alag values ho (jaise 3 countries), toh har country ke block me alag-alag jump kar lete hai. Phir bhi design karte waqt left-prefix rule hi maan ke chalo.*
 
### 4.2 Equality pehle, range last — kyu?
Index `(user_id, created_at, status)` aur query `WHERE user_id=123 AND status='active' AND created_at > '2024-01-01'`:
```
user_id=123 ka block (created_at se sorted, phir status):
  (123, Jan-02, cancelled)
  (123, Jan-05, active)     ← created_at range shuru
  (123, Feb-10, cancelled)
  (123, Mar-01, active)
  ...
→ created_at range consecutive hai, but uske andar 'active' bikhre hai → har entry check karni padi
```
Index `(user_id, status, created_at)`:
```
(123, active, Jan-05)  ← user=123 + active ka block ek jagah, andar date sorted
(123, active, Mar-01)
(123, cancelled, ...)
→ jump to (123, active, '2024-01-01') aur wahan se seedha padho ✅ — koi faltu entry nahi
```
**Rule:** range condition ke **baad wale** columns sorted order ka fayda nahi le paate (range ke andar woh bikhar jaate hai). Isliye **equality columns pehle, range/sort wala column last.** Aur jo column zyada queries me aata hai, use left me rakho (taaki zyada queries left-prefix match karein).
 
```sql
-- GOOD: equality first, range last
CREATE INDEX idx ON orders (user_id, status, created_at);
-- BAD: range beech me → uske baad wala status index se filter nahi ho paata
CREATE INDEX idx ON orders (user_id, created_at, status);
```
 
---
 
## 5. Covering Index — table touch hi mat karo
 
Normal index entry me sirf **key + pointer** hota hai. Agar query ko aur columns chahiye, toh har match ke liye **table pe jaana** padta hai (ek extra page read, aksar random jagah). **Covering index = index me hi query ke saare columns daal do**, taaki table jaana hi na pade (**index-only scan**).
 
**Example — "My Orders" page:**
```sql
-- orders table: 20 columns (address, items JSON, payment details...) — ek row bhaari hai
SELECT id, status, total
FROM orders
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 20;
```
**Normal index `(user_id, created_at)`:**
```
1. Index se user 123 ki latest 20 entries → 20 pointers   (2-3 page reads)
2. Har pointer ke liye TABLE pe jao status, total ke liye  (20 random page reads!)
```
**Covering index:**
```sql
CREATE INDEX idx_orders_cover ON orders (user_id, created_at DESC) INCLUDE (status, total, id);
-- INCLUDE (Postgres 11+): ye columns leaves me "saath" rakh do, sorting me hissa nahi
```
```
1. Index se 20 entries — status, total, id wahi pade hai → DONE. Table chhui hi nahi.
```
**Analogy:** book ke index me har topic ke saath ek line ka summary bhi likha ho — tumhe aksar page kholna hi nahi padega.
 
**Trade-off:** index bada (zyada disk + RAM) aur har write pe zyada update. Sirf **hot, baar-baar chalne wali** queries ke liye. *(Postgres note: index-only scan tabhi poora kaam karta hai jab table ka "visibility map" up-to-date ho — VACUUM ye sambhalta hai.)*
 
---
 
## 6. GIN, GiST, BRIN (+ vector indexes) — special cases
 
B-Tree "ek value ↔ ek row" ke liye hai. Kuch data aisa hota hai jahan ye fit nahi hota:
 
**GIN (Generalized Inverted Index) — "andar kya-kya hai" ke liye**
- Ek column me **kai items** ho: array (`tags`), JSONB, ya text ke words. GIN har **item → us item wali rows ki list** rakhta hai (book ke peeche wale word-index jaisa — isliye "inverted").
- Use: `WHERE tags @> '{python}'` (python tag wale posts), `WHERE data @> '{"plan":"pro"}'` (JSONB), full-text search (`to_tsvector(body) @@ to_tsquery('refund')`), `pg_trgm` ke saath `LIKE '%bob%'`.
- Cost: writes slow, size bada.
 
**GiST (Generalized Search Tree) — "overlap / paas / andar" ke liye**
- Shapes, locations aur ranges ke liye: *"ye point us area ke andar?"*, *"ye time slot us slot se overlap karta?"*, *"sabse paas ke 10"*.
- Use: **PostGIS** — "mere 2 km ke andar drivers/restaurants"; **booking systems** — ek room ke time-ranges overlap na ho (exclusion constraint se double booking hi impossible); nearest-neighbor.
 
**BRIN (Block Range INdex) — bahut badi, naturally ordered tables ke liye**
- Har **block range** (e.g. 128 pages) ka sirf **min aur max** yaad rakhta hai. Query aaye toh jin ranges ka min-max match nahi karta unhe skip.
- Kaam tabhi karta hai jab data **disk pe already order me** ho — jaise logs/events jo time ke saath append hote hai (`created_at` badhta hi jaata hai).
- Use: 1 TB ki logs/metrics/IoT table pe `WHERE created_at > now() - interval '1 day'`. Index sirf **KBs** ka. Random order data pe bekaar.
 
**Bonus — Vector indexes (AI ke liye):** embeddings pe "sabse similar" dhundhna — **pgvector** ke **HNSW / IVFFlat** indexes (RAG me yahi use hote hai). Ye approximate hote hai: thoda accuracy dekar bahut speed.
 
---
 
## 7. Index types quick comparison
 
| Type | Lookup | Range | Sort | Size | Best for |
|---|---|---|---|---|---|
| B-Tree | O(log N) | ✅ | ✅ | Medium | General (default) |
| Hash | O(1) avg | ❌ | ❌ | Small | Exact match |
| Composite | O(log N) | ✅ (leftmost) | ✅ | Larger | Multi-column |
| Covering | O(log N) | ✅ | ✅ | Large | Avoid table hops |
| GIN | Depends | ❌ | ❌ | Large | Full-text, arrays, JSON |
| GiST | O(log N)-ish | ✅ | Depends | Medium | Geospatial, ranges, nearest |
| BRIN | Scan of tiny summary | ✅ | ❌ | Tiny | Naturally-ordered data (timestamps) |
| HNSW (pgvector) | ~O(log N) approx | ❌ | ❌ | Large | Vector similarity (RAG) |
 
---
 
## 8. The Cost of Indexes — kab NAHI lagana
 
### 8.1 Write overhead kya hai?
**`INSERT`** = SQL ka command jo table me **nayi row** daalta hai (`INSERT INTO users (name, age) VALUES ('Vikrant', 28);`). "INSERT into table" = table me row daalna.
 
Ek INSERT pe DB ko kya-kya likhna padta hai:
```
INSERT INTO users ... (users pe 5 indexes hai)
  1. Row → table ke page me
  2. email index   → sahi leaf dhundho, sorted jagah pe entry ghusao (page full? → page split)
  3. age index     → same
  4. country index → same
  5. (country, age) composite → same
  6. created_at index → same
  (+ har cheez ka WAL log, aur replicas ko bhi yahi sab bhejna)
= 1 table write + 5 index writes = 6 writes
```
`UPDATE` pe indexed column badla → purani entry hatao + nayi daalo. `DELETE` → entries mark karo, baad me cleanup. **Har extra index = har write thoda slow + zyada disk + zyada replication lag.** Reads ke liye index fayda, writes ke liye tax.
 
```
1. WRITE OVERHEAD — har INSERT table + saare indexes update karta hai
                    (5 indexes = 6 writes per INSERT)
2. STORAGE        — indexes disk space lete hai (kabhi table se bhi bade)
3. MEMORY         — indexes RAM me fit hone chahiye (more indexes = more RAM)
4. MAINTENANCE    — fragment/bloat ho sakte hai (REINDEX), migrations slow
```
 
### 8.2 Cardinality — kya, kyu, kahan ⭐
**Cardinality = ek column me kitni alag-alag (distinct) values hai.** 10M users ki table me:
| Column | Distinct values | Cardinality |
|---|---|---|
| `email` | ~10,000,000 (sab unique) | **High** |
| `city` | ~5,000 | Medium |
| `country` | ~200 | Low-medium |
| `is_active` / `gender` | 2-3 | **Low** |
 
**Kyu matter karta hai:** index tab fayda deta hai jab query **chhota hissa** rows ka laaye (isko **selectivity** kehte hai).
- `WHERE email = 'x'` → 10M me se **1 row** → index perfect.
- `WHERE is_active = true` → 10M me se **9M rows** → index se 9M baar alag-alag jagah jump karna, seedha poori table padhne (seq scan) se **slower**. Isliye planner yahan index **ignore** kar deta hai, chahe tumne banaya ho.
 
**Kahan-kahan use hota hai:**
- **Index lagana hai ya nahi** — high cardinality columns best.
- **Query planner** — Postgres `ANALYZE` har column ki stats (distinct values, common values) collect karta hai; planner inhi se decide karta hai index ya scan.
- **Composite order** — jo column zyada filter kare (zyada selective) aur zyada queries me aaye, use left me.
- **Sharding key** — low cardinality key (jaise `country`) pe shard kiya → kuch shards pe pahaad (hot shards) (Topic 2 lesson 13).
- **Exception — skewed data:** `status` ki 3 values but `'failed'` sirf 0.1% rows → **partial index** `CREATE INDEX ON payments (created_at) WHERE status = 'failed';` — chhota aur bahut useful.
 
> **Do matlab, confuse mat hona:** Data modeling me "cardinality" ka matlab **relationship** bhi hota hai — 1:1, 1:N, N:M (ek user ke kai orders = one-to-many). Indexing me matlab = distinct values ki ginti.
 
### 8.3 Kab index NAHI lagao
| Scenario | Kyu |
|---|---|
| Chhoti tables (< 1000 rows) | Full scan already fast (kuch pages), overhead worth nahi |
| Write-heavy tables (logs, events) | Har write saare indexes update karta hai (slow writes) |
| Low-cardinality columns | Boolean, status (3 values) — planner waise bhi ignore karega (partial index exception) |
| Rarely-queried columns | Index unused, but storage/write cost leta hai |
| Frequently-updated columns | Har update = index modification |
| Wide columns (TEXT/BLOB) | Bade values pe index expensive (GIN/hash/prefix socho) |
 
---
 
## 9. Index selection strategy (interview-ready)
 
```
Step 1: Hot queries identify karo (most frequent, slowest) — pg_stat_statements, slow query log
Step 2: Har query ke WHERE aur ORDER BY columns dekho
Step 3: Composite index banao:
        - equality columns pehle (left to right)
        - range/sort column last
Step 4: Covering index worth hai? (critical queries ke liye)
Step 5: EXPLAIN ANALYZE se confirm karo ki index laga
Step 6: Over-indexing avoid karo (max ~5-7 per table), unused indexes hatao
```
```sql
-- Hot query: "user ke recent orders, status se filter"
-- SELECT * FROM orders WHERE user_id=? AND status=? ORDER BY created_at DESC
CREATE INDEX idx ON orders (user_id, status, created_at DESC);
-- user_id: equality (selective), status: equality, created_at: range/sort last
```
 
**Real companies:** Uber (geospatial indexes for "drivers near me"), Twitter (composite on `(user_id, created_at)` for timelines), Stripe (B-tree on `payment_intent_id` + idempotency key), Netflix (BRIN-style time indexes for time-range).
 
---
 
## 10. Clustered vs Secondary index (interview favorite)
 
### Problem kahan se aaya?
Table ki rows ko disk pe **kisi ek order me** rakhna padta hai. Do design choices hai:
1. Rows bas **jaise aaye waise** daal do (Postgres ka "heap"), aur saare indexes alag se banao.
2. Rows ko hi **primary key ke order me sorted** rakho — table khud ek B-Tree ban jaaye (MySQL InnoDB, SQL Server).
 
### Clustered index
**Table = index.** Rows **physically primary key ke order me**, B-Tree ki leaves me hi **poori row** padi hai (sirf pointer nahi).
```
Clustered (by id) — leaves me hi poori rows:
  [id:1 Bob 25 USA] [id:2 Alice 30 India] [id:3 Eve 18 USA] ...
```
- **Ek hi per table** — rows physically ek hi tarike se sorted ho sakti hai (ek kitaab ke pages ek hi order me).
- **Kaun banata hai:** MySQL InnoDB me jab tum `PRIMARY KEY` define karte ho, wahi **automatically** clustered index ban jaata hai (PK na ho toh pehli `UNIQUE NOT NULL` column, warna hidden row id).
- **Fayda:** PK pe range (`WHERE id BETWEEN 1000 AND 2000`) = consecutive pages; aur PK lookup me table ka alag hop nahi.
 
### Secondary index
**Har doosra index** (email, age…) — ek alag tree jo **primary key** (InnoDB) ya row address (Postgres) ki taraf point karta hai.
```
Secondary (on email): separate tree
  "alice@" → id 2,  "bob@" → id 1
Lookup email='bob@': secondary tree → id 1 → clustered tree me id 1 → full row  (2 tree walks)
```
Isko **"bookmark lookup"** bhi kehte hai. (Covering index yahi doosra walk bachata hai.)
 
**Analogy:** **Dictionary** = clustered (kitaab khud shabdon ke order me hai — word mila toh meaning wahi). **Textbook ke peeche ka index** = secondary (topic → page number, phir page kholna padta hai).
 
### Postgres alag kaise?
Postgres me **clustered index by default hota hi nahi** — table ek **heap** hai (insertion order), aur **primary key bhi ek secondary-style index** hai jo row address (ctid) pe point karta hai. `CLUSTER` command ek baar sort kar deta hai, but nayi rows ke saath maintain nahi karta.
 
### Real-life fayde aur gotchas
- **Primary key chhota rakho (InnoDB):** har secondary index entry me PK **copy** hota hai. PK = 36-char UUID string → har index bada.
- **Random UUID (v4) PK = page splits:** naye rows tree me **random jagah** ghuste hai → pages bar-bar split, cache kharab. Isliye **auto-increment** ya **time-ordered IDs (UUIDv7, ULID, Snowflake)** — naye rows hamesha "end" pe jaate hai.
- **Related data ko paas rakhna:** chat app me messages ka PK `(conversation_id, message_id)` → ek conversation ke saare messages disk pe **ek saath** → "is chat ke latest 50 messages" = bas kuch pages. (Cassandra me isi idea ko partition key + **clustering key** kehte hai.)
 
---
 
## Quick Self-Check
1. Full table scan andar se kaise chalta hai (page-by-page loop)? `EXPLAIN` me kaise dikhta hai?
2. Index kaun banata hai? DB khud kab banata hai? "On" kaise hota hai — query me index ka naam likhna padta hai?
3. Page/block kya hai (size, kahan rehta hai)? Key aur pointer kya hai — library card analogy?
4. 8-row table ka B-Tree banao: root node kaun, leaf node kaun? `WHERE age=25` me kitne hops?
5. "1 million rows → depth 3" ka matlab? Binary tree (2 raaste) vs B-Tree (300 raaste)?
6. Range, ORDER BY, aur `LIKE 'Bob%'` B-Tree se kaise? `LIKE '%bob'` kyu nahi?
7. Hash bucket me collision kya hai? Phir bhi O(1) kaise? Range kyu nahi?
8. Composite index `(country, age)` ki leaves ka order likho. `WHERE age=25` kyu index use nahi karta (left-prefix rule kahan se aata hai)?
9. Equality pehle, range last — `(user_id, created_at, status)` kyu bura?
10. Covering index kaunsa hop bachata hai? `INCLUDE` kya karta hai?
11. GIN, GiST, BRIN — har ek ka ek real use?
12. Ek INSERT pe 5 indexes = kitne writes aur kyu?
13. Cardinality kya hai? `is_active` pe index planner kyu ignore karta hai? Partial index kab?
14. Clustered vs secondary — InnoDB vs Postgres? UUIDv4 primary key kyu bura?
 
---
 
## Flow — ek query index ke saath kaise chalti hai
1. **Query aayi** (`SELECT ... WHERE user_id=123 AND status='active' ORDER BY created_at DESC`) → **planner** table ki stats (cardinality, rows count) dekh ke decide karta hai: seq scan ya kaunsa index (left-prefix match hota hai? kitni rows aayengi?).
2. **Index chuna** → root page (aksar RAM me) → node ke andar binary search → sahi child → … → **leaf** tak (har level = 1 hop = 1 page). 1M rows pe ~3 hops.
3. **Leaf pe matching entries** milti hai — B-Tree ki wajah se ye **consecutive** hai, toh range/order ke liye leaves pe side me chalte jao, `LIMIT` pe ruk jao.
4. **Covering hai?** → answer index se hi. **Nahi** → har pointer se **table page** padho (secondary → clustered/heap) aur poori row lo.
5. **Writes pe ulta tax:** har `INSERT/UPDATE/DELETE` table ke saath **har index** ko bhi update karta hai — isliye sirf hot queries ke liye, sahi order me, kam se kam indexes.
 
---
**Next:** `07-Query-Complexity-Big-O.md` — har operation ki exact cost, aur DB-by-DB comparison.
 