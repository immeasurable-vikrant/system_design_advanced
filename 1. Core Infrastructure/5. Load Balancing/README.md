# 05 — Load Balancing

> **Is lesson me:** Load balancer kya/kyu, request path me LB kahan baithta hai, LB algorithms (round robin, least connections, IP hash, consistent hashing, + power of two choices), **L4 vs L7 load balancing** (lesson 01 ka L4/L7 yahan apply hota hai), client IP ka kya hota hai (`X-Forwarded-For`), health checks + connection draining, sticky sessions, aur LB ko khud single-point-of-failure banne se kaise rokein. Horizontal scaling (lesson 04) ke baad ye zaroori hai.

Prerequisites: `04-Scalability.md`, `01-Networking-I` (L4/L7)

---

## 1. Load balancer kya hai

**Load balancer (LB)** = ek component jo incoming requests ko **multiple servers (instances) me distribute** karta hai, taaki koi ek server overwhelmed na ho.

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
- **Scalability** — traffic badhe toh peeche aur servers add karo (lesson 04 ka auto-scaling LB me hi naye instances register karta hai)
- **Performance** — load evenly spread, koi bottleneck nahi
- **Maintenance** — ek server offline karke update karo, users affect nahi (rolling deploys)
- **Single entry point** — client ko sirf LB ka ek stable address pata hota hai; peeche servers aate-jaate rahe, client ko farak nahi

### Request path me LB kahan baithta hai?
```
Client → DNS (lesson 03) → [CDN/edge] → LB → instance 1 / 2 / 3 → Redis / DB
                                         ↑
                    yahi DNS ka A/ALIAS record point karta hai
```
Client asal me **LB ke IP** se connect karta hai, kisi server se directly nahi. Isliye `api.example.com` ka DNS record LB pe point karta hai (AWS me ALB ke hostname pe CNAME/ALIAS — lesson 03).

LB **kai jagah** ho sakta hai, sirf ek nahi:
- **Global** — regions ke beech (India user → Mumbai, US user → Virginia): DNS/GeoDNS ya anycast (lesson 03). Isko **GSLB** (Global Server Load Balancing) kehte hai.
- **Edge / external** — internet se aane wala traffic, region ke andar instances me baantna (AWS ALB/NLB).
- **Internal** — ek service se doosri service ke beech (internal ALB, Kubernetes Service, service mesh — lesson 06).
- **Client-side** — client khud servers ki list rakh ke chunta hai (gRPC clients, Redis cluster clients).

---

## 2. LB Algorithms

Kaunsa server next request paaye — ye decide karne ke tarike:

### Round Robin — simplest
Servers me baari-baari: 1 → 2 → 3 → 1 → 2 → 3...
**Pros:** dead simple, even distribution. **Cons:** server ki capacity/current load nahi dekhta — agar Server 2 pe teen slow requests atki hai, phir bhi usko apni baari milegi. **Kab:** saare servers identical + requests similar time lete ho.

### Weighted Round Robin
Same, but strong servers ko zyada traffic (weight ke hisaab se).
```
Server A (weight=5): har 8 me se 5 requests
Server B (weight=2): har 8 me se 2
Server C (weight=1): har 8 me se 1
```
**Kab:** mixed hardware (kuch servers zyada powerful). **Bonus use:** canary deploy — naya version weight=1, purana weight=99 → sirf 1% traffic naye pe.

### Least Connections — adaptive ⭐
Us server ko bhejo jiske **abhi sabse kam active connections** hai.
```
Server A: 45 active   Server B: 12 active ← next yaha   Server C: 67 active
```
**Pros:** real-time load pe adapt karta hai (slow requests wale server ke connections khule rehte hai → naturally usko kam milta hai). **Cons:** thoda overhead (connection count track karna). **Kab:** requests ka processing time vary karta ho (kuch fast, kuch slow) — **most common good default.**

> **Variants:** *Least response time* (sabse kam connections **aur** sabse fast response wala). *Power of two choices* — random 2 servers uthao, unme se jiska load kam ho use bhejo. Sunne me ajeeb, but almost least-connections jitna accha aur bahut sasta (poori list scan nahi karni); Envoy/Nginx me available.

### IP Hash
Client ke IP ko hash karke server decide — **same client hamesha same server.**
```
hash("203.0.113.7") % 3 = 0 → Server A (hamesha)
```
**Pros:** session persistence bina cookies ke, cache locality (same user ka data same server pe). **Cons:** uneven — ek office ya mobile carrier ke hazaaron users **NAT** ki wajah se ek hi public IP pe dikhte hai (lesson 01) → sab ek server pe. Aur server add/remove pe `% N` badalta hai → saari mappings badal jaati hai.

### Consistent Hashing
IP Hash ka advanced version jo server add/remove pe **minimal disruption** deta hai — sirf ~K/N keys remap hoti hai.
> **Recap (Topic 2 lesson 11):** consistent hashing ring pe servers + keys ko map karta hai; server add/remove pe sirf ~1/N keys move hoti hai (naive hash % N me ~sab move hoti hai). **Kab:** distributed caches (Redis cluster — Topic 3), DB sharding, session affinity.

### Comparison
| Algorithm | Even distribution | Session affinity | Adapts to load | Complexity |
|---|---|---|---|---|
| Round Robin | Yes | No | No | Very Low |
| Weighted RR | Configurable | No | No | Low |
| **Least Connections** | Adaptive | No | **Yes** | Medium |
| IP Hash | Depends (NAT pe kharab) | **Yes** | No | Low |
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
| Connections | Usually client ka connection hi aage forward | **Do alag** TCP connections: client↔LB aur LB↔server |
| Balances per | Connection | **Request** |
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

**Interview me: L7 almost hamesha chahiye** — microservices ke liye path-based routing zaroori hai. L4 tab jab raw throughput, non-HTTP protocols (databases, game servers on UDP), ya client IP preserve karna ho bina `X-Forwarded-For` games ke.

### Client ka IP kahan gaya? — `X-Forwarded-For`
L7 LB client ka connection **khatam** karke server se **naya** connection kholta hai. Toh server ko lagta hai request LB ke IP se aayi:
```
Client 203.0.113.7 → ALB 10.0.1.5 → Server
Server ko dikhta: source IP = 10.0.1.5 ❌ (asli user nahi)
```
Fix: LB ek header jodta hai — `X-Forwarded-For: 203.0.113.7` (saath me `X-Forwarded-Proto: https`). Rate limiting, logging, geo — sab isi header se. **Gotcha:** ye header client khud bhi fake bhej sakta hai, isliye sirf **apne LB** ki daali hui value pe bharosa karo (FastAPI/uvicorn me `--forwarded-allow-ips`, Express me `trust proxy`).

### HTTP/2 / gRPC wala catch
gRPC ek **long-lived** HTTP/2 connection pe saari requests multiplex karta hai (lesson 02). **L4 LB per-connection** balance karta hai → client ka ek connection = saari requests ek hi server pe → baaki server khaali. Isliye gRPC ke liye **L7 (request-level) LB** chahiye (Envoy) ya client-side LB. Same issue WebSockets me — connection ek baar jud gaya toh usi server pe rehta hai.

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

**Thresholds:** ek miss pe hi dead mat maano (network blip ho sakta hai). Typical: **2-3 fail → unhealthy**, aur wapas lene ke liye **2-5 pass → healthy**. Naya instance (auto-scaling) bhi tabhi traffic paata hai jab health check pass kare.

> ⚠️ **Health check ke 2 rules (interview gold):**
> 1. Health endpoint **cheap** rakho (bhaari kaam mat karo).
> 2. Downstream dependencies ko **check mat karo** — warna ek DB blip poore fleet ko unhealthy mark kar dega, aur LB ke paas traffic bhejne ko kuch nahi bachega.
>
> **L4 health check** = "port khula hai?" (ek hung app jiska socket khula hai, pass ho jaayega — bad). **L7 health check** = "`/health` 200 deta hai?" (much better).

> **Kubernetes naming:** **liveness** = "process zinda hai? nahi toh restart karo". **Readiness** = "traffic lene ke liye ready hai? nahi toh LB se hatao (restart mat karo)". Startup ke time (model load, cache warm) readiness fail, liveness pass.

### Connection draining (deregistration delay)
Instance ko hatana ho (deploy, ya auto-scaling scale-in — lesson 04), toh LB use **turant kaat'ta nahi**:
1. Us pe **naye** requests bhejna band.
2. Jo requests already chal rahi hai unhe khatam hone ka time do (e.g. 30-300s; AWS ALB default 300s).
3. Phir instance band.
Bina draining ke, har deploy pe kuch users ko beech me hi 502 milta hai.

---

## 5. Sticky Sessions (Session Affinity)

**Problem (lesson 04 ka stateful):** User Server A pe login (session A ki memory me). Agli request Server B → session nahi → logged out.

**Sticky session kaise kaam karta hai:** pehli response me LB ek cookie set karta hai (AWS ALB me `AWSALB`) jisme likha hota hai "ye user Server A ka". Agli requests me browser woh cookie bhejta hai → LB seedha Server A.

**Solutions (preference order):**
| Solution | Kaise | Pros | Cons |
|---|---|---|---|
| **External session store (Redis)** ⭐ | Sessions shared Redis me | Stateless servers, best | Extra infra |
| Sticky sessions (cookie) | LB same user ko same server | Simple | Server death = sessions lost; uneven load; scale-out pe naye server ko purane users nahi milte |
| JWT tokens | Session token me hi (lesson 04) | No server storage | Revoke mushkil, token size |

**Best practice: sessions Redis me rakho.** Phir servers stateless, LB koi bhi algorithm use kare bina session affinity ki chinta ke. (Ye lesson 04 ke stateless principle ka direct application hai.) Stickiness ka legit use: per-server warm cache, ya WebSocket reconnect.

---

## 6. LB khud single point of failure na ban jaye

Ek akela LB khud ek weak point hai — LB mara toh peeche ke 50 healthy servers bhi bekaar. Solutions:
- **Active-Passive** — 2 LBs; primary sab handle kare, mare toh passive le le. Dono ek **floating IP** (virtual IP) share karte hai jo primary ke mare hi standby pe shift ho jaata hai (keepalived/VRRP), ya DNS failover.
- **Active-Active** — dono LBs traffic handle kare (DNS dono IPs de, ya anycast); ek mare toh doosra sab le le. Dono ki capacity aisi rakho ki akela bhi poora load uthaa sake.
- **DNS load balancing** — DNS me multiple LB IPs, client ek pick kare (coarse but simple — lesson 03 ki limits yaad rakho).

**Managed LBs (AWS ALB)** andar se khud kai nodes hote hai, kai availability zones me, aur auto-scale + HA. Isliye ALB tumhe ek **DNS name** deta hai, fixed IP nahi — uske peeche ke IPs badalte rehte hai. Isi wajah se tum usko CNAME/ALIAS karte ho (lesson 03).

**Real tools:** AWS ALB (L7, managed, most web apps), AWS NLB (L4, ultra-low latency, static IPs), Nginx (L7, self-managed reverse proxy + LB), HAProxy (L4/L7, high-perf), Envoy (L7, service mesh, gRPC), Cloudflare (L7, edge + DDoS), Kubernetes Service/Ingress (cluster ke andar).

---

## Quick Self-Check
1. Load balancer kya karta hai + 2 reasons (HA, scalability) — hostess analogy? Request path me kahan baithta hai?
2. Global vs edge vs internal vs client-side LB — ek-ek line?
3. Least Connections kab best hai, aur Round Robin se kaise alag? Power of two choices kya hai?
4. IP Hash aur Consistent Hashing — dono session affinity dete hai, but difference (server add/remove pe)? IP Hash NAT pe kyu kharab?
5. L4 vs L7 load balancing — routing ke terms me? Interview me kaunsa default aur kyu?
6. Server ko asli client IP kaise pata chalta hai? `X-Forwarded-For` pe blind trust kyu nahi?
7. gRPC/WebSocket ke saath L4 LB uneven kyu ho jaata hai?
8. Health check ke 2 rules — downstream dependency check kyu nahi karni? Liveness vs readiness?
9. Connection draining kya hai aur bina iske deploy pe kya hota hai?
10. Sticky sessions ka best solution kya, aur wo lesson 04 se kaise connect hota hai?
11. LB ko single-point-of-failure banne se rokne ke 2 tarike? ALB fixed IP kyu nahi deta?

---

## Flow — client se server tak, LB ke saath
1. **Client → DNS:** `api.example.com` resolve hota hai → LB ka IP milta hai (multi-region ho toh GeoDNS/anycast nearest region chunta hai — global LB).
2. **Client → edge (CDN/WAF, optional) → LB:** TCP + TLS handshake LB pe hota hai (L7 LB TLS terminate karta hai), aur woh `X-Forwarded-For` header jodta hai.
3. **LB → (API gateway, agar hai — auth, rate limit, lesson 06/08) → healthy instance:** L7 LB path/host dekh ke sahi pool chunta hai, phir algorithm (least connections / round robin) se us pool ka ek **healthy** instance — ek naya ya pehle se khula keep-alive connection use karke.
4. **Instance → Redis / DB → response:** stateless instance session Redis/JWT se padhta hai, kaam karta hai, response wapas LB → client.
5. **Background me:** health checks dead instances hatate hai, auto-scaling naye instances register karta hai (health pass hone ke baad), aur scale-in/deploy pe **connection draining** chalu requests poori hone deta hai.

---
**Next:** `06-Proxies.md` — reverse proxy (LB ka bada bhai), forward proxy, sidecar.