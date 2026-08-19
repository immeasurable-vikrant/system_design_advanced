# 07 — Rate Limiting Algorithms

> **Is lesson me:** Rate limiting kya/kyu, aur 5 algorithms — token bucket, leaky bucket, fixed window, sliding window log, sliding window counter — har ek ka mechanism, pros/cons, aur interview me kaunsa bolna. Ye system ko overload/abuse se bachane ka core hai.

Prerequisites: `06-Redis-vs-Memcached.md` (Redis structures rate limiting me use honge)

---

## 1. Rate limiting kya hai

**Rate limiting** = control karna ki ek client ek time-window me **kitni requests** kar sakta hai. Ye system ko overwhelm hone se bachata hai — chahe wo bug ho, abuse ho, ya DDoS attack.

**Analogy:** Ek nightclub with a bouncer. Capacity 200. 200 log andar → bouncer aur logon ko rokta hai jab tak koi nikle. Kaun ho farak nahi padta — limit sab pe.

```
Without rate limiting:
  Buggy client → 10,000 req/sec → servers CPU maxed → DB pool exhausted
  → legitimate users ko errors → system crash 💥

With rate limiting:
  Buggy client → 10,000 req/sec → limiter allows 100/sec, rejects 9,900
  → client ko 429 Too Many Requests → system healthy for everyone else ✅
```

**Kyu zaroori:** ek buggy/malicious client poore system ko gira sakta hai. Rate limiting = "blast radius" contain karna. (Yaad hai lesson 05 — cache down pe DB flood? Rate limiting wahan bhi bachata hai.)

---

## 2. Kaha lagta hai

```
Client → CDN/Edge (Cloudflare) → API Gateway (per-user limits) → Service
                                        ▲                            ▼
                                  best place              Per-endpoint limits
```

**Best practice: edge/API-Gateway level pe** lagao — requests kisi backend service tak pahunche **usse pehle**. Har microservice ke andar mat lagao (too late, aur logic duplicate).

---

## 3. Algorithm 1 — Token Bucket (most popular) ⭐

**Idea:** ek bucket me tokens hote hai. Tokens fixed rate se add hote hai. Har request ek token "kharch" karti hai. Bucket khali → request reject.

```
Bucket capacity: 10 tokens (max burst)
Refill rate: 2 tokens/second

t=0: bucket=10 → 5 requests aayi → 5 used → 5 bache
t=1: +2 refill → 7 → 3 requests → 3 used → 4 bache
t=2: +2 refill → 6 → 8 requests aayi → sirf 6 allowed, 2 REJECTED
```

**Properties:** bursts allow karta hai (bucket capacity tak), average rate = refill rate, simple.

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate   # tokens/sec
        self.last_refill = time.time()

    def allow_request(self):
        now = time.time()
        elapsed = now - self.last_refill
        # elapsed time ke hisaab se refill (capacity se zyada nahi)
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True    # allowed
        return False       # rejected (429)
```

**Used by:** AWS API Gateway, Stripe, most cloud providers.
**Bursts kyu acche:** real traffic bursty hota hai (user ek page khole → 10 API calls ek saath). Token bucket controlled bursts allow karta hai — user-friendly.

---

## 4. Algorithm 2 — Leaky Bucket

**Idea:** requests ek queue (bucket) me aati hai. Queue **fixed rate** pe process hoti hai. Queue full → nayi requests drop.

```
Queue capacity: 10, Processing rate: 5 req/sec
Burst of 15 requests:
  10 queue me enter → 5 immediately dropped
  Queue 5/sec drain hoti hai (2 sec me empty)
```

**Token bucket se difference:**
- **Token bucket:** bursts allow (tokens ho toh turant process) — output bursty ho sakta hai
- **Leaky bucket:** output **smooth** karta hai (hamesha fixed rate pe process, chahe input bursty ho)

**Used by:** network traffic shaping (TCP congestion control). API rate limiting me kam common (bursts allow karna usually chahiye).

---

## 5. Algorithm 3 — Fixed Window Counter (simplest, but flawed)

**Idea:** fixed time-windows me count karo (minute 1, minute 2...).
```
Window: 1 min, Limit: 100
Minute 1 (0:00-0:59): 60 req → allowed
Minute 2 (1:00-1:59): counter reset → 0 se phir
```

**The fatal flaw — boundary burst:**
```
Minute 1, second 59: 100 requests (just under limit) ✅
Minute 2, second 00: 100 requests (naya window, counter reset) ✅
→ 2 seconds me 200 requests! Intended rate ka DOUBLE.
```
Window boundary pe client 2x limit nikaal sakta hai. Isliye production me **avoid** — sliding window better hai.

**Pro:** super simple, 1 counter. **Con:** edge burst.

---

## 6. Algorithm 4 — Sliding Window Log (most precise)

**Idea:** har request ka **timestamp** store karo. Nayi request pe, count karo kitne timestamps last N seconds me aate hai.

```
Window: 60s, Limit: 100
Request at t=150: range [90,150] me timestamps count = 98 → under → allow, store 150
Request at t=151: range [91,151] me count = 100 → AT limit → reject
```

**Redis se (Sorted Set — yaad hai lesson 06/DB lesson 07):**
```
ZREMRANGEBYSCORE key 0 (now - window)   # purane timestamps hatao
ZCARD key                                # current window count
ZADD key now now                         # ye request add karo
EXPIRE key window                        # cleanup
```

**Pros:** perfectly accurate, koi boundary burst nahi.
**Cons:** **memory-heavy** — har request ka timestamp store. 10K req/sec = 600K entries per minute per user. Bahut memory.

---

## 7. Algorithm 5 — Sliding Window Counter (best tradeoff) ⭐

**Idea:** do fixed windows combine karo, previous window ko time-overlap se weight karo. ~99% accurate, sirf **2 counters** (minimal memory).

```
Window: 1 min. Abhi current minute me 15 sec (25%) in.
Previous minute count: 80,  Current minute count: 20

Weighted estimate = 80 × 75% + 20 × 100% = 60 + 20 = 80
  (75% kyunki previous window ka 75% abhi bhi sliding window me overlap karta hai)
Limit 100 → 80 < 100 → allowed
```

**Approximate kyu:** assume karta hai previous window me requests evenly distributed the. Reality me bursty ho sakte, but error typically **< 1%** — acceptable.

**Pros:** sirf 2 integers per key (memory-light), very fast, no bad boundary burst.
**Cons:** ~1% inaccuracy (most cases me theek).

**Used by:** Cloudflare, most production rate limiters.

---

## 8. Comparison + verdict

| Algorithm | Accuracy | Memory | Burst handling | Complexity |
|---|---|---|---|---|
| **Token Bucket** | Good | Low (1 counter + timestamp) | Allows controlled bursts | Low |
| Sliding Window Log | Perfect | High (all timestamps) | No boundary burst | Medium |
| **Sliding Window Counter** | ~99% | Low (2 counters) | Minimal boundary burst | Low |
| Leaky Bucket | Good | Medium (queue) | Smooths all bursts | Medium |
| Fixed Window | Poor (edge burst) | Low (1 counter) | Bad at boundaries | Very Low |

**Interview me: "Token Bucket" ya "Sliding Window Counter" bolo.**
- **Token Bucket** — jab controlled bursts chahiye (most APIs), simple, 2 values per key.
- **Sliding Window Counter** — jab strict-but-cheap accuracy chahiye.
- Agar poocha jaye "fixed window kyu nahi" → **boundary burst problem** explain karo.

---

## Quick Self-Check
1. Rate limiting kya bachata hai — ek buggy client ke scenario se?
2. Rate limiting kaha lagana best (aur har microservice me kyu nahi)?
3. Token bucket kaise kaam karta hai, aur bursts kyu allow karta hai (good)?
4. Token bucket vs leaky bucket — output ke terms me difference?
5. Fixed window ka fatal flaw kya hai (boundary burst example)?
6. Sliding window log accurate hai but kya problem (memory)? Redis me kis structure se?
7. Sliding window counter 2 counters se ~99% accuracy kaise deta hai?
8. Interview me kaunse 2 algorithms bolne chahiye?

---
**Next:** `08-Distributed-Rate-Limiting.md` — jab 10 servers ho, global limit kaise enforce karo.