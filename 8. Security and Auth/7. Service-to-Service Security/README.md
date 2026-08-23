# 07 — Service-to-Service Security

> **Is lesson me:** Services ek dusre ko kaise securely authenticate karein — API keys, mTLS, service accounts (OAuth client credentials), aur zero-trust. User auth (lessons 03-05) alag hai; ye machine-to-machine (M2M) auth hai, jo microservices me critical hai (Topic 6).

Prerequisites: `04-Token-Based-Auth-JWT.md`, Topic 1 lesson 06 (service mesh)

---

## 1. The problem — services trusting services

Microservices me (Topic 6), Order Service ko Payment Service call karni hai. Payment Service kaise jaane "ye legit Order Service hai, koi attacker nahi"? User auth (password/JWT) yahan fit nahi — koi user nahi, do machines hai. Chahiye **machine-to-machine (M2M) authentication.**

> ⚠️ **Common mistake:** internal services ko "trusted" maan lena ("hamare network me hai toh safe"). Ye galat hai — agar ek service compromise ho (ya network breach), attacker freely doosri services ko call kar sakta. **Zero-trust:** har service call authenticated ho, chahe internal ho (section 5).

---

## 2. API Keys — simplest M2M

Ek service ko ek **secret key** do; wo har request me bheje; receiver verify kare.
```
Order Service → Payment Service
  Header: X-API-Key: sk_live_abc123...
Payment Service: key valid hai? known service ka? → allow/deny
```
**Pros:** simple, easy to implement.
**Cons:** static (rotate karna manual), agar leak ho toh full access (jab tak rotate na karo), no built-in expiry, transit me protect karna padta (TLS zaroori).

**Kab:** third-party API access (Stripe/Twilio tumhe API key dete hai), simple internal M2M, external developers.

**Best practices:** keys secrets manager me (lesson 09, never in code/git), rotate regularly, scope them (per-service, limited permissions — least privilege), TLS always (key transit me expose na ho).

**Frontend analogy:** tumne kisi API (Stripe, Google Maps) ka API key use kiya hoga. Tum wo key backend me rakhte ho (frontend me nahi — leak ho jaayega). Same principle internal services me.

---

## 3. mTLS (Mutual TLS) ⭐ — the strong option

> **Recap (Topic 1 lesson 03):** normal TLS me sirf **server** apna certificate deta hai (client server ko verify karta). **mTLS** = **dono** taraf certificates — client bhi apna cert deta hai, dono ek dusre ko verify karte hai.
```
Normal TLS:  client → verifies server's cert (server ki identity)
mTLS:        client ↔ server, DONO ek dusre ka cert verify karte hai
             → dono cryptographically prove karte hai "main jo hu wahi hu"
```

Service-to-service me: har service ka apna certificate (identity). Payment Service Order Service ke cert se verify kare "ye genuinely Order Service hai." Attacker ke paas valid cert nahi → reject.

**Pros:** very strong (cryptographic identity, hard to forge), no shared secrets (each has own cert), mutual (both verified).
**Cons:** cert management complex (issue, rotate, revoke — needs a CA/PKI).

> **Recap (Topic 1 lesson 06 — service mesh):** service mesh (Istio + Envoy sidecars) **mTLS automatically** handle karta hai — har service ke sidecar cert manage karte hai, saara service-to-service traffic auto-mTLS. Ye zero-trust ka standard implementation hai — developers ko manually kuch nahi karna. **Isliye mTLS + service mesh modern microservices ka answer hai.**

---

## 4. Service Accounts (OAuth Client Credentials) ⭐

> **Recap (lesson 05):** OAuth client credentials grant — no user, app apni identity se token leta.
```
Service (client) → Auth Server: "main Order Service (client_id + client_secret), token do"
Auth Server → verify → JWT access token (scoped to what Order Service can do)
Order Service → Payment Service: Authorization: Bearer <token>
Payment Service → token verify (signature + scope) → allow
```
Ye user JWT (lesson 04) jaisa hi hai, but user ke bajaye ek **service identity** (service account) ke liye. Token scoped + expiring (better than static API key).

**Pros:** scoped + expiring tokens (better than static keys), centralized (auth server), works with existing OAuth infra.
**Kab:** microservices with an auth server, cloud (AWS IAM roles, GCP service accounts ye pattern use karte).

---

## 5. Zero-Trust Architecture ⭐

**Zero-trust** = "never trust, always verify" — koi request trusted nahi, chahe internal network se ho. Har service call authenticated + authorized, har jagah.
```
Old (perimeter security): "firewall ke andar = trusted" (castle-and-moat)
  → ek breach = attacker freely andar ghoome

Zero-trust: har request verify, chahe internal
  → mTLS between all services (identity)
  → authZ on every call (least privilege)
  → encrypt everything (transit + rest)
  → ek breach = attacker abhi bhi har service pe blocked (no free movement)
```
> **Recap (Topic 1 lesson 06):** service mesh zero-trust ko implement karta hai — har pod ka sidecar mTLS + policy enforce karta. Network compromise ho toh bhi attacker eavesdrop/impersonate nahi kar sakta.

**Frontend analogy:** ye "assume the network is hostile" mindset hai — bilkul jaise tum client ko trust nahi karte (lesson 01), zero-trust me internal services ko bhi implicitly trust nahi karte. Har cheez verified.

---

## 6. Comparison + decision

| Method | Strength | Complexity | Best for |
|---|---|---|---|
| **API Keys** | Medium | Low | Third-party, simple M2M |
| **mTLS** | High | High (certs) | Internal microservices (via mesh) |
| **Service accounts (OAuth CC)** | High | Medium | Microservices with auth server, cloud |

```
Third-party / external devs         → API keys (scoped, rotated)
Internal microservices              → mTLS (via service mesh) — zero-trust
With an OAuth auth server / cloud    → service accounts (client credentials)
```

**Interview line:** *"Third-party access ke liye scoped, rotatable API keys (secrets manager, TLS). Internal microservices ke liye mTLS via service mesh (Istio) — mutual cryptographic identity, automatic, zero-trust (no implicit internal trust). Ya OAuth client credentials for scoped, expiring service tokens. Never treat internal traffic as trusted — every call authenticated + least-privilege authorized."*

---

## Quick Self-Check
1. Service-to-service auth user auth se kyu alag (no user, machine-to-machine)?
2. Internal services ko "trusted" maan lena kyu galat (zero-trust)?
3. API keys — pros/cons? Best practices (secrets manager, rotate, scope)?
4. mTLS normal TLS se kaise alag (dono verify)? Service mesh ka role (Topic 1)?
5. Service accounts (client credentials) — user JWT se kaise similar/alag?
6. Zero-trust kya hai — perimeter security se kaise alag?
7. Third-party vs internal microservices — kaunsa method?

---
**Next:** `08-Common-Vulnerabilities.md` — the attacks you must defend against.