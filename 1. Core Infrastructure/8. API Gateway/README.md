# 08 — API Gateway

> **Is lesson me:** API Gateway kya hai (microservices ka single entry point), uski responsibilities (routing, auth, rate limiting, aggregation), BFF pattern, gateway vs load balancer (aur dono saath kaise), gateway vs service mesh, aur anti-patterns. Ye reverse proxy (lesson 06) + LB (lesson 05) + rate limiting (Topic 3) ko ek smart entry point me combine karta hai.

Prerequisites: `06-Proxies.md`, `05-Load-Balancing.md`, helpful: Topic 3 (Rate Limiting)

---

## 1. API Gateway kya hai

**API Gateway** = ek microservices architecture me saare client requests ke liye **single entry point**. Ye ek reverse proxy ki tarah kaam karta hai jo requests ko sahi backend service pe route karta hai, saath me **cross-cutting concerns** (authentication, rate limiting, response transformation) handle karta hai.

**Analogy:** Ek luxury hotel ka concierge desk. Guests khud building me ghoomte nahi housekeeping/restaurant/spa dhundhne. Wo concierge ko batate hai kya chahiye, aur concierge request route karta hai, check karta hai registered guest hai (auth), ensure karta hai daily spa limit cross nahi hui (rate limiting), aur requests translate karta hai ("khaana chahiye" → specific restaurant reservation). API gateway wahi concierge hai.

---

## 2. Kyu chahiye — gateway ke bina problem

Gateway ke bina, **har client** ko:
- Har microservice ka address pata hona chahiye
- Har service ke saath alag se authentication handle karni padegi
- Ek view banane ke liye multiple round-trips
- Alag protocols (gRPC, REST, GraphQL) deal karne padenge

```
Without Gateway (client complexity):        With Gateway (simple client):
Mobile App ─┬─▶ User Service                 Mobile App ─▶ API Gateway ─┬─▶ User Service
            ├─▶ Order Service                                          ├─▶ Order Service
            ├─▶ Payment Service                                        ├─▶ Payment Service
            └─▶ Notification Service                                   └─▶ Notification Service
```
Gateway saari complexity ko ek jagah absorb karta hai — client sirf gateway se baat karta hai.

---

## 3. Gateway responsibilities

| Responsibility | Kya karta hai |
|---|---|
| **Routing** | `/api/users/123` → User Service, `/api/orders` → Order Service |
| **Authentication** | JWT/OAuth token validate before forwarding |
| **Rate limiting** | Per-user/per-IP request limits (Topic 3 lesson 07-08!) |
| **Request transformation** | REST → gRPC convert, headers add, paths rewrite |
| **Response aggregation** | Multiple services ke responses ek me combine |
| **Caching** | GET responses edge pe cache |
| **Circuit breaking** | Unhealthy service ko forwarding rok do |
| **SSL termination** | TLS gateway pe handle (lesson 03), internally plain HTTP |
| **Observability** | Centralized logging, tracing, metrics |

> **Response aggregation ka example:** mobile app ko ek screen ke liye user-info + orders + recommendations chahiye. Client 3 alag calls karne ke bajaye, gateway ek call leta hai, 3 services se fetch karta hai, combine karke ek response deta hai. **Kam round-trips = mobile pe fast** (lesson 01 ka RTT).

> **Circuit breaker kya hai (quick):** agar ek service baar-baar fail/slow ho rahi ho, circuit breaker us pe requests bhejna **temporarily rok** deta hai (fail-fast), taaki ek slow service poore system ko na le doobe. Kuch time baad "test" request bhej ke check karta hai theek hui ya nahi. (Electrical fuse jaisa.)

---

## 4. Backend for Frontend (BFF) Pattern

Alag clients ki alag needs hoti hai — mobile app ko compact response chahiye, web dashboard ko rich data. **BFF** = har client type ke liye ek dedicated gateway.

```
Mobile App  → Mobile BFF  ─┐
Web App     → Web BFF      ─┼─▶ User / Order / Analytics Services
Partner API → Partner BFF  ─┘
```

**Benefits:** har client ke liye optimized payload (mobile ko kam data), alag auth flows (mobile OAuth+biometric, web session cookies), independent deployment per client team, multiple backend calls ko ek client-optimized response me aggregate.

**Used by:** Netflix (per-device BFF), SoundCloud, Spotify.

**Frontend analogy:** BFF = ek dedicated backend layer jo *specifically* tumhare frontend ki zarurat ke hisaab se data shape karta hai — taaki frontend ko 5 APIs orchestrate na karni pade, ek clean endpoint mile.

---

## 5. Gateway vs Load Balancer (important distinction)

| Aspect | API Gateway | Load Balancer |
|---|---|---|
| Layer | L7 (application) | L4 (TCP) ya L7 (HTTP) |
| Routing logic | Path, header, method-based | Round-robin, least-conn |
| Authentication | **Yes** | No (typically) |
| Rate limiting | **Yes** | Basic only |
| Response transformation | **Yes** | No |
| Protocol translation | Yes (REST ↔ gRPC) | No |
| Use case | External API entry point | Internal traffic distribution |

**Practically dono use hote hai:** gateway external concerns handle karta hai (auth, rate limiting, routing to services). Phir **har service ke andar**, ek load balancer instances me distribute karta hai:
```
Client → API Gateway → LB (User Service) → User Instance 1, 2
                     → LB (Order Service) → Order Instance 1, 2
```
**Gateway = "kaunsi service", LB = "us service ka kaunsa instance".**

---

## 6. Gateway vs Service Mesh (lesson 06 connect)

- **API Gateway** = **north-south** traffic (external clients → tumhari services): auth, rate limiting, public API concerns.
- **Service Mesh** (sidecars — lesson 06) = **east-west** traffic (service → service internally): mTLS, retries, circuit breaking between your own services.

**Complementary hai, competing nahi.** External request pehle gateway hit karta hai, phir andar service mesh internal routing handle karta hai. (Kuch teams Envoy dono me use karte hai — gateway mode + sidecar mode.)

---

## 7. Anti-patterns (kya na karo)

| Anti-pattern | Problem | Solution |
|---|---|---|
| **God Gateway** | Saara business logic gateway me | Gateway thin rakho; logic services me |
| Single gateway for all | Ek team ka deploy sabko block kare | BFF ya team-owned gateways |
| No timeout | Slow backend gateway ko hang kar de | Aggressive timeouts + circuit breaker |
| No rate limiting | Ek abusive client sabko DoS kare | Per-tenant rate limits (Topic 3) |
| Tight coupling | Gateway internal schemas jaanta ho | Service contracts, versioned APIs |

> ⚠️ **Sabse bada:** **God Gateway** — gateway ko thin rakho. Wo routing/auth/rate-limit kare, **business logic services me** rahe. Warna gateway ek monolith ban jaata hai jo sabko block karta hai.

---

## 8. Gateway ko single-point-of-failure na banao

(Lesson 05 ka LB principle yahan bhi.) Multiple gateway instances ek LB ke peeche. Gateway ko **stateless** rakho (rate-limit counters Redis me — Topic 3 lesson 08, session state Redis me — lesson 04). Health checks se unhealthy instances hataao. Managed (AWS API Gateway) khud HA deta hai.

**Tools:** Kong (open-source, plugins), AWS API Gateway (managed, Lambda), Nginx (self-managed, high-perf), Envoy (service mesh, gRPC), Traefik (Kubernetes auto-discovery), Apigee (monetization).

**API versioning at gateway:** path-based (`/v1/users`, `/v2/users`) most common — clear, cacheable, aur gateway alag versions ko alag deployments pe route kar sakta hai (gradual rollout).

---

## Quick Self-Check
1. API Gateway kya hai — concierge analogy se? Gateway ke bina client ko kya-kya jhelna padta?
2. Gateway ki 4 core responsibilities?
3. Response aggregation kya hai, aur mobile pe kyu helpful (RTT)?
4. Circuit breaker kya karta hai (fuse analogy)?
5. BFF pattern kya solve karta hai — mobile vs web?
6. **Gateway vs Load Balancer** — "kaunsi service" vs "kaunsa instance" wala split?
7. Gateway (north-south) vs service mesh (east-west) — difference?
8. "God Gateway" anti-pattern kya hai aur fix?

---
**Next:** `09-Connection-Pooling.md` — app se database tak ka efficient last mile.