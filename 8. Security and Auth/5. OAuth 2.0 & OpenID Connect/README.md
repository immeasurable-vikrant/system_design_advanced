# 05 — OAuth 2.0 & OpenID Connect

> **Is lesson me:** "Login with Google" andar se kaise kaam karta hai — OAuth 2.0 (delegated authorization), uske roles, authorization code flow + PKCE, aur OpenID Connect (OAuth ke upar authentication). Ye ek confusing area hai — clearly samjhenge.

Prerequisites: `04-Token-Based-Auth-JWT.md`

---

## 1. The problem OAuth solves

Socho ek app chahta hai tumhari Google Photos access kare (e.g., print service). Purana (bad) tarika: app tumse Google password maange. **Disaster** — app ke paas tumhara password, full access, forever.

**OAuth 2.0** = ek app ko **limited access** dena doosre service ke tumhare data ka, **bina password share kiye**. Tum Google pe authorize karte ho, Google app ko ek **limited token** deta hai (sirf photos, read-only, revocable).

**Analogy — hotel valet key:** tum valet ko apni **master car key** nahi dete (jo ghar bhi kholti ho). Tum ek **valet key** dete ho — sirf car start + drive, trunk/glovebox nahi. OAuth wahi — limited, scoped, revocable access, master credentials (password) diye bina.

> ⚠️ **OAuth = AUTHORIZATION (kya access kar sakta), NOT authentication.** Ye common confusion hai. OAuth "delegated access" deta hai. "Login with Google" (authentication) ke liye **OpenID Connect** chahiye (OAuth ke upar — section 5).

---

## 2. OAuth roles ⭐

Char players (samajhna zaroori hai):
```
RESOURCE OWNER:        tum (jo data own karta — tumhari photos)
CLIENT:                app jo access chahta (print service)
AUTHORIZATION SERVER:  jo tokens issue karta (Google's auth server)
RESOURCE SERVER:       jaha data hai (Google Photos API)
```

```
Tum (owner) → authorize karo → Auth Server (Google) → token de → Client (app)
                                                                    │
                                                    token se → Resource Server (Photos API)
```

---

## 3. Authorization Code Flow ⭐ (the main flow)

Sabse common + secure flow (web apps, with PKCE for SPAs/mobile):
```
1. User "Connect Google Photos" click kare (on the app/client)
2. App user ko Google (auth server) pe redirect kare:
   "ye app photos.read access maang raha hai — allow?"
   (redirect with: client_id, scope=photos.read, redirect_uri)
3. User Google pe login kare (agar nahi hai) + "Allow" kare
4. Google user ko app pe wapas redirect kare ek AUTHORIZATION CODE ke saath
   (short-lived, one-time code — not the token yet)
5. App (backend) ye code + apna client_secret Google ko bheje
   "ye code, mujhe token do" (server-to-server, secure)
6. Google ACCESS TOKEN de (+ refresh token)
7. App token se Photos API call kare (scoped to photos.read)
```

**Kyu 2 steps (code phir token)?** Authorization code browser se guzarta hai (redirect — exposed). Actual token exchange **backend-to-backend** hota hai (client_secret ke saath, hidden). Isse token kabhi browser/URL me expose nahi hota.

> **PKCE (Proof Key for Code Exchange):** SPAs/mobile apps ke paas securely `client_secret` rakhne ki jagah nahi (frontend code visible). PKCE add karta hai — client ek random "code_verifier" banata hai, uska hash ("code_challenge") step 2 me bhejta hai, aur step 5 me original verifier bhejta hai. Isse intercepted code useless ho jaata hai (verifier ke bina). **Public clients (SPA/mobile) ke liye PKCE mandatory hai ab.**

**Frontend analogy:** tumne "Login with Google/GitHub" implement kiya hoga — wo redirect → consent → callback with code → backend exchanges for token — exactly ye flow hai. Ab andar ka mechanism clear hai.

---

## 4. OAuth grant types (flows) — quick

| Grant type | Kab |
|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile (the standard, most secure) |
| **Client Credentials** | Service-to-service (no user — app apni identity se, lesson 07) |
| Device Code | TVs, CLI, limited-input devices |
| ~~Implicit~~ | Deprecated (token in URL — insecure) |
| ~~Password~~ | Deprecated (app handles password — defeats the purpose) |

**Interview:** *"Authorization code flow with PKCE for user-facing apps (web/mobile/SPA). Client credentials for service-to-service. Implicit aur password grants deprecated — insecure."*

---

## 5. OpenID Connect (OIDC) — authentication on top ⭐

OAuth authorization deta hai (access to data). But "**Login** with Google" me tum **authentication** chahte ho (kaun ye user hai). **OpenID Connect** = ek thin layer OAuth ke upar jo authentication add karta hai.

```
OAuth 2.0:      "ye app tumhari photos access kar sakta" (authorization)
OIDC (on top):  "+ yahan is user ki identity hai" (authentication)
                → ek ID TOKEN (JWT) deta hai with user info (sub, email, name)
```

```
OIDC flow = OAuth authorization code flow + ek extra "openid" scope
  → response me ACCESS token (OAuth) + ID token (OIDC — a JWT with identity)
  ID token: { sub: "google_user_123", email: "...", name: "...", ... }
  → app ID token se user ko authenticate kare ("ye Bob hai, verified by Google")
```

**Ek line me:**
- **OAuth** → "app kya access kar sakta hai" (authorization, access token)
- **OIDC** → "ye user kaun hai" (authentication, ID token — a JWT)
- "Login with Google/GitHub/Apple" = OIDC (built on OAuth)

**Frontend analogy:** jab tum "Sign in with Google" use karte ho aur app ko tumhara naam/email/photo mil jaata hai — wo **ID token (OIDC)** hai. Jab app tumhare Google Calendar ko access kare — wo **access token (OAuth)** hai. Do alag tokens, do alag purposes.

---

## 6. Why use OAuth/OIDC (benefits)

```
- No password handling — app kabhi user ka Google password nahi dekhta (safer)
- Delegated + scoped — limited access (sirf jo chahiye), revocable
- SSO (Single Sign-On) — ek Google account se kai apps login
- Trust — identity Google/Apple verify karta (tumhe password infra nahi banana)
- Standard — well-tested, libraries available
```

**Interview line:** *"User login ke liye OIDC (OAuth ke upar) — 'Login with Google', ID token se identity. Third-party data access ke liye OAuth authorization code flow + PKCE, scoped tokens, no password sharing. Service-to-service ke liye client credentials grant. Ye password handling avoid karta, scoped + revocable access deta, aur SSO enable karta."*

---

## Quick Self-Check
1. OAuth kya problem solve karta hai (limited access, no password share)? Valet key analogy?
2. OAuth authentication hai ya authorization? (trap!)
3. OAuth ke 4 roles (owner/client/auth-server/resource-server)?
4. Authorization code flow ke steps? Code phir token — 2 steps kyu?
5. PKCE kya hai aur kis clients ke liye (SPA/mobile) zaroori?
6. Client credentials grant kab (service-to-service)?
7. OIDC OAuth ke upar kya add karta hai (authentication, ID token)?
8. "Login with Google" — OAuth ya OIDC? Access token vs ID token?

---
**Next:** `06-Authorization-Models.md` — permissions kaise model karo.