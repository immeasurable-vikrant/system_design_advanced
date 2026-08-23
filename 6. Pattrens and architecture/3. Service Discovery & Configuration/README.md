# 03 — Service Discovery & Configuration

> **Is lesson me:** Microservices ek dusre ko kaise **dhundhte** hai (jab instances aate-jaate rehte hai, IPs badalte hai) — service registry, client-side vs server-side discovery, health checks — aur configuration management (secrets/config ko code se bahar rakhna). Ye microservices ki "plumbing" hai.

Prerequisites: `01-Monolith-vs-Microservices.md`, Topic 1 lesson 05 (load balancing)

---

## 1. The problem — services move around

Monolith me, ek function doosre ko directly call karta hai (in-process). Microservices me, Order Service ko Payment Service call karni hai — but **Payment Service kaha hai?**

```
Payment Service ke 5 instances chal rahe:
  10.0.1.5:8080, 10.0.1.9:8080, 10.0.2.3:8080, ...
Aur ye badalte rehte hai:
  - Auto-scaling naye instances add kare (Topic 1 lesson 04) → naye IPs
  - Crash/deploy instances hataye → purane IPs dead
  - Cloud har restart pe naya IP de sakta hai
```

Order Service IPs ko **hardcode nahi** kar sakti — wo constantly badalte hai. Chahiye ek dynamic tarika "Payment Service abhi kaha hai" pata karne ka. Ye **service discovery** hai.

**Frontend analogy:** ye DNS jaisa hai (Topic 1 lesson 03) — tum `api.example.com` use karte ho, na ki IP, kyunki IP badal sakta hai. Service discovery internal services ke liye "DNS" hai, but zyada dynamic (health-aware, real-time).

---

## 2. Service Registry — the phone book

**Service registry** = ek central database jo track karta hai kaunse service instances abhi zinda hai aur kaha (IP:port).
```
Service Registry:
  payment-service → [10.0.1.5:8080, 10.0.1.9:8080, 10.0.2.3:8080]
  order-service   → [10.0.3.1:8080, 10.0.3.7:8080]
  user-service    → [10.0.4.2:8080]
```

**Registration (instance registry me kaise aata hai):**
- **Self-registration** — instance startup pe khud registry ko bataye "main zinda hu, yahan hu", aur periodically heartbeat bheje
- **Third-party registration** — ek separate agent (ya orchestrator jaise Kubernetes) instances detect karke register kare

**Health checks (registry ko kaise pata instance mar gaya):**
> **Recap (Topic 1 lesson 05):** registry periodically instance ko ping kare (`/health`). Miss → unhealthy mark → registry se hata → traffic nahi jayega. Yehi mechanism.

**Tools:** Consul, etcd (Topic 5 lesson 04 — Raft!), ZooKeeper, Eureka, Kubernetes built-in (CoreDNS + Services).

---

## 3. Client-side vs Server-side discovery

Kaun registry se poochta hai — client ya ek intermediary?

### Client-side discovery
Client (calling service) khud registry se poochta hai aur ek instance choose karta hai.
```
Order Service → Registry: "payment-service ke instances?"
Registry → [10.0.1.5, 10.0.1.9, 10.0.2.3]
Order Service → khud load balance kare (round-robin) → 10.0.1.9 ko call
```
**Pros:** ek network hop kam (no intermediary), client smart load balancing kar sakta.
**Cons:** har client me discovery logic (har language me), tight coupling to registry.
**Example:** Netflix Eureka + Ribbon.

### Server-side discovery
Client ek **load balancer** ko call karta hai, jo registry se pooch ke route karta hai.
```
Order Service → Load Balancer (fixed address) → registry se pooch ke → 10.0.1.9 ko route
```
**Pros:** client simple (bas ek address janta), discovery logic centralized.
**Cons:** ek extra hop, LB ek dependency.
**Example:** AWS ALB, Kubernetes Services (kube-proxy).

> **Recap (Topic 1 lesson 05-06):** ye load balancer / reverse proxy hi hai, but registry-aware. Kubernetes me ye built-in hai — tum service ko naam se call karte ho (`payment-service`), aur K8s discovery + LB dono handle karta hai.

**Interview:** *"Kubernetes environment me built-in service discovery (DNS-based, server-side) use karunga — services ek dusre ko naam se call karein, K8s IPs + health resolve kare. Non-K8s me Consul ya etcd."*

---

## 4. Service Mesh (discovery + more) — recap

> **Recap (Topic 1 lesson 06):** service mesh (Istio + Envoy sidecars) service-to-service communication ko app-code ke bahar handle karta hai — including **discovery**, load balancing, mTLS, retries, observability. Har service ke bagal me sidecar registry se aware rehta hai. Modern microservices me ye discovery ka common answer hai.

---

## 5. Configuration Management

Alag problem: services ko config chahiye (DB URLs, API keys, feature flags, timeouts) — aur ye **environment se badalti hai** (dev/staging/prod — Topic 2 lesson 05).

**Anti-pattern:** config code me hardcode karna. Deploy karne pe har environment ke liye code badalna padega.

**Solution — externalized config:**
```
Config code se BAHAR:
  - Environment variables (12-factor app — Topic 2 lesson 05 me dekha)
  - Config service (Spring Cloud Config, Consul KV, AWS Parameter Store)
  - Secrets manager (Vault, AWS Secrets Manager) — passwords/keys ke liye ALAG
```

**Key principles:**
- **Config ≠ code** — same code artifact, alag config per environment
- **Secrets alag** — passwords/API keys ko config se alag, encrypted secrets manager me (kabhi git me nahi!)
- **Dynamic config** — kuch config (feature flags, rate limits) runtime pe badle bina redeploy (feature flags service)

> ⚠️ **Secrets in git = disaster.** API keys / passwords kabhi code repo me commit mat karo. Secrets manager use karo. Ye #1 security mistake hai.

**Frontend analogy:** tumne `.env` files use ki hai (`REACT_APP_API_URL`) — same principle. Config bahar, environment-specific, secrets alag. Backend me ye scale pe ek config/secrets service ban jaata hai.

---

## Quick Self-Check
1. Service discovery kis problem ko solve karti hai (instances move around)?
2. Ye DNS se kaise similar/different hai (dynamic, health-aware)?
3. Service registry kya store karta hai? Instances register kaise hote hai (self vs third-party)?
4. Client-side vs server-side discovery — kaun registry se poochta hai, trade-offs?
5. Kubernetes service discovery kaise deta hai (naam se call)?
6. Service mesh discovery me kya role (Topic 1 recap)?
7. Config externalize kyu karte hai? Secrets alag kyu (aur git me kyu nahi)?

---
**Next:** `04-Resilience-Patterns.md` — ek service fail ho toh system kaise bacha rahe.