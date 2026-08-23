# 09 — Data Protection & Secrets

> **Is lesson me:** Data ko protect karna — encryption in transit (TLS recap) aur at rest, secrets management (API keys/passwords kaha rakho), PII handling, aur compliance basics. Ye "data leak = disaster" ko rokta hai.

Prerequisites: `02-Password-Security.md`, Topic 1 lesson 03 (TLS), Topic 6 lesson 03 (config)

---

## 1. Two states of data to protect

```
DATA IN TRANSIT:  network pe move ho raha (client↔server, service↔service)
                  → Encryption via TLS (Topic 1 lesson 03)
DATA AT REST:     stored (disk, DB, backups, S3)
                  → Encryption at rest
```
Dono protect karne padte hai — TLS transit me bachata hai, but agar DB/disk chori ho (ya insider access), at-rest encryption chahiye.

---

## 2. Encryption in Transit (TLS recap)

> **Recap (Topic 1 lesson 03):** TLS data ko encrypt karta hai network pe — koi beech me (man-in-the-middle) padh/badal na sake. HTTPS = HTTP over TLS.
```
- HTTPS everywhere (HTTP se redirect, HSTS header force karta)
- TLS 1.2+ (purane versions insecure)
- Valid certs (expiry monitor — Topic 1 lesson 03 ka classic outage)
- Service-to-service bhi encrypt (mTLS — lesson 07)
```
**Ek line:** external (HTTPS/TLS) + internal (mTLS) — sab traffic encrypted. Plain HTTP kabhi nahi (passwords/tokens plaintext travel karenge).

---

## 3. Encryption at Rest

Stored data ko encrypt karo taaki disk/DB/backup chori ho toh bhi useless ho.
```
LEVELS:
  1. Disk/volume encryption — poora disk encrypted (AWS EBS encryption, LUKS)
     → machine chori/disposed, data safe
  2. Database encryption (TDE — Transparent Data Encryption)
     → DB files encrypted at rest (Postgres/MySQL support)
  3. Application-level / field encryption — specific sensitive fields
     (SSN, credit card) app me encrypt karke store
     → DB admin bhi plaintext na dekhe (highest sensitivity)
```

**Key management (critical):** encryption keys ko **alag** aur **secure** rakho — data ke saath nahi (warna dono chori). **KMS (Key Management Service)** use karo (AWS KMS, GCP KMS, HashiCorp Vault) — keys manage, rotate, audit.
```
❌ Encryption key code/config me (leak = data decryptable)
✅ Key in KMS — app KMS se key maange (audited, rotatable, access-controlled)
```

> **Recap (lesson 02 — pepper):** password pepper bhi ek secret jo alag rehta (KMS/secrets manager). Same principle — secrets data se alag.

---

## 4. Secrets Management ⚠️⭐

**Secrets** = API keys, DB passwords, encryption keys, tokens, certificates — cheezein jo access dete hai.

> **Recap (Topic 6 lesson 03):** secrets **kabhi code/git me nahi**. Externalize karo.
```
❌ NEVER:
  - Secrets in code (git history me forever, repo access = all secrets)
  - Secrets in config files committed to git
  - Secrets in Docker images, logs, error messages, client-side code

✅ ALWAYS:
  - Secrets manager (AWS Secrets Manager, Vault, GCP Secret Manager)
  - Environment variables (injected at runtime, not committed) — 12-factor
  - Rotate regularly (leak ho toh window chhota)
  - Least privilege (har service sirf apne secrets access kare — lesson 01)
  - Audit access (kaun kab secret access kiya)
```

**Secrets manager flow:**
```
App startup → secrets manager se secrets fetch (authenticated) → memory me use
  → secrets kabhi disk/code/git me nahi, sirf runtime memory me
```

> ⚠️ **#1 real-world mistake: secrets in git.** Ek `AWS_SECRET_KEY = "..."` commit ho gaya → git history me forever (delete karne pe bhi history me), aur public repo = bots seconds me steal kar lete. **`.env` ko `.gitignore` me, secrets manager use karo, aur agar galti se commit ho jaye — key IMMEDIATELY rotate karo** (git se hatana kaafi nahi).

**Frontend analogy:** tumne `.env` files use ki hai (API keys), aur unhe `.gitignore` kiya hai. Backend me ye scale pe ek proper secrets manager ban jaata hai — but principle same: secrets code se bahar, git se bahar, runtime pe inject.

---

## 5. PII & Sensitive Data Handling

**PII (Personally Identifiable Information)** = data jo kisi individual ko identify kare (naam, email, phone, SSN, address, location).
```
- Minimize — sirf jo genuinely chahiye collect karo (least data)
- Encrypt sensitive fields (at rest — section 3)
- Mask/redact in logs — kabhi PII/passwords/tokens logs me mat likho
  (log "user_id=5" not "email=bob@... password=...")
- Access control — sirf authorized services/people PII dekhein (least privilege)
- Retention — jab zaroorat na ho, delete karo (data minimization)
- Anonymize/pseudonymize — analytics ke liye identity strip karo
```

> ⚠️ **Logs ek common leak source hai** — debug logs me accidentally passwords, tokens, PII, full request bodies aa jaate hai. **Structured logging with redaction** — sensitive fields explicitly mask karo.

---

## 6. Compliance (awareness)

Regulations data handling govern karte hai — awareness kaafi hai (interview me depth nahi):
```
GDPR (EU):    consent, right to be forgotten (delete), data portability, breach notify
HIPAA (US):   healthcare data protection
PCI-DSS:      payment card data (agar cards handle karo — usually Stripe/PayPal use
              karke SCOPE se bachte ho — tum cards store hi nahi karte)
SOC 2:        security controls audit (B2B SaaS ke liye common)
```
**Practical:** compliance requirements **NFRs** hai (Topic 7 lesson 02) — ye architecture affect karte hai (data residency, encryption, audit logs, deletion capability). E.g., GDPR "right to be forgotten" → tumhe user data delete karne ka mechanism chahiye (event sourcing — Topic 6 lesson 07 — ke saath tension, crypto-shredding se handle).

---

## 7. The data protection checklist

```
1. Transit:   HTTPS/TLS everywhere + mTLS internal (Topic 1, lesson 07)
2. At rest:   disk + DB encryption; field-level for highly sensitive
3. Keys:      KMS (separate from data), rotate, audit
4. Secrets:   secrets manager, NEVER git/code, rotate, least privilege
5. PII:       minimize, encrypt, mask in logs, access control, retention
6. Compliance: treat as NFRs (GDPR deletion, audit logs)
```

**Interview line:** *"Data in transit: TLS everywhere (HTTPS + mTLS internal). At rest: disk + DB encryption, field-level for highly sensitive (SSN/cards), keys in KMS separate from data. Secrets in a secrets manager — never in git/code — rotated, least-privilege. PII minimized, masked in logs, access-controlled. Compliance (GDPR/PCI) treated as NFRs shaping the design."*

---

## Quick Self-Check
1. Data ke 2 states protect karne hai — transit vs rest? Dono kyu (TLS kaafi nahi)?
2. Encryption at rest ke 3 levels (disk/DB/field)?
3. Encryption keys kaha rakho aur kyu (KMS, separate from data)?
4. Secrets kabhi kaha nahi (git/code/logs)? Kaha (secrets manager)?
5. Secret galti se git me commit ho jaye toh (rotate immediately)?
6. PII handling ke practices (minimize, mask logs, encrypt)?
7. Logs security risk kaise ban jaate hai?
8. Compliance ko kaise treat karo (NFRs — Topic 7)?

---
**Next:** `10-Security-Playbook-FastAPI.md` — sab tie together, FastAPI implementation.