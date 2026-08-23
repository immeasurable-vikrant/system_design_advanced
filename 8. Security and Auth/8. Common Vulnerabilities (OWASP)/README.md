# 08 — Common Vulnerabilities (OWASP)

> **Is lesson me:** Sabse common web attacks — injection (SQL), XSS, CSRF, SSRF, broken access control — aur har ek ka defense. Ye OWASP Top 10 ke highlights hai. Frontend se tum kuch already jaante ho (XSS, CSRF); yahan attacker + defender dono side samjhenge.

Prerequisites: `01-Security-Fundamentals-AuthN-vs-AuthZ.md`, Topic 2 lesson 05 (SQL injection)

---

## 1. OWASP Top 10 — the reference

**OWASP Top 10** = most critical web security risks ki list (industry standard). Sab yaad karne ki zarurat nahi; ye lesson sabse important + common ones cover karta hai. Core mindset (lesson 01): **never trust input, always check authZ, defense in depth.**

---

## 2. Injection (SQL Injection) ⚠️

**Attack:** user input ko directly query/command me daalna, jisse attacker malicious code inject kare.
```
❌ VULNERABLE:
  query = "SELECT * FROM users WHERE name = '" + user_input + "'"
  Attacker input: ' OR '1'='1' --
  → SELECT * FROM users WHERE name = '' OR '1'='1' --'
  → saare users return! (ya DROP TABLE, data exfiltration)
```

> **Recap (Topic 2 lesson 05):** **Defense = parameterized queries** (prepared statements). Input ko data ki tarah treat karo, code ki tarah nahi.
```
✅ SAFE (parameterized):
  cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
  → driver input ko safely escape karta, injection impossible

✅ ORM (SQLAlchemy) auto-parameterizes (Topic 2 lesson 05)
✅ Pydantic (FastAPI) input validation adds a layer
```
**Kabhi** string concatenation se queries mat banao. Ye applies to SQL, NoSQL, OS commands (command injection), LDAP — koi bhi interpreter.

---

## 3. XSS (Cross-Site Scripting) ⚠️

**Attack:** attacker malicious JavaScript inject karta hai ek page me, jo doosre users ke browsers me chalta hai (unki cookies/data chura sakta).
```
❌ VULNERABLE:
  User ek comment post kare: <script>steal(document.cookie)</script>
  App use directly render kare → doosre users ke browser me script chale
  → session cookies chori (session hijacking — lesson 03)
```
**Types:** stored (DB me saved, sabko serve), reflected (URL/input se echo), DOM-based (client-side).

**Defenses:**
```
1. OUTPUT ENCODING/ESCAPING — user content render karte waqt escape karo
   (< → &lt;) — script text ban jaata, execute nahi hota
   (React ye by default karta — {userContent} auto-escapes!)
2. Content Security Policy (CSP) header — kaunse scripts chal sakte restrict
3. HttpOnly cookies (lesson 03) — JS cookie access na kar sake (damage limit)
4. Input validation/sanitization (rich text ke liye — DOMPurify)
```

**Frontend analogy:** tum React use karte ho — `{userInput}` **auto-escapes** (XSS safe). But `dangerouslySetInnerHTML` bypass karta hai (naam me warning hai!). Backend me bhi: templates auto-escape karein, ya manually encode. XSS tumhare domain me hai — ab attacker side bhi clear.

---

## 4. CSRF (Cross-Site Request Forgery) ⚠️

**Attack:** attacker ek user ke browser ko trick karta hai ek authenticated request bhejne ke liye (user ko pata bhi nahi).
```
User bank.com pe logged in hai (session cookie hai)
Attacker user ko evil.com pe laata hai, jaha ek hidden form:
  <form action="bank.com/transfer" method="POST">
    <input name="to" value="attacker"> <input name="amount" value="10000">
  </form> (auto-submit)
Browser AUTOMATICALLY bank.com cookie attach karta (lesson 03) → transfer ho jaata!
```
Root cause: browser cookies **automatically** bhejta hai (lesson 03), toh attacker site se bhi authenticated request ban sakti.

**Defenses:**
```
1. SameSite cookies (lesson 03) — Lax/Strict → cross-site pe cookie na jaaye (primary defense)
2. CSRF tokens — ek unpredictable token har form me, server verify kare
   (attacker ko token pata nahi → forged request fail)
3. Check Origin/Referer header
4. For APIs with Authorization header (not cookies) → CSRF largely N/A
   (attacker header set nahi kar sakta cross-site; ye JWT-in-header ka ek plus)
```
> **Note:** CSRF mainly **cookie-based auth** ka problem hai. Agar tum JWT `Authorization` header me bhejte ho (lesson 04), CSRF mostly N/A (browser wo header auto-attach nahi karta cross-site).

---

## 5. Broken Access Control (OWASP #1) ⚠️⭐

Sabse common serious bug — authZ check bhool jaana ya galat karna.
```
IDOR (Insecure Direct Object Reference):
  GET /api/orders/1234  → tumhara order
  GET /api/orders/1235  → koi aur ka order — but server check nahi karta!
  → attacker IDs iterate karke sabka data padh le

Missing function-level authZ:
  /admin/delete-user — sirf UI me hidden, but API directly callable
  → non-admin API hit karke delete kar de
```
> **Recap (lesson 01, 06):** **Defense = server-side authZ on EVERY request.** "Kya ye resource is logged-in user ka hai / is action ki permission hai?" — client-sent ID/role pe kabhi trust nahi. Frontend hiding ≠ security.

---

## 6. SSRF (Server-Side Request Forgery)

**Attack:** attacker server ko trick karta hai ek internal/unintended request bhejne ke liye.
```
App user se ek URL le ("fetch this image URL"):
  Attacker de: http://169.254.169.254/  (cloud metadata endpoint!)
  → server us internal endpoint ko fetch kare → cloud credentials leak!
```
**Defense:** URL allowlists (sirf approved domains), internal IPs block, metadata endpoints firewall, validate + sanitize URLs.

---

## 7. Other important ones (quick)

| Vulnerability | Defense |
|---|---|
| **Security misconfiguration** | Secure defaults, disable debug in prod, patch, minimal exposure |
| **Sensitive data exposure** | Encrypt transit (TLS) + rest, no secrets in logs/errors (lesson 09) |
| **Vulnerable dependencies** | `pip audit`/Dependabot, keep libraries updated |
| **Insufficient logging** | Log security events, monitor, alert (detect attacks) |
| **Auth failures** | Strong passwords (lesson 02), MFA, rate limit (brute-force) |

---

## 8. The defense checklist

```
1. Injection    → parameterized queries (never concatenate) — Topic 2
2. XSS          → output encoding + CSP + HttpOnly cookies
3. CSRF         → SameSite cookies + CSRF tokens (or header-based auth)
4. Access ctrl  → server-side authZ every request (never trust client) — #1!
5. SSRF         → URL allowlists, block internal
6. Data         → encrypt transit + rest, no secrets in logs (lesson 09)
7. Deps         → keep updated, audit
8. Rate limit   → brute-force + abuse (Topic 3)
9. Monitor      → log + alert on anomalies
```

**Interview line:** *"Injection ke liye parameterized queries + input validation (Pydantic). XSS ke liye output encoding + CSP + HttpOnly cookies. CSRF ke liye SameSite cookies (ya header-based JWT). Broken access control — OWASP #1 — ke liye server-side authZ on every request, never trusting client-sent IDs/roles (IDOR). SSRF ke liye URL allowlists. Plus TLS, encryption at rest, dependency audits, rate limiting, aur security logging."*

---

## Quick Self-Check
1. SQL injection kaise kaam karta, aur defense (parameterized — Topic 2)?
2. XSS kya hai — attacker script kaise doosre users tak? React auto-escape se relate?
3. CSRF kaise kaam karta (browser auto-sends cookies)? SameSite defense?
4. CSRF header-based JWT ke saath kyu mostly N/A?
5. Broken Access Control / IDOR — kyu #1? Defense (server-side authZ)?
6. SSRF kya hai (server tricked into internal request)? Defense?
7. Defense checklist ke top 4?

---
**Next:** `09-Data-Protection-And-Secrets.md` — data + secrets ko protect karna.