# 02 — Password Security

> **Is lesson me:** Passwords ko safely store karna — kyu plaintext/encryption dono galat hai, hashing (bcrypt/argon2), salt aur pepper, kyu slow hashing acchi hai, aur breach handling. Ye ek "galti = disaster" area hai (password breaches sabse common + damaging).

Prerequisites: `01-Security-Fundamentals-AuthN-vs-AuthZ.md`

---

## 1. Passwords ko KABHI aise store mat karo

```
❌ PLAINTEXT:   password = "hunter2"
   Breach → saare passwords exposed. Users doosri sites pe bhi wahi password
   use karte hai → cascade disaster. NEVER.

❌ ENCRYPTED:   password = encrypt("hunter2", key)
   "Encryption reversible hai" — agar key leak (aur breach me aksar hoti hai),
   saare passwords decrypt. Aur tumhe original password kabhi chahiye hi nahi!
```

**Key insight:** tumhe user ka **actual password kabhi nahi chahiye.** Tumhe sirf verify karna hai "kya login pe diya password wahi hai jo signup pe tha." Iske liye **hashing** perfect hai (one-way).

---

## 2. Hashing — the right way ⭐

**Hash function** (Topic 2 lesson 11 recap) = input → fixed output, **one-way** (output se input nahi nikal sakte), deterministic.
```
Signup:  hash("hunter2") = "a3f9c2..." → DB me ye HASH store karo (password nahi)
Login:   hash(entered_password) == stored_hash?  → match → authenticated
         (kabhi original password store/compare nahi kiya)
```
Breach me attacker ko **hashes** milte hai, passwords nahi. Hash se original nikaalna (ideally) impossible.

**Frontend analogy:** ye ek content-hash jaisa hai (Git commit hash, file checksum) — same input → same hash, but hash se content reconstruct nahi kar sakte. Passwords ke liye wahi one-way property use karte hai.

---

## 3. But NOT MD5/SHA — the speed problem ⚠️

Normal hashes (MD5, SHA-256) **bahut fast** hai — aur yehi problem hai. Attacker breach ke baad **billions of guesses/sec** kar sakta hai (GPU pe):
```
SHA-256: ~10 billion hashes/sec on a GPU
→ common passwords ("password123") turant crack
→ "rainbow tables" (precomputed hash→password maps) instant lookup
```

**Solution — deliberately SLOW hashing:** password-specific algorithms jo **intentionally slow** hai (aur tunable):
| Algorithm | Notes |
|---|---|
| **Argon2id** | Current best (2015 winner), memory-hard, recommended default |
| **bcrypt** | Battle-tested, widely used, good default (cost factor tunable) |
| **scrypt** | Memory-hard, good |
| PBKDF2 | Older, acceptable (many iterations) |

Ye ~100ms per hash lete hai (fast enough for one login, but attacker ke billions-of-guesses ko **impossibly slow** bana dete hai — 100ms × billions = years).

> **"Work factor" / cost:** ye algorithms tunable hai — hardware fast hota jaaye toh cost badha do (e.g., bcrypt cost 12 → 14). Ek login pe ~100-250ms rakho — user ko farak nahi, attacker ko dinosaur-slow.

---

## 4. Salt — rainbow tables ko todo ⭐

**Problem:** agar do users ka same password ho, unke hashes same honge → attacker ek crack kare toh dono pata. Aur precomputed rainbow tables kaam karti hai.

**Salt** = ek **random unique value** per password, jo hash se pehle password ke saath mila di jaati hai:
```
User A: hash("hunter2" + salt_A="x7f2..") = "abc..."
User B: hash("hunter2" + salt_B="k9m1..") = "xyz..."   ← same password, DIFFERENT hash!
```
- Salt **unique per user** (random generate)
- Salt **hash ke saath store** hoti hai (secret nahi — uska kaam uniqueness hai, secrecy nahi)
- Rainbow tables useless (har password ke liye alag table chahiye)

**Good news:** bcrypt/argon2 **salt automatically handle** karte hai — salt generate karke hash ke saath output me embed kar dete hai. Tumhe manually kuch nahi karna, bas library use karo.
```
bcrypt output: $2b$12$[22-char salt][31-char hash]
               │    │   └── salt embedded  └── hash
               │    └── cost factor (12)
               └── algorithm (bcrypt)
```

---

## 5. Pepper (extra layer, optional)

**Pepper** = ek **secret value** (salt ke alag) jo saare passwords me add hoti hai, but **DB me nahi** — app config / secrets manager me (lesson 09).
```
hash(password + salt + pepper)
  salt → DB me (unique per user)
  pepper → secrets manager me (same for all, SECRET, not in DB)
```
Agar sirf DB leak ho (but secrets manager safe), pepper ke bina hashes crack nahi honge. Extra defense-in-depth layer (lesson 01). Optional but nice for high-security.

---

## 6. Complete password flow

```
SIGNUP:
  1. User password de: "hunter2"
  2. Salt auto-generate (bcrypt/argon2 karta hai)
  3. hash = bcrypt("hunter2", salt, cost=12)   [+ pepper optional]
  4. DB me store: username + hash (salt embedded) — NEVER the password

LOGIN:
  1. User password de: "hunter2"
  2. DB se stored hash fetch
  3. bcrypt.verify("hunter2", stored_hash) → true/false
     (bcrypt salt embedded se nikaal ke same hash banata, compare karta)
  4. Match → authenticated (session/token issue — lessons 03-04)
```

**FastAPI (passlib):**
```python
from passlib.context import CryptContext
pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")   # or argon2

hashed = pwd.hash("hunter2")            # signup — salt auto
pwd.verify("hunter2", hashed)           # login — returns True/False
```

---

## 7. Additional password protections

```
1. RATE LIMITING (Topic 3): login attempts cap (brute-force rok) —
   "5 attempts per account per 15 min" → account lockout / captcha
2. PASSWORD POLICY: min length (12+), but DON'T force weird rules
   (NIST ab length > complexity kehta hai; "must have symbol" outdated)
3. BREACH CHECK: HaveIBeenPwned API se check "ye password breach me hai?"
   (Topic 3 lesson 09 — bloom filter jaisa membership check)
4. MFA (Multi-Factor Auth): password + OTP/authenticator app — huge security boost
5. NO password hints, security questions (weak — often guessable)
```

> ⚠️ **Timing attacks:** password comparison **constant-time** hona chahiye (bcrypt/argon2 verify ye handle karta hai). Naive `==` string compare timing leak kar sakta hai (kitne chars match hue). Library use karo, manual compare nahi.

**Interview line:** *"Passwords bcrypt (ya argon2id) se hash karunga — deliberately slow, auto-salted per user. Plaintext/encryption kabhi nahi (reversible = bad, aur original chahiye hi nahi). Login attempts rate-limited (brute-force), MFA offer, breached-password check on signup. Pepper in secrets manager for extra layer. Constant-time verify."*

---

## Quick Self-Check
1. Plaintext aur encryption dono passwords ke liye galat kyu?
2. Hashing kyu sahi hai (one-way, original kabhi nahi chahiye)?
3. MD5/SHA passwords ke liye kyu nahi (speed → brute-force)?
4. bcrypt/argon2 "slow" hona kyu acchi baat hai (work factor)?
5. Salt kya solve karta hai (rainbow tables, same-password)? Secret hai ya nahi?
6. Pepper salt se kaise alag (secret, not in DB)?
7. Password ke saath 3 additional protections (rate limit, MFA, breach check)?

---
**Next:** `03-Session-Based-Authentication.md` — login ke baad user ko "yaad" kaise rakho.