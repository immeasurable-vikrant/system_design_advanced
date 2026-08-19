# 05 — Load Balancing

> **Is lesson me:** Load balancer kya/kyu, LB algorithms (round robin, least connections, IP hash, consistent hashing), **L4 vs L7 load balancing** (lesson 01 ka L4/L7 yahan apply hota hai), health checks, sticky sessions, aur LB ko khud single-point-of-failure banne se kaise rokein. Horizontal scaling (lesson 04) ke baad ye zaroori hai.

Prerequisites: `04-Scalability.md`, `01-Networking-I` (L4/L7)

---

## 1. Load balancer kya hai

**Load balancer (LB)** = ek component jo incoming requests ko **multiple servers me distribute** karta hai, taaki koi ek server overwhelmed na ho.

**Analogy:** Ek restaurant hostess. Customers (requests) darwaze pe aate hai. Hostess (LB) unhe alag tables (servers) pe baithati hai taaki koi ek waiter overload na ho jabki baaki khaali baithe ho.

```
Without LB:
  Saare 10,000 users → Server 1 → crash 💥

With LB:
  10,000 users → Load Balancer → Server 1 (3,333)
                                → Server 2 (3,333)
                                → Server 3 (3,334)
```

**Kyu chahiye:**
- **High availability** — ek server mare, LB healthy servers pe route kare (zero downtime)
- **Scalability** — traffic badhe toh peeche aur servers add karo (lesson 04)
- **Performance** — load evenly spread, koi bottleneck nahi
- **Maintenance** — ek server offline karke update karo, users affect nahi

---

## 2. LB Algorithms

Kaunsa server next request paaye — ye decide karne ke tarike:

### Round Robin — simplest
Servers me baari-baari: 1 → 2 → 3 → 1 → 2 → 3...
**Pros:** dead simple, even distribution. **Cons:** server ki capacity/current load nahi dekhta. **Kab:** saare servers identical + requests similar time lete ho.

### Weighted Round Robin
Same, but strong servers ko zyada traffic (weight ke hisaab se).
```
Server A (weight=5): har 8 me se 5 requests
Server B (weight=2): har 8 me se 2
Server C (weight=1): har 8 me se 1
```
**Kab:** mixed hardware (kuch servers zyada powerful).

### Least Connections — adaptive ⭐
Us server ko bhejo jiske **abhi sabse kam active connections** hai.
```
Server A: 45 active   Server B: 12 active ← next yaha   Server C: 67 active
```
**Pros:** real-time load pe adapt karta hai (slow requests wale server ko naturally kam milta hai). **Cons:** thoda overhead (connection count track karna). **Kab:** requests ka processing time vary karta ho (kuch fast, kuch slow) — **most common good default.**

### IP Hash
Client ke IP ko hash karke server decide — **same client hamesha same server.**
```
hash("192.168.1.1") % 3 = 0 → Server A (hamesha)
```
**Pros:** session persistence bina cookies ke, cache locality (same user ka data same server pe). **Cons:** uneven agar kuch IPs zyada traffic bheje; server add/remove pe saari mappings badal jaati hai.

### Consistent Hashing
IP Hash ka advanced version jo server add/remove pe **minimal disruption** deta hai — sirf K/N keys remap hoti hai.
> **Recap (Topic 2 lesson 11):** consistent hashing ring pe servers + keys ko map karta hai; server add/remove pe sirf ~1/N keys move hoti hai (naive hash % N me ~sab move hoti hai). **Kab:** distributed caches (Redis cluster — Topic 3), DB sharding, session affinity.

### Comparison
| Algorithm | Even distribution | Session affinity | Adapts to load | Complexity |
|---|---|---|---|---|
| Round Robin | Yes | No | No | Very Low |
| Weighted RR | Configurable | No | No | Low |
| **Least Connections** | Adaptive | No | **Yes** | Medium |
| IP Hash | Depends | **Yes** | No | Low |
| Consistent Hashing | Good | **Yes** | No | Medium |

**Interview default:** *"Least Connections most cases me (varying request times pe adapt karta hai). Round Robin agar sab requests similar duration. IP/Consistent Hashing agar session affinity ya cache locality chahiye."*

---

## 3. L4 vs L7 Load Balancing (lesson 01 apply hota hai)

Yaad hai lesson 01 — L4 sirf IP+port dekhta hai, L7 poori HTTP request. Ye directly LB me:

| | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| Dekhta hai | IP + port + TCP/UDP | Full HTTP: URL, headers, cookies |
| Route on | Source/dest IP, port | Path, host, header, cookie, method |
| Speed | Faster (kam parsing) | Slower (full request inspect) |
| TLS | Passthrough (usually) | Terminates, inspect kar sakta |
| Retries on 5xx | No (dekh nahi sakta) | Yes |
| Example | AWS NLB, HAProxy TCP | AWS ALB, Nginx, Envoy |

**L4 example:** "port 443 ka saara traffic → koi backend" (pata nahi `/api/users` hai ya `/static/img.png`).
**L7 example:**
```
/api/*     → API server pool
/static/*  → static/CDN pool
/ws/*      → WebSocket server pool
Host: admin.example.com → Admin servers
```

**Interview me: L7 almost hamesha chahiye** — microservices ke liye path-based routing zaroori hai. L4 tab jab raw throughput, non-HTTP protocols (databases), ya client IP preserve karna ho bina `X-Forwarded-For` games ke.

---

## 4. Health Checks — LB ko kaise pata server zinda hai

```
LB → GET /health → Server 200 OK   (har 10 seconds)
LB → GET /health → timeout (1st miss)
LB → GET /health → timeout (2nd miss)
LB: "Server dead" → us pe traffic band → alert
```

**Types:**
- **Active** — LB periodically ping kare (most common)
- **Passive** — real traffic monitor kare; N consecutive 5xx → unhealthy mark

> ⚠️ **Health check ke 2 rules (interview gold):**
> 1. Health endpoint **cheap** rakho (bhaari kaam mat karo).
> 2. Downstream dependencies ko **check mat karo** — warna ek DB blip poore fleet ko unhealthy mark kar dega, aur LB ke paas traffic bhejne ko kuch nahi bachega.
>
> **L4 health check** = "port khula hai?" (ek hung app jiska socket khula hai, pass ho jaayega — bad). **L7 health check** = "`/health` 200 deta hai?" (much better).

---

## 5. Sticky Sessions (Session Affinity)

**Problem (lesson 04 ka stateful):** User Server A pe login (session A ki memory me). Agli request Server B → session nahi → logged out.

**Solutions (preference order):**
| Solution | Kaise | Pros | Cons |
|---|---|---|---|
| **External session store (Redis)** ⭐ | Sessions shared Redis me | Stateless servers, best | Extra infra |
| Sticky sessions (cookie) | LB same user ko same server | Simple | Server death = sessions lost |
| JWT tokens | Session token me hi (lesson 04) | No server storage | Revoke mushkil, token size |

**Best practice: sessions Redis me rakho.** Phir servers stateless, LB koi bhi algorithm use kare bina session affinity ki chinta ke. (Ye lesson 04 ke stateless principle ka direct application hai.)

---

## 6. LB khud single point of failure na ban jaye

Ek akela LB khud ek weak point hai. Solutions:
- **Active-Passive** — 2 LBs; primary sab handle kare, mare toh passive le le (floating IP/DNS failover)
- **Active-Active** — dono LBs traffic handle kare (DNS dono IPs de); ek mare toh doosra sab le le
- **DNS load balancing** — DNS me multiple LB IPs, client ek pick kare (coarse but simple)

Managed LBs (AWS ALB) khud auto-scale + HA hote hai.

**Real tools:** AWS ALB (L7, managed, most web apps), AWS NLB (L4, ultra-low latency), Nginx (L7, self-managed reverse proxy + LB), HAProxy (L4/L7, high-perf), Envoy (L7, service mesh), Cloudflare (L7, edge + DDoS).

---

## Quick Self-Check
1. Load balancer kya karta hai + 2 reasons (HA, scalability) — hostess analogy?
2. Least Connections kab best hai, aur Round Robin se kaise alag?
3. IP Hash aur Consistent Hashing — dono session affinity dete hai, but difference (server add/remove pe)?
4. L4 vs L7 load balancing — routing ke terms me? Interview me kaunsa default aur kyu?
5. Health check ke 2 rules — downstream dependency check kyu nahi karni?
6. Sticky sessions ka best solution kya, aur wo lesson 04 se kaise connect hota hai?
7. LB ko single-point-of-failure banne se rokne ke 2 tarike?

---
**Next:** `06-Proxies.md` — reverse proxy (LB ka bada bhai), forward proxy, sidecar.