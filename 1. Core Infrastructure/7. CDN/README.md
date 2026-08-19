# 07 — CDN (Content Delivery Network)

> **Is lesson me:** CDN kya hai aur kaise latency kam karta hai (edge servers), pull vs push CDN, kya cache karo/na karo, CDN cache invalidation (versioned URLs/purge), aur edge computing. Topic 3 me CDN ko "Layer 2 cache" bola tha — ye uski poori delivery-infrastructure story hai.

Prerequisites: `06-Proxies.md` (CDN edge ek specialized reverse proxy hai), `03-Networking-III` (DNS/anycast)

---

## 1. CDN kya hai

**CDN (Content Delivery Network)** = duniya bhar me phaile servers (**edge servers**) ka network jo content ko **user ke sabse paas wali location** se cache karke serve karta hai.

**Analogy:** Amazon warehouses. Har order ko ek central warehouse (Seattle) se ship karne ke bajaye (high latency, expensive), Amazon har bade city me warehouse (edge location) rakhta hai. Tumhara package 20 miles se aata hai, 2000 se nahi.

```
Without CDN:
  Mumbai user → image request → US server tak → 200ms
  London user → image request → US server tak → 150ms

With CDN:
  Mumbai user → Mumbai CDN edge → 10ms
  London user → London CDN edge → 10ms
```

**Kyu itna fast:** content user ke paas hai, aur physics — light ki speed fiber me bhi limited hai (lesson 01 ka RTT concept, lesson 03 ka anycast routing). Kam distance = kam RTT = fast.

---

## 2. CDN kaise kaam karta hai

```
Pehli request (cache MISS):
  User → CDN Edge (Mumbai) → "mere paas nahi" → Origin (US) → image return
  CDN Edge image locally cache karta hai
  User ko return (ye ek baar slow)

Agli requests (cache HIT):
  User → CDN Edge (Mumbai) → "mere paas hai!" → turant return (fast)
  Origin ko contact hi nahi kiya
```

Ye bilkul cache hit/miss (Topic 3 lesson 02) hai, but **geographically distributed**. Origin server ka load 90%+ kam ho jaata hai (edge sab handle karta hai).

---

## 3. Pull vs Push CDN

### Pull CDN (lazy — most common)
CDN content ko origin se **tab** fetch karta hai jab pehli baar request ho. Koi request na kare toh cache hi nahi hota.
```
User file maange → CDN ke paas nahi → origin se pull → cache → serve
Agla user same file → CDN ke paas hai → cache se serve
```
**Pros:** sirf popular content cache hota hai, no upfront setup. **Cons:** pehla user slow (origin fetch); spike me edge ke paas na ho.
**Used by:** Cloudflare, CloudFront (default), most CDNs.

### Push CDN
Tum content ko **explicitly** edge servers pe upload karte ho. CDN bina origin contact kiye serve karta hai.
```
Tum video CDN pe upload → CDN saare edges pe distribute
User video maange → CDN ke paas already → serve
```
**Pros:** pehli request bhi fast, tum control karte ho kya cache ho. **Cons:** storage cost (saare edges ke liye pay), uploads manage karo.
**Used by:** Netflix (popular content ISP servers pe pre-position karta hai), video platforms.

### Kaunsa kab
| Content | Approach |
|---|---|
| Static assets (CSS/JS/images) | Pull CDN |
| User-uploaded media (profile pics) | Pull CDN |
| Popular video content | Push CDN (pre-position) |
| Live streaming | Specialized CDN (not cached) |
| API responses | Usually NOT CDN (dynamic), unless cacheable |

---

## 4. Kya cache karo, kya nahi

| Cache karo | Cache mat karo |
|---|---|
| Static files (CSS, JS, fonts) | User-specific data (dashboard, settings) |
| Images, videos, PDFs | Authentication tokens |
| Public API responses (product catalog) | Real-time data (stock prices, chat) |
| HTML pages (anonymous users ke liye) | POST/PUT/DELETE responses |

**Golden rule:** **public + rarely-changing** content cache karo. **Personalized + real-time** content nahi.

---

## 5. CDN Cache Invalidation

Content update kiya, ab CDN naya version kaise serve kare?

### Strategy 1: Versioned URLs (best) ⭐
```
v1: /static/app.v1.js
v2: /static/app.v2.js   (naya URL = CDN naye file ki tarah treat karta = no stale cache)
```
CDN indefinitely cache kar sakta hai. Naya deployment = naya URL. Purana URL bhi kaam karta rehta hai (purane HTML wale users ke liye). **Ye sabse clean tarika hai** — tumne frontend build me `app.abc123.js` (hash in filename) dekha hoga, wahi ye hai.

### Strategy 2: Cache-Control headers (lesson recap)
```
Cache-Control: public, max-age=86400   (24 ghante cache)
Cache-Control: no-cache                 (hamesha origin se revalidate)
Cache-Control: no-store                 (kabhi cache nahi)
```

### Strategy 3: Purge API
Most CDNs ek API dete hai force-invalidate ke liye:
```
POST /purge {"urls": ["https://example.com/image.png"]}
→ CDN saare edges se hataata hai → agli request fresh origin se
```
**Trade-off:** purge ko propagate hone me time lagta hai (seconds to minutes across all edges).

---

## 6. Edge Computing (CDN + logic)

Modern CDNs edge pe **code bhi run** kar sakte hai (sirf static cache nahi):
```
Traditional: User → CDN (static only) → Origin (all logic)
Edge compute: User → CDN Edge (simple logic: auth, redirects, A/B) → Origin (sirf DB ke liye)
```

**Use cases:** edge pe URL redirects (origin hit nahi), A/B testing (variant edge pe decide), auth token validation (unauthenticated requests origin se pehle reject), image resizing at edge.

**Examples:** Cloudflare Workers, AWS Lambda@Edge, Fastly Compute@Edge.

> **Connect to Topic 3:** URL shortener me humne bola tha "popular links CDN se resolve ho jaaye" — edge computing se redirect logic edge pe hi chal sakta hai, origin tak pahunche bina. Aur rate limiting (Topic 3) bhi edge pe ho sakti hai.

---

## 7. CDN performance numbers + providers

| Metric | Without CDN | With CDN |
|---|---|---|
| Latency (India user, US server) | 200ms | 15ms (India edge se) |
| Origin server load | 100% requests | 5-10% (90%+ CDN se) |
| Bandwidth cost | High | Lower (CDN absorb karta) |
| Availability during origin outage | 0% | Partial (cached content serve) |

**Providers:** Cloudflare (general + DDoS, free tier, Workers), AWS CloudFront (S3 integration, Lambda@Edge), Fastly (instant purge <150ms), Akamai (largest, 300K+ edges), Google Cloud CDN (anycast).

**Interview how-to-say:** *"User-uploaded images S3 me store hoti hai. Hum CloudFront ko S3 ke saamne rakhte hai taaki images nearest edge se serve ho. Pehli request origin se, baaki cache se ~10ms me. Cache busting ke liye versioned URLs."* (Ye Topic 2 lesson 15 ka S3 + ye CDN = complete media delivery.)

---

## 8. Common interview Qs (quick)
- **"Origin down ho jaye?"** → CDN cached content serve karta rehta hai ("stale-while-error" — complete outage se better).
- **"CDN vs read replica?"** → CDN final rendered bytes cache karta hai; read replica DB-level queries. CDN cached request ke liye server hi eliminate kar deta hai. **Dono use karo** — CDN static, replicas dynamic (Topic 2 lesson 12).
- **"Personalized content CDN pe?"** → Mat karo. CDN sirf static/public. Personalized directly origin, ya edge computing se assemble.

---

## Quick Self-Check
1. CDN latency kaise kam karta hai — Amazon warehouse analogy + physics reason?
2. CDN cache hit/miss Topic 3 ke caching se kaise same/alag hai?
3. Pull vs push CDN — difference aur kab kaunsa (Netflix push kyu)?
4. Kya cache karo vs nahi — public vs personalized rule?
5. Versioned URLs cache invalidation kaise solve karte hai (frontend build se relate)?
6. Edge computing kya hai + ek use case?
7. "CDN vs read replica" — dono kaha use karte hai?

---
**Next:** `08-API-Gateway.md` — microservices ka single smart entry point.