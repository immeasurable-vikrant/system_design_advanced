# 10 — Security Playbook & FastAPI Implementation

> **Is lesson me:** Sab kuch ek saath — ek security checklist/playbook, aur FastAPI me auth concretely kaise implement karo (password hashing, JWT, OAuth2 dependency, authZ, CORS, rate limiting). Ye poore module ka synthesis + tumhare actual build ka reference hai.

Prerequisites: **saare pichle lessons (01-09), Topic 7 lesson 08 (FastAPI stack).**

---

## 1. The security decision playbook

```
STEP 1 — AUTHENTICATION (kaun ho? — lessons 03-05)
  Web app (server-rendered)?  → sessions + HttpOnly cookies (Redis store)
  API / mobile / SPA?         → JWT (access + refresh tokens)
  Third-party login?          → OAuth2 + OIDC ("Login with Google")
  Passwords?                  → bcrypt/argon2 (lesson 02), never plaintext

STEP 2 — AUTHORIZATION (kya kar sakte ho? — lesson 06)
  → RBAC default; ABAC if context matters; server-side, every request
  → Never trust client-sent roles/IDs (IDOR — lesson 01, 08)

STEP 3 — SERVICE-TO-SERVICE (lesson 07)
  → mTLS (service mesh) internal; API keys / client credentials external
  → Zero-trust: no implicit internal trust

STEP 4 — VULNERABILITIES (lesson 08)
  → Parameterized queries, output encoding, SameSite cookies, server-side authZ

STEP 5 — DATA (lesson 09)
  → TLS transit + encryption at rest, secrets manager, PII masked

STEP 6 — DEFENSE IN DEPTH (lesson 01)
  → Rate limiting, monitoring, least privilege, MFA
```

---

## 2. FastAPI auth — complete implementation ⭐

### Password hashing (lesson 02)
```python
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(p: str) -> str:      return pwd_context.hash(p)
def verify_password(p, h) -> bool:     return pwd_context.verify(p, h)
```

### JWT create + verify (lesson 04)
```python
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError

SECRET = settings.jwt_secret          # from secrets manager (lesson 09), NOT hardcoded
ALGO = "HS256"                        # RS256 for microservices (lesson 04)

def create_access_token(user_id: str, role: str) -> str:
    payload = {
        "sub": user_id, "role": role,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=15),   # short-lived
        "iat": datetime.now(timezone.utc),
    }
    return jwt.encode(payload, SECRET, algorithm=ALGO)
```

### The auth dependency — `get_current_user` (lesson 01, 04) ⭐
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")   # reads Bearer header + Swagger UI

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    creds_error = HTTPException(status.HTTP_401_UNAUTHORIZED,   # 401 = not authenticated
                               detail="Invalid credentials",
                               headers={"WWW-Authenticate": "Bearer"})
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGO])   # verify signature + exp
        user_id = payload.get("sub")
        if user_id is None:
            raise creds_error
    except JWTError:
        raise creds_error
    user = await fetch_user(user_id)     # optional: DB lookup for fresh data / revocation check
    if user is None:
        raise creds_error
    return user
```

### Login endpoint (lesson 02, 04)
```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/auth/login")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = await fetch_user_by_email(form.username)
    if not user or not verify_password(form.password, user.password_hash):
        raise HTTPException(401, "Invalid credentials")   # same error for both (don't leak which)
    return {"access_token": create_access_token(user.id, user.role),
            "token_type": "bearer"}
```

### Authorization — protecting endpoints (lesson 06) ⭐
```python
# AuthN: just require a valid user
@app.get("/profile")
async def profile(user: User = Depends(get_current_user)):
    return user

# AuthZ: require a role (RBAC — lesson 06)
def require_role(role: str):
    async def checker(user: User = Depends(get_current_user)):
        if user.role != role:
            raise HTTPException(403, "Insufficient permissions")   # 403 = authenticated but not allowed
        return user
    return checker

@app.delete("/admin/users/{id}")
async def delete_user(id: int, admin: User = Depends(require_role("admin"))):
    ...   # only admins reach here

# IDOR protection (lesson 08) — check ownership, never trust the ID alone
@app.get("/orders/{order_id}")
async def get_order(order_id: int, user: User = Depends(get_current_user)):
    order = await fetch_order(order_id)
    if order.user_id != user.id:              # ← server-side ownership check!
        raise HTTPException(403, "Not your order")
    return order
```

> **Ye poora auth system dependency injection pe khada hai** (Topic 7) — `Depends(get_current_user)` har protected endpoint pe. Clean, reusable, testable. FastAPI ka Swagger UI bhi auto "Authorize" button dikhata hai.

---

## 3. FastAPI security middleware & config

### CORS (lesson 08 — cross-origin)
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],   # SPECIFIC origins, NOT ["*"] in prod!
    allow_credentials=True,                # cookies allowed
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```
> ⚠️ **`allow_origins=["*"]` + `allow_credentials=True` = insecure/invalid.** Prod me specific origins list karo. CORS browser-enforced hai (server protection nahi) — but misconfigure karke tum apni hi API expose kar dete ho.

### Input validation (lesson 08 — injection) — Pydantic (free!)
```python
from pydantic import BaseModel, EmailStr, field_validator

class SignupRequest(BaseModel):
    email: EmailStr                        # auto-validates email format
    password: str
    @field_validator("password")
    def strong(cls, v):
        if len(v) < 12: raise ValueError("Password too short")
        return v
# Invalid input → auto 422, never reaches your logic (validation layer — lesson 08)
```

### Rate limiting (Topic 3, lesson 02) — slowapi
```python
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address, storage_uri="redis://...")   # distributed

@app.post("/auth/login")
@limiter.limit("5/minute")     # brute-force protection (lesson 02)
async def login(...): ...
```

### HTTPS + secrets (lesson 09)
```python
# HTTPS: TLS terminate at proxy/LB (Topic 1 lesson 06), FastAPI behind it
#   → trust proxy headers (X-Forwarded-Proto), redirect HTTP→HTTPS at edge
# Secrets: pydantic-settings from env / secrets manager (NEVER hardcode)
from pydantic_settings import BaseSettings
class Settings(BaseSettings):
    jwt_secret: str            # loaded from env var (injected at runtime)
    database_url: str
    class Config: env_file = ".env"    # .env in .gitignore! (lesson 09)
```

---

## 4. The security checklist (production-ready)

```
AUTHENTICATION
  □ Passwords: bcrypt/argon2, never plaintext (lesson 02)
  □ JWT: short-lived access + refresh tokens; no secrets in payload (lesson 04)
  □ Sessions: HttpOnly + Secure + SameSite cookies, Redis store (lesson 03)
  □ MFA for sensitive accounts (lesson 02)

AUTHORIZATION
  □ Server-side authZ on EVERY protected endpoint (lesson 01, 06)
  □ Ownership checks (IDOR protection) — never trust client IDs (lesson 08)
  □ Least privilege (roles, service permissions) (lesson 01, 06)

INPUT & OUTPUT
  □ Parameterized queries (SQLAlchemy/Pydantic) — no injection (lesson 08)
  □ Output encoding + CSP (XSS) (lesson 08)
  □ SameSite cookies / header-auth (CSRF) (lesson 08)
  □ Input validation (Pydantic) (lesson 08)

TRANSPORT & DATA
  □ HTTPS everywhere + mTLS internal (lesson 07, 09)
  □ Encryption at rest + KMS (lesson 09)
  □ Secrets in secrets manager, never git (lesson 09)
  □ PII minimized + masked in logs (lesson 09)

OPERATIONS
  □ Rate limiting (brute-force + abuse) (Topic 3)
  □ CORS locked to specific origins
  □ Dependency audits (pip audit)
  □ Security logging + monitoring (Topic 9!)
  □ Least-privilege DB/service accounts
```

---

## 5. Interview synthesis

**Interview line (full):** *"Authentication: JWT with short-lived access + refresh tokens for the API, bcrypt-hashed passwords, OIDC for social login. Authorization: RBAC, enforced server-side on every request via a FastAPI dependency, with ownership checks to prevent IDOR — never trusting client-sent IDs. Service-to-service: mTLS via service mesh, zero-trust. Defense in depth: parameterized queries, Pydantic validation, SameSite cookies, TLS + encryption at rest, secrets in a manager (never git), rate limiting, and security monitoring. Every layer independent so one failure doesn't cascade."*

---

## Quick Self-Check (capstone)
1. Security playbook ke 6 steps (authN → defense in depth)?
2. FastAPI me password hashing kaise (passlib)?
3. `get_current_user` dependency kya karta hai (token verify → user)?
4. RBAC endpoint protection FastAPI me kaise (`require_role`)?
5. IDOR protection code me kaise (ownership check)?
6. CORS `allow_origins=["*"]` + credentials kyu insecure?
7. Pydantic injection/validation me kaise help karta?
8. Security checklist ke top items (authN/authZ/data)?

---

**🎉 Topic 8 complete — aur 8-topic curriculum!**

Tumne ab system ko **secure** banana bhi seekh liya — authentication (sessions/JWT/OAuth), authorization (RBAC/ABAC), service security (mTLS/zero-trust), vulnerabilities (OWASP), data protection (encryption/secrets), aur FastAPI me concrete implementation.

**8 topics:**
1. Core Infrastructure · 2. Database & Storage · 3. Caching & Performance · 4. Communication & Messaging · 5. Distributed Systems · 6. Patterns & Architecture · 7. Architecture Decisions · 8. Security & Auth

Ab tum ek system design **end-to-end** — infra se data se performance se communication se distributed-theory se patterns se decisions se **security** tak — sab cover kar sakte ho. Interview me "how do you secure this?" ab tumhare liye ek strength hai, weakness nahi. 🔐🚀