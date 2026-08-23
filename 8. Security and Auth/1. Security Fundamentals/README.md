# 01 — Security Fundamentals: AuthN vs AuthZ

> **Is lesson me:** Security ka foundation — CIA triad (kya protect karte hai), threat modeling, defense in depth, least privilege, aur sabse important distinction: **authentication (tum kaun ho) vs authorization (tum kya kar sakte ho).** Ye poore module ki neenv hai.

Prerequisites: Topics 1-7 helpful. Ye Topic 8 ki pehli file hai.

---

## 1. Security kya protect karti hai — the CIA triad

Security ke 3 core goals (CIA):
```
C — Confidentiality: data sirf authorized log dekh sakein (encryption, access control)
I — Integrity:       data tamper na ho (kisi ne badla toh pata chale — hashing, signatures)
A — Availability:    system available rahe (DDoS/attacks se down na ho — rate limiting, Topic 3)
```

Har security measure in teen me se kisi ko serve karta hai:
```
Encryption (TLS)        → Confidentiality (beech me koi na padhe)
Password hashing        → Confidentiality (breach me passwords safe)
Digital signatures      → Integrity (tampering detect)
Rate limiting (Topic 3) → Availability (overload/abuse rok)
Access control          → Confidentiality + Integrity (sahi log hi access karein)
```

**Frontend analogy:** tum ye already jeete ho — HTTPS (confidentiality), Subresource Integrity `integrity=` hashes (integrity check on CDN scripts), rate-limited APIs (availability). Backend security wahi principles, deeper.

---

## 2. AuthN vs AuthZ — the fundamental distinction ⭐

Ye do cheezein constantly confuse hoti hai. Clear karo:

```
AUTHENTICATION (AuthN):  "Tum KAUN ho?"  → identity verify karna
                         (login: email + password, "Login with Google")

AUTHORIZATION (AuthZ):   "Tum KYA kar sakte ho?"  → permissions check karna
                         (kya ye user is admin page ko access kar sakta?)
```

**Order:** pehle authenticate (kaun ho), phir authorize (kya kar sakte ho).
```
Request aati hai:
  1. AuthN: kya ye valid logged-in user hai? (token/session verify)
     NAHI → 401 Unauthorized ("pehle login karo")
  2. AuthZ: kya is user ke paas is action ki permission hai?
     NAHI → 403 Forbidden ("tum logged-in ho, but ye nahi kar sakte")
```

> ⚠️ **401 vs 403 (interview + practical):**
> - **401 Unauthorized** = "identity nahi pata / invalid" → authenticate karo (naam thoda ulta hai — "unauthorized" but actually means unauthenticated)
> - **403 Forbidden** = "identity pata hai, but ye action allowed nahi" → authorized nahi ho

**Analogy:** ek office building. **AuthN** = reception pe apna ID card dikhana (tum kaun ho, verified). **AuthZ** = tumhara card sirf kuch floors kholta hai (tum kya access kar sakte ho). ID valid hai (authenticated) but 10th floor (server room) nahi khulta (not authorized).

**Frontend analogy:** login form = AuthN. Route guards / conditionally rendering an "Admin" button based on `user.role` = AuthZ. But ⚠️ frontend AuthZ sirf UX hai — **real AuthZ hamesha backend pe** (client ko trust nahi kar sakte — koi bhi API directly call kar sakta hai, section 4).

---

## 3. Defense in Depth — layers of security ⭐

Ek single security measure pe rely mat karo. **Multiple layers** — ek fail ho toh agli bache.
```
Layer 1: Network      → firewall, TLS, DDoS protection (Topic 1)
Layer 2: Application  → authN/authZ, input validation, rate limiting
Layer 3: Data         → encryption at rest, access control, hashing
Layer 4: Monitoring   → logging, anomaly detection, alerts
```
Ek attacker ko **saari** layers todni padein. Ek TLS bypass ho jaye toh bhi encryption-at-rest data bachaye; ek auth bug ho toh bhi least-privilege damage limit kare.

**Analogy:** ek castle — moat + walls + guards + locked vault. Ek defense todne se pura castle nahi girta. Security = layers, not a single wall.

---

## 4. Never trust the client ⭐ (the golden rule)

Sabse important security principle backend developers ke liye: **client se aane wali koi bhi cheez trusted nahi hai.** Browser, mobile app, API caller — koi bhi input manipulate ho sakta hai.

```
❌ TRUST NAHI karo:
  - Frontend validation (bypass ho sakta — direct API call)
  - Client-sent user roles/IDs ("user_id=5, role=admin" — attacker koi bhi bhej de)
  - Hidden form fields, URL params (easily changed)
  - "Ye request hamare app se hi aayi" (koi bhi curl kar sakta)

✅ HAMESHA server-side:
  - Re-validate saara input (Pydantic — FastAPI, lesson 10)
  - AuthZ checks server pe (frontend button hide karna security nahi)
  - User identity token/session se derive karo (client-sent nahi)
```

> ⚠️ **Classic bug:** `GET /api/orders/{order_id}` — user apna order_id badalke doosre ka order dekh le (IDOR — Insecure Direct Object Reference — OWASP Broken Access Control, lesson 08). **Fix:** server check kare "kya ye order is logged-in user ka hai?" — client-sent order_id pe blindly trust nahi.

**Frontend analogy:** tumne "disable submit button" ya "hide admin link" kiya hoga — wo UX hai, security nahi. Koi bhi DevTools se button enable kar sakta, ya API directly hit kar sakta. **Real check backend pe.** Ye mindset shift frontend → backend security ka core hai.

---

## 5. Least Privilege

**Principle of least privilege** = har user/service/component ko sirf **minimum permissions** do jo uske kaam ke liye chahiye — zyada nahi.
```
❌ App DB user = superuser (sab kuch kar sakta)
✅ App DB user = sirf apni tables pe SELECT/INSERT/UPDATE (no DROP, no other DBs)

❌ Service ke paas saare S3 buckets ka access
✅ Service ke paas sirf apne bucket ka access
```
Agar wo user/service compromise ho, damage **limited** rehta hai (blast radius chhota).

**Frontend analogy:** ye ek API token ko sirf zaroori scopes dena jaisa hai (GitHub token with only `repo:read`, not `admin`). Minimum access = minimum damage if leaked.

---

## 6. The security mindset (interview framing)

```
Har feature ke liye poocho:
  1. AuthN: kaun access kar sakta hai? (logged in? verified?)
  2. AuthZ: unhe kya karne ki permission hai? (server-side check)
  3. Input: kya validate/sanitize kiya? (never trust client)
  4. Data: sensitive data encrypted? (transit + rest)
  5. Blast radius: compromise ho toh damage kitna? (least privilege)
  6. Monitoring: attack detect kaise hoga? (logging, alerts)
```

**Interview line:** *"Security ke liye main defense-in-depth lunga: TLS in transit, hashed passwords + encryption at rest, authN via tokens, authZ checks server-side on every request (never trust client), least-privilege for services, input validation, rate limiting for abuse, aur monitoring/logging for detection. Ek layer fail ho toh baaki bachein."*

---

## Quick Self-Check
1. CIA triad — teeno kya protect karte hai? Har ek ka ek measure?
2. AuthN vs AuthZ — "kaun ho" vs "kya kar sakte ho"? Order?
3. 401 vs 403 — difference?
4. Defense in depth kya hai — castle analogy?
5. "Never trust the client" — 3 cheezein jo trust nahi karni? IDOR example?
6. Frontend validation/button-hiding security kyu nahi hai?
7. Least privilege kya hai — DB user example?

---
**Next:** `02-Password-Security.md` — passwords kaise store karo (safely).