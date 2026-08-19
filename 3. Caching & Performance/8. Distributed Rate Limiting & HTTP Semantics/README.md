# 08 — Distributed Rate Limiting & HTTP Semantics

> **Is lesson me:** Jab tumhare paas 10 servers ho (har ek alag requests dekh raha), **global** limit kaise enforce karo — centralized Redis (atomic Lua se), local+sync approach, HTTP 429 response semantics, kis cheez pe limit lagao, aur Redis down pe kya. Ye lesson 07 ko real distributed systems me le jaata hai.

Prerequisites: `07-Rate-Limiting-Algorithms.md`

---

## 1. Problem: single-server easy, multi-server hard

Ek server pe rate limiting trivial hai — ek in-memory counter (lesson 07 ka `TokenBucket`). But real systems me **kayi servers** hote hai (load-balanced), aur har server alag requests dekh raha hai:

```
User 123 ki requests load balancer se bat gayi:
  Server A: 40 requests dekhe
  Server B: 35 requests dekhe
  Server C: 30 requests dekhe
Total = 105 (limit 100 se zyada!) but koi single server ko pata nahi.
```

Har server ko lagta hai user under-limit hai. **Global count kisi ek ke paas nahi.** Ise fix karne ke 2 approaches.

---

## 2. Approach 1 — Centralized Store (Redis) ⭐

Saare servers ek hi **shared Redis** se check karte hai before allowing a request. Ab global count ek jagah.

```
Server A: redis.INCR("user:123:minute:500") → 51 → under 100 → allow
Server B: redis.INCR("user:123:minute:500") → 52 → under 100 → allow
...
Server C: redis.INCR("user:123:minute:500") → 101 → over → REJECT
```
Sab same key increment karte hai → true global count.

### Problem: race condition
`INCR` aur "limit check" do alag steps hai. Do servers ek saath: dono `INCR` kare (dono ko 100 mile), dono check kare (dono "under 100"), dono allow → limit cross. Ye classic race (yaad hai DB lesson 09 — lost update / check-then-act).

### Fix: Redis Lua script (atomic)
Lua script Redis pe **atomically** chalta hai (Redis single-threaded — lesson 06 — toh script ke beech koi aur command nahi ghusta). Increment + check ek indivisible operation:
```lua
-- Atomic rate limit check
local count = redis.call('INCR', KEYS[1])
if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])   -- pehli request pe TTL set
end
if count > tonumber(ARGV[2]) then
    return 0    -- rejected
end
return 1        -- allowed
```
Ab increment aur check ek saath — race impossible. **Ye distributed rate limiting ka standard tarika hai.**

> **DB lesson 09 connection:** ye bilkul wahi "atomic operation to prevent race" pattern hai jo humne double-spend rokne me dekha tha (`UPDATE ... WHERE balance >= x`). Yahan Redis Lua wahi atomicity deta hai counter ke liye.

**Trade-off:** har request pe ek Redis round-trip (network hop, ~1-5ms — lesson 02). Usually acceptable, but high-QPS pe ye Redis khud bottleneck ban sakta hai (tab sharding/local-sync).

---

## 3. Approach 2 — Local + Sync

Har server ek **local counter** rakhta hai, aur periodically central store se sync karta hai. Har request pe Redis nahi jaata (faster), but kam accurate.

```
Server A: local_count=30, har 5s me sync
Server B: local_count=25, har 5s me sync
Central: total=55 (under 100) → sab theek
```

**Trade-off:** limit **overshoot** ho sakta hai — up to `(num_servers × requests-in-sync-interval)`. E.g., 10 servers, 5s sync → thoda over-allow. Most use cases me acceptable, aur Redis pe har-request load nahi.

**Kab kaunsa:**
```
Strict global limit chahiye (billing, security)  → Centralized Redis + Lua
High QPS, thoda overshoot ok, Redis load bachao   → Local + sync
```

---

## 4. HTTP semantics — rate-limited request ka response

Jab request limit cross kare, **HTTP 429 Too Many Requests** return karo, saath me helpful headers:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625097600

{ "error": "Rate limit exceeded. Try again in 30 seconds." }
```

| Header | Matlab |
|---|---|
| `Retry-After` | Kitne seconds baad retry karo |
| `X-RateLimit-Limit` | Window me total allowed |
| `X-RateLimit-Remaining` | Kitne bache |
| `X-RateLimit-Reset` | Window kab reset hoga (Unix timestamp) |

**Kyu ye headers matter:** achhe clients inhe padhke **khud slow down** karte hai (retry se pehle wait). Bina inke, client blindly retry karta rehta hai → aur load. **Frontend analogy:** tumne API integration me `429` + `Retry-After` handle kiya hoga (exponential backoff) — server-side ye hi provide karna hai.

---

## 5. Kis cheez pe limit lagao (the "by what" question)

| Limit by | Kab | Example |
|---|---|---|
| **User ID** | Authenticated APIs | "User 1000 API calls/hour" |
| **IP address** | Public endpoints, login | "10 login attempts/IP/min" (brute-force rok) |
| **API key** | Third-party integrations | "Free: 100/day, Pro: 10,000/day" |
| **Endpoint** | Expensive operations | "/search: 30/min, /upload: 5/min" |
| **Combination** | Fine-grained | "User X on /upload: 50/min" |

Real systems **multiple** combine karte hai — per-user AND per-IP AND per-endpoint, alag limits.

---

## 6. Redis down ho gaya? (fail open vs closed)

**Common interview Q:** "Rate-limiter ka Redis down ho jaye toh?"

- **Fail open** — sab requests allow karo (rate limiting temporarily off). System available rehta hai, but overload ka risk. **Usually preferred** — kuch extra requests serve karna, sab users ko reject karne se better.
- **Fail closed** — sab reject. Backend bacha rehta hai, but users blocked.

**Standard answer:** *"Fail open — better to serve some extra requests than reject everyone. Failure ko log + alert karo, aur ek local fallback limiter (per-server coarse limit) rakho taaki total anarchy na ho."*

(Yaad hai lesson 05 — cache down pe bhi fail-open + graceful degradation. Same philosophy.)

---

## 7. Rate limiting vs Throttling (quick distinction)

- **Rate limiting** = hard reject after limit (429).
- **Throttling** = slow down / queue / delay instead of reject.

Rate limiting simpler + more common. Throttling tab jab requests drop nahi kar sakte but pace control karna hai.

---

## Real-world limits (context)
GitHub API: 5000/hour (authenticated, token-based). Stripe: 100/sec per API key. Google Maps: 50/sec per project. OpenAI: dual limit (tokens/min + requests/min). Twitter: 300 tweets/3hr per user.

---

## Quick Self-Check
1. Multi-server pe global limit kyu mushkil hai (har server alag count)?
2. Centralized Redis approach me race condition kaha aata hai, aur Lua script kaise fix karta hai?
3. Redis Lua atomicity kis DB concept (lesson 09) se relate karta hai?
4. Local+sync approach ka trade-off (overshoot)? Kab use karo vs centralized?
5. Rate-limited request ka HTTP status + 3 important headers?
6. `Retry-After` header kyu zaroori (client behavior)?
7. Rate limit "by what" — 3 options aur unke use cases?
8. Redis down pe fail open kyu preferred?

---
**Next:** `09-Bloom-Filters.md` — "ye set me hai?" ko extreme efficiency se answer karna.