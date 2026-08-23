# 03 — Session-Based Authentication

> **Is lesson me:** Login ke baad user ko "yaad" kaise rakho — session-based auth, cookies, cookie security flags (HttpOnly/Secure/SameSite), session store, aur logout. Ye classic, stateful auth approach hai (JWT lesson 04 me — comparison bhi).

Prerequisites: `02-Password-Security.md`, Topic 1 lesson 04 (stateless)

---

## 1. The problem — HTTP is stateless

HTTP stateless hai (Topic 1 lesson 04) — har request independent, server ko "yaad" nahi ki tum kaun ho. Login ke baad, agli request pe server ko kaise pata chale "ye wahi logged-in user hai"? Har request pe password bhejna? Nahi (insecure + annoying).

**Solution:** login ek baar, phir server ek **session** banaye aur user ko ek **identifier** de jo wo har request pe bheje. Do approaches: session-based (ye lesson) aur token-based (lesson 04).

---

## 2. Session-based auth — how it works

```
LOGIN:
  1. User email + password bheje
  2. Server verify kare (bcrypt — lesson 02)
  3. Server ek SESSION banaye — ek random session_id + user data server-side store kare
     Session Store: session_id "abc123" → { user_id: 5, role: "user", expires: ... }
  4. Server session_id ko ek COOKIE me bhejke browser ko de
     Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax

SUBSEQUENT REQUESTS:
  1. Browser AUTOMATICALLY cookie bhejta hai (har request pe)
     Cookie: session_id=abc123
  2. Server session store me "abc123" dekhe → user_id=5 mila → authenticated
  3. Request process kare us user ke liye

LOGOUT:
  Server session store se "abc123" delete kare → cookie ab useless
```

**Key:** session **data server-side** hai (session store me); cookie me sirf ek **opaque ID** hai (koi sensitive data nahi). Ye "stateful" auth hai — server session state rakhta hai.

**Frontend analogy:** tumne cookies dekhi hai (`document.cookie`, DevTools → Application → Cookies). Session-based auth me wo `session_id` cookie hi tumhe logged-in rakhti hai. Browser use automatically har request pe bhejta hai — isliye tum "logged in" rehte ho bina kuch kiye.

---

## 3. Cookies — the security flags ⭐

Cookie security in flags pe depend karti hai. Ye interview + practical dono me critical:

```
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax; Max-Age=3600
```

| Flag | Kya karta hai | Kis attack se bachata |
|---|---|---|
| **HttpOnly** | JavaScript cookie access nahi kar sakta (`document.cookie` se hidden) | **XSS** (lesson 08) — injected script cookie chura na sake |
| **Secure** | Cookie sirf HTTPS pe bheji jaaye (plain HTTP pe nahi) | **Man-in-the-middle** (network sniffing) |
| **SameSite** | Cross-site requests pe cookie bheje ya nahi (Strict/Lax/None) | **CSRF** (lesson 08) — attacker site se authenticated request na bane |
| **Max-Age/Expires** | Cookie kab expire | Stale sessions limit |
| **Domain/Path** | Kaunse domain/path pe cookie valid | Scope limit |

> ⚠️ **HttpOnly is critical:** bina iske, ek XSS attack (lesson 08) `document.cookie` se session_id chura ke user ka account hijack kar sakta hai (session hijacking). HttpOnly = JS cookie ko chhu bhi nahi sakta.

> **SameSite values:**
> - `Strict` — cookie sirf same-site requests pe (max CSRF protection, but external links se aane pe logged-out dikhta)
> - `Lax` — top-level navigation pe bheji jaaye (good default — CSRF protection + usability)
> - `None` — hamesha bheji (cross-site allowed, `Secure` mandatory) — sirf jab genuinely cross-site chahiye

**Frontend analogy:** tumne CORS aur cookie issues debug kiye honge. `SameSite`, `Secure`, `HttpOnly` wahi flags hai jo tumhare auth cookies pe lagti hai — ab pata hai har ek kis attack se bachata hai.

---

## 4. Session store — kaha rakho

Session data server-side kaha store ho?
```
In-memory (single server):  fast, but server restart = sab logged out;
                            + doesn't work with multiple servers (Topic 1 lesson 04!)
Database:                   persistent, but slower (DB hit per request)
Redis (recommended) ⭐:     fast (in-memory — Topic 3), shared across servers,
                            TTL for auto-expiry, survives app restarts
```

> **Recap (Topic 1 lesson 04 — stateless):** yaad hai "stateless servers" — session server ki memory me rakhoge toh load balancer ke peeche multiple servers pe kaam nahi karega (user Server A pe login, Server B pe request → session nahi milta). **Fix: session Redis me** (shared) → servers stateless → horizontal scale. Ye wahi principle hai.

---

## 5. Session-based ke pros/cons

**Pros:**
- Server-side control — logout instant (session delete → done), revoke easy
- Cookie me sensitive data nahi (bas opaque ID)
- Well-understood, secure with right flags

**Cons:**
- **Stateful** — server ko session store maintain karna padta hai (Redis lookup per request)
- Scaling ke liye shared session store chahiye (Redis)
- Cross-domain/mobile thoda awkward (cookies browser-centric)

Ye cons hi JWT (lesson 04) ko motivate karte hai — stateless alternative. But session-based **still excellent** aur often simpler/safer.

---

## Quick Self-Check
1. HTTP stateless hai — login ke baad user kaise "yaad" rehta hai (session + cookie)?
2. Session-based auth me data kaha (server-side), cookie me kya (opaque ID)?
3. HttpOnly, Secure, SameSite — teeno kis attack se bachate hai?
4. HttpOnly critical kyu (session hijacking)?
5. SameSite Strict vs Lax — difference?
6. Session store kaha rakho aur kyu (Redis — Topic 1 stateless)?
7. Session-based ke pros/cons (stateful)?

---
**Next:** `04-Token-Based-Auth-JWT.md` — stateless auth with JWT.