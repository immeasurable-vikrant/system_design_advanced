# 06 — Proxies (Forward, Reverse, Sidecar)

> **Is lesson me:** Proxy kya hai, forward vs reverse proxy (dono intermediary hai but alag side ke liye), transparent proxy, sidecar proxy (service mesh), Nginx ka role, aur sabse important — **reverse proxy vs load balancer** ka confusion clear. Load balancer (lesson 05) actually ek type of reverse proxy hai.

Prerequisites: `05-Load-Balancing.md`

---

## 1. Proxy kya hai

**Proxy** = ek intermediary server jo client aur destination server ke **beech** baithta hai, aur ek side ki taraf se requests forward karta hai. Wo traffic ko modify, filter, cache, ya log kar sakta hai.

**Analogy:** Proxy = ek personal assistant.
- **Forward proxy** = TUMHARA assistant. Tum bolte ho "Company X se info laao", wo tumhari taraf se jaata hai (Company X assistant ko dekhti hai, tumhe nahi).
- **Reverse proxy** = COMPANY X ka receptionist. Tum company ko call karte ho, receptionist uthata hai, sahi department route karta hai, aur tumhe internal extension numbers kabhi pata nahi chalte.

Dono intermediary hai, but **alag sides ke liye** kaam karte hai.

---

## 2. Forward Proxy (client-side)

Clients ke **saamne** baithta hai, unki requests internet ko forward karta hai. Destination server proxy ka IP dekhta hai, client ka nahi.

```
Client 1 ─┐
Client 2 ─┼──▶ Forward Proxy ──▶ Website A / B / C
Client 3 ─┘    (client ka IP chhupa)
```

**Use cases:** corporate filtering (kuch sites block), anonymity (client IP chhupao), caching (bandwidth bachao), access control (auth before internet), geo-bypass (VPN), logging (outbound traffic monitor).

**Examples:** Squid proxy, corporate firewalls, VPN services, Tor.

**Frontend analogy:** VPN jo tum use karte ho = forward proxy. Tumhari requests uske through jaati hai, websites uska IP dekhti hai.

---

## 3. Reverse Proxy (server-side) ⭐

Servers ke **saamne** baithta hai, internet se incoming requests handle karta hai. Clients proxy ka address dekhte hai, actual server ka nahi.

```
Internet Clients ──▶ Reverse Proxy ──▶ App Server 1
                     (backend chhupa)  App Server 2
                                       App Server 3
```

**Use cases:** load balancing (lesson 05!), SSL/TLS termination (lesson 03), caching (static assets/API responses), compression (gzip), security (backend topology chhupao, WAF, DDoS protection), A/B testing, canary deployments (5% traffic naye version ko).

**Examples:** Nginx, HAProxy, Cloudflare, AWS ALB, Apache.

---

## 4. Reverse Proxy vs Load Balancer (the classic confusion) ⭐

Ye interview me bahut poocha jaata hai. Clear karo:

- **Load balancer** = ek **specific function** (traffic ko servers me distribute karna).
- **Reverse proxy** = ek **broader concept** jisme load balancing **plus** SSL termination, caching, compression, request/response transformation, A/B testing, security — sab shaamil hai.

```
Har reverse proxy load balance kar SAKTA hai,
but har load balancer poora reverse proxy NAHI hai.

AWS ALB = dono (reverse proxy + LB)
AWS NLB = mainly LB (L4, no caching/transformation)
```

**Ek line me:** load balancing reverse proxy ke kaamon me se **ek** kaam hai. Reverse proxy uska superset hai.

---

## 5. Transparent Proxy

Client ki **jaankari ya configuration ke bina** traffic intercept karta hai — client ko pata bhi nahi ki proxy hai.

```
Client (sochta hai directly server se baat kar raha)
   → Transparent Proxy (in-path intercept) → Destination Server
```

**Use cases:** ISP caching, corporate content filtering (bina client config), CDN edge caching, government censorship.
**Limitation:** HTTPS ke saath achhe se kaam nahi karta (encrypted traffic inspect nahi kar sakta bina MITM certificate ke — lesson 03 TLS).

---

## 6. Sidecar Proxy (Service Mesh)

**Sidecar proxy** = har service instance ke **bagal me** ek proxy chalta hai, jo us service ka saara inbound/outbound network traffic transparently handle karta hai. Application `localhost` se baat karti hai; sidecar service discovery, TLS, retries, observability handle karta hai.

```
Pod A                    Pod B
┌──────────┐            ┌──────────┐
│  App A    │            │  App B    │
│    │      │            │    │      │
│ Envoy ────┼──── mTLS ──┼─ Envoy   │
│ Sidecar   │            │ Sidecar   │
└──────────┘            └──────────┘
      └────── Control Plane (Istio) ──────┘
```

**Sidecar (Envoy) features:** L7 load balancing (HTTP/2, gRPC native), automatic retries + circuit breaking, **mTLS** between all services (zero-trust — lesson 03), distributed tracing, rate limiting (Topic 3).

> **Service mesh kya hai:** ek pattern jahan **har** service ke saath ek sidecar proxy chalti hai, aur ye saare sidecars milke ek "mesh" banate hai jo service-to-service communication (retries, security, observability) ko application code ke bahar handle karta hai. Matlab: developers business logic likhe, mesh networking-concerns sambhaale.

> **Zero-trust kaise:** har pod ka sidecar mTLS terminate/originate karta hai. Saara pod-to-pod traffic sidecars ke through jaata hai (kabhi directly nahi). Har sidecar ki ek cryptographic identity hoti hai. Network compromise ho jaye toh bhi attacker eavesdrop/impersonate nahi kar sakta — har connection mutually authenticated.

**Used by:** Istio (Envoy sidecar), Linkerd, Consul Connect.

**Frontend analogy:** Sidecar = har component ke saath ek automatic wrapper (jaise ek HOC ya middleware) jo cross-cutting concerns (auth, logging, retry) handle kare, taaki component khud sirf apna kaam kare.

---

## 7. Nginx — the workhorse

Nginx sabse popular reverse proxy hai. Ek hi tool me: load balancing (round-robin, least-conn, IP hash), SSL/TLS termination + HTTP/2, caching (configurable TTL), compression (gzip/brotli), rate limiting (per-IP/per-URI), health checks, WebSocket upgrade forwarding, 100K+ concurrent connections modest hardware pe.

**Kyu Nginx ek app ke saamne rakhte ho (jiska apna HTTP server ho):**
1. **SSL termination** — Nginx TLS efficiently handle kare, app CPU bache
2. **Static files** — Nginx disk se static serve kare, app tak na aaye
3. **Connection buffering** — Nginx slow clients absorb kare (slow-loris se app threads bache)
4. **Rate limiting + security** — app-code ke bina
5. **Load balancing** + **caching**

---

## 8. Proxy protocols (awareness)

| Protocol | Layer | Use case |
|---|---|---|
| HTTP Proxy | L7 | Web, API routing (Nginx, HAProxy) |
| SOCKS5 | L5 | General TCP, anonymity (Tor, SSH tunnels) |
| TCP Proxy | L4 | Database connections, raw TCP (HAProxy, NLB) |
| UDP Proxy | L4 | DNS, gaming, streaming (Envoy) |

---

## Quick Self-Check
1. Proxy kya hai — forward vs reverse ka assistant analogy?
2. Forward proxy kiska IP chhupata hai, reverse proxy kiska?
3. **Reverse proxy vs load balancer** — exact relationship (superset/subset)? AWS ALB vs NLB?
4. Transparent proxy kya hai, aur HTTPS ke saath kya problem?
5. Sidecar proxy + service mesh kya hai — developers ko kya fayda?
6. Zero-trust mTLS sidecars se kaise achieve hota hai?
7. Ek app ke saamne Nginx kyu rakhte hai (3 reasons)?

---
**Next:** `07-CDN.md` — content ko user ke paas laana (edge delivery).