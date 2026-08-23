# 04 — Token-Based Auth & JWT

> **Is lesson me:** Stateless authentication — JWT (JSON Web Token) ka anatomy, signing, kyu stateless, access + refresh tokens, revocation problem, aur session vs JWT decision. Ye modern APIs/microservices ka common auth approach hai.

Prerequisites: `03-Session-Based-Authentication.md`

---

## 1. The idea — self-contained tokens

Session-based me server session state rakhta hai (lesson 03). **Token-based (JWT)** ise flip karta hai: token khud **saari info** rakhta hai (signed), toh server ko kuch store nahi karna — **stateless.**

```
Session:  cookie me opaque ID → server store me dekho → user data
JWT:      token me user data KHUD hai (signed) → server bas signature verify kare
          → no lookup, no store (stateless — Topic 1 lesson 04)
```

**Analogy:** session = ek coat-check ticket (number, jo counter pe dekhna padta hai kaunsa coat). JWT = ek tamper-proof ID badge jispe saari info likhi hai (naam, role, expiry) + ek hologram (signature) jo prove kare fake nahi. Badge dekh ke hi sab pata — counter pe check karne ki zarurat nahi.

---

## 2. JWT anatomy ⭐

Ek JWT teen parts, dots se separated: `header.payload.signature`
```
eyJhbGc...  .  eyJzdWI...  .  SflKxwRJ...
  HEADER         PAYLOAD       SIGNATURE
```

Har part base64url-encoded:
```
HEADER (algorithm + type):
  { "alg": "HS256", "typ": "JWT" }

PAYLOAD (claims — the data):
  { "sub": "user_5", "role": "admin", "exp": 1735689600, "iat": 1735686000 }
  sub=subject(user), exp=expiry, iat=issued-at, + custom claims

SIGNATURE:
  HMAC-SHA256(base64(header) + "." + base64(payload), SECRET_KEY)
  → ye prove karta hai token tamper nahi hua aur trusted server ne banaya
```

> ⚠️ **CRITICAL: JWT payload ENCRYPTED NAHI hai — sirf ENCODED (base64).** Koi bhi payload decode karke padh sakta hai (jwt.io pe paste karo, sab dikhega). **Isliye JWT me sensitive data (password, secrets, PII) KABHI mat daalo.** Signature tampering rokti hai (integrity), but confidentiality nahi deti. Payload = public-readable.

**Signature kaise protect karti hai:**
```
Attacker payload badalta hai ("role": "user" → "admin")
  → but signature secret key ke bina regenerate nahi kar sakta
  → server signature verify kare → mismatch → REJECT
Secret key sirf server ke paas → attacker valid token forge nahi kar sakta
```

---

## 3. Signing — HMAC vs RSA

```
HS256 (HMAC — symmetric):
  Same SECRET key se sign aur verify. Simple, fast.
  Sab services ko secret share karna padta (agar multiple verify karein).

RS256 (RSA — asymmetric):
  PRIVATE key se sign (sirf auth server), PUBLIC key se verify (koi bhi).
  Better for microservices — auth server private key rakhe, baaki services
  public key se verify karein (secret share nahi karna). ⭐ for distributed.
```
> **Recap (Topic 1 lesson 03 — TLS):** asymmetric crypto (private sign / public verify) — wahi concept JWT signing me. Microservices me RS256 preferred (secret ek jagah).

---

## 4. JWT auth flow

```
LOGIN:
  1. User email + password → server verify (bcrypt)
  2. Server JWT banaye: sign({user_id, role, exp}, SECRET)
  3. Token client ko de (response body, ya cookie)

SUBSEQUENT REQUESTS:
  1. Client token bheje: Authorization: Bearer eyJhbGc...
  2. Server signature verify kare (SECRET/public key se) + exp check
     Valid → payload se user_id/role nikaalo → authenticated (NO DB lookup!)
     Invalid/expired → 401
  3. Request process
```

**FastAPI:**
```python
from jose import jwt   # python-jose

# create
token = jwt.encode({"sub": "user_5", "role": "admin", "exp": ...}, SECRET, algorithm="HS256")

# verify (in a dependency — lesson 10)
payload = jwt.decode(token, SECRET, algorithms=["HS256"])   # raises if invalid/expired
user_id = payload["sub"]
```

---

## 5. The revocation problem ⚠️ (JWT's big weakness)

Session-based me logout = session delete (instant revoke). But JWT **stateless** hai — server kuch store nahi karta, toh ek issued token ko **revoke karna mushkil** hai. Token expiry tak valid rehta hai, chahe user logout kare ya compromise ho.
```
Problem: user ka token chori ho gaya → attacker use kar sakta jab tak exp na aaye
         User "logout" kare → but token abhi bhi valid (server ko yaad nahi)
```

**Solutions:**
```
1. SHORT-LIVED access tokens (~15 min exp) + REFRESH tokens (section 6)
   → chori hua token jaldi expire; damage window chhota
2. DENYLIST (blocklist) — revoked tokens ka ek list (Redis) maintain karo
   → har request pe check "ye token denylist me?" (but ye stateful bana deta — JWT ka point kho jaata thoda)
3. Token versioning — user record me a "token_version"; JWT me bhi;
   mismatch → reject (password change pe version bump = all tokens invalid)
```

---

## 6. Access + Refresh tokens ⭐ (the standard pattern)

Revocation problem ko balance karne ka standard tarika:
```
ACCESS TOKEN:  short-lived (~15 min), used for every API request
               → chori ho toh jaldi expire (small damage window)
REFRESH TOKEN: long-lived (~7-30 days), stored securely (HttpOnly cookie),
               used ONLY to get a new access token
               → revocable (stored server-side / denylist), rarely transmitted

FLOW:
  Login → access token (15min) + refresh token (30 days)
  API calls → access token
  Access token expire → refresh token se naya access token maango
  Logout / compromise → refresh token revoke (server-side) → no new access tokens
```
Isse: fast stateless verification (access token, no lookup) + revocability (refresh token, server-tracked). Best of both.

---

## 7. Session vs JWT — the decision ⭐

| | Session-based | JWT (token) |
|---|---|---|
| State | Stateful (server store) | Stateless (self-contained) |
| Storage | Redis/DB per session | None (or refresh tokens only) |
| Revocation | Instant (delete session) | Hard (expiry-bound / denylist) |
| Scaling | Shared session store needed | No store (access tokens) |
| Data exposure | Opaque ID (nothing leaked) | Payload readable (no secrets!) |
| Cross-domain/mobile | Awkward (cookies) | Easy (Authorization header) |
| Best for | Web apps, need instant revoke | APIs, microservices, mobile, SPAs |

**Interview line:** *"Web app with server-rendered pages → session-based (Redis store, HttpOnly cookies) — simpler, instant revocation. APIs/mobile/microservices → JWT with short-lived access tokens + refresh tokens — stateless verification (no lookup per request), refresh token revocable. JWT me kabhi sensitive data nahi (payload readable). Microservices me RS256 (services public key se verify karein)."*

> **Common hybrid:** access token = JWT (stateless, fast), refresh token = opaque + stored server-side (revocable). Best of both.

---

## Quick Self-Check
1. JWT session se kaise alag (self-contained vs server store)? Coat-check vs ID-badge analogy?
2. JWT ke 3 parts? Payload encrypted hai ya encoded? Implication?
3. Signature kaise tampering rokti hai (secret key)?
4. HS256 vs RS256 — kab kaunsa (microservices)?
5. JWT ka revocation problem kya hai, aur 2 solutions?
6. Access + refresh token pattern — dono ka role aur expiry?
7. Session vs JWT — kab kaunsa (web vs API/mobile)?

---
**Next:** `05-OAuth-And-OpenID-Connect.md` — "Login with Google" andar se.