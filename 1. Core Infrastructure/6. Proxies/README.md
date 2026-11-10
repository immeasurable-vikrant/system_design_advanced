# 06 — Proxies (Forward, Reverse, Transparent, Sidecar) + Gateway + BFF

> **Is lesson me:** Proxy kya hai, forward vs reverse proxy (dono intermediary hai but alag side ke liye), transparent proxy, **SSL/TLS termination** (kya, kyu, kaise), **reverse proxy vs load balancer vs API gateway**, Eureka (proxy nahi — registry), kaunsa kaam kis layer pe kyu, sidecar proxy + mTLS + pod (service mesh), Nginx ka role, **BFF pattern**, aur ek real local dev setup (React + Nginx) ka poora flow — config kahan se padhi jaati hai aur prod me kaun banata hai.

Prerequisites: `05-Load-Balancing.md`

---

## 1. Proxy kya hai

**Proxy** = ek intermediary server jo client aur destination server ke **beech** baithta hai, aur ek side ki taraf se requests forward karta hai. Wo traffic ko modify, filter, cache, ya log kar sakta hai.

**Analogy:** Proxy = ek personal assistant.
- **Forward proxy** = TUMHARA assistant. Tum bolte ho "Company X se info laao", wo tumhari taraf se jaata hai (Company X assistant ko dekhti hai, tumhe nahi).
- **Reverse proxy** = COMPANY X ka receptionist. Tum company ko call karte ho, receptionist uthata hai, sahi department route karta hai, aur tumhe internal extension numbers kabhi pata nahi chalte.

Dono intermediary hai, but **alag sides ke liye** kaam karte hai. Sawaal hamesha ye pucho: **"ye proxy kiski taraf se kaam kar raha hai — client ki ya server ki?"**

---

## 2. Forward Proxy (client-side)

Clients ke **saamne** baithta hai, unki requests internet ko forward karta hai. Destination server proxy ka IP dekhta hai, client ka nahi.

```
Client 1 ─┐
Client 2 ─┼──▶ Forward Proxy ──▶ Website A / B / C
Client 3 ─┘    (client ka IP chhupa)
```

### Kaise kaam karta hai
- **Client ko pata hota hai** ki proxy hai — usse **configure** kiya gaya hai: browser proxy settings, `HTTP_PROXY` / `HTTPS_PROXY` env variables, ya company ki PAC file.
- Client proxy se connect karke bolta hai: *"mere liye `example.com` le aao"*. Proxy apni taraf se request bhejta hai, response wapas deta hai.
- **HTTPS ke saath:** client bolta hai `CONNECT example.com:443`. Proxy bas ek **tunnel** bana deta hai aur encrypted bytes aage-peeche karta hai — use sirf **domain** dikhta hai, andar ka page/data nahi.

### Real-life examples
| Kahan | Kya hota hai |
|---|---|
| **Office network** | Saara bahar ka traffic company proxy se jaata hai: kuch sites block, sab log. Isliye office me kabhi `npm install` / `pip install` tab tak fail hota hai jab tak `HTTP_PROXY` set na karo |
| **Server ka outbound (egress) proxy** | Tumhara backend bank/payment API call karta hai jo sirf **whitelisted IPs** allow karta hai → saara outbound ek fixed-IP proxy (ya NAT gateway) se. Saath me security: servers random internet sites call na kar sake |
| **Scraping** | Rotating proxies — har request alag IP se, taaki block na ho |
| **Anonymity** | Tor, public proxies — site ko tumhara IP nahi dikhta |
| **Caching** | Squid jaise proxies bandwidth bachane ke liye common responses cache karte hai |

> **VPN ≈ forward proxy role**, but technically alag: VPN tumhare **poore device ka network** (L3) tunnel karta hai, forward proxy usually sirf app-level (HTTP) traffic. Kaam same jaisa: websites VPN server ka IP dekhti hai.

---

## 3. Reverse Proxy (server-side) ⭐

Servers ke **saamne** baithta hai, internet se incoming requests handle karta hai. Clients proxy ka address dekhte hai, actual server ka nahi. **Client ko lagta hai yahi server hai** — usko andar ke servers ka pata hi nahi.

```
Internet Clients ──▶ Reverse Proxy ──▶ App Server 1
                     (backend chhupa)  App Server 2
                                       App Server 3
```

**Use cases:** load balancing (lesson 05!), SSL/TLS termination (neeche), caching (static assets/API responses), compression (gzip), security (backend topology chhupao, WAF, DDoS protection), A/B testing, canary deployments (5% traffic naye version ko).

**Examples:** Nginx, HAProxy, Cloudflare, AWS ALB, Apache, Envoy, Traefik.

---

## 4. SSL/TLS termination — kya, kyu, kaise ⭐

> SSL TLS ka purana naam hai. Log aaj bhi "SSL" bolte hai, matlab TLS hi hota hai (lesson 03).

**Termination = woh jagah jahan HTTPS ka encryption "khatam" hota hai** — yaani jahan traffic **decrypt** hota hai. Reverse proxy ke paas certificate + private key hoti hai. Client ke saath TLS handshake **proxy** karta hai, data decrypt karta hai, aur aage app ko plain HTTP bhejta hai (ya dobara encrypt karke — re-encryption).

```
Browser ══HTTPS (encrypted)══▶ Nginx/ALB ──HTTP (plain, private network)──▶ App
                               ↑ cert + private key yahan
                               ↑ TLS yahin "terminate" hua
```

**Kyu proxy pe karte hai:**
1. **Proxy ko andar dekhna padta hai** — path routing, caching, gzip, WAF, headers jodna… ye sab encrypted data pe nahi ho sakta. L7 kaam ke liye decrypt zaroori.
2. **Certificates ek jagah** — 50 instances pe cert install/renew karne ki jagah sirf proxy/LB pe. Expiry outage (lesson 03) ka risk kam.
3. **App ka CPU bachta hai** — handshakes ka crypto proxy kare (Nginx isme bahut efficient hai), app sirf business logic.
4. **Ek jagah TLS policy** — kaunse TLS versions/ciphers allowed, HTTP→HTTPS redirect, HSTS.

**Kaise (Nginx me):**
```nginx
server {
    listen 443 ssl;                                   # yahan HTTPS aata hai
    server_name api.example.com;                      # SNI se sahi cert chuna jaata hai (lesson 03)
    ssl_certificate     /etc/ssl/certs/example.crt;   # certificate (public)
    ssl_certificate_key /etc/ssl/private/example.key; # private key (secret!) — sirf proxy pe

    location / {
        proxy_pass http://app_servers;                # NOTE: aage plain http
        proxy_set_header X-Forwarded-Proto https;     # app ko batao asli request HTTPS thi
        proxy_set_header X-Forwarded-For $remote_addr;# asli client IP (lesson 05)
    }
}
```
`X-Forwarded-Proto` zaroori hai — warna app ko lagega request `http` thi, aur woh galat redirects banayega ya `Secure` cookies set nahi karega.

**Risk:** proxy ke peeche traffic plain hai. Trusted private network (VPC) me theek; zyada security chahiye toh **re-encrypt** ya **mTLS** (section 8).

---

## 5. Reverse Proxy vs Load Balancer vs API Gateway ⭐

Ye interview me bahut poocha jaata hai. Teeno **reverse proxy family** ke hai — farak sirf kitne kaam karte hai:

```
Reverse Proxy (broad concept: server ki taraf se requests receive karke aage bhejna)
   ├── Load Balancer  = RP ka ek KAAM: traffic ko servers me baantna
   ├── API Gateway    = RP + API-specific features (auth, rate limit per client, versioning…)
   ├── Ingress (K8s)  = cluster ka entry RP
   └── CDN            = duniya bhar me faile caching RPs (lesson 07)
```

- **Load balancer** = ek **specific function** (traffic distribute karna).
- **Reverse proxy** = **broader concept** — load balancing **plus** SSL termination, caching, compression, transformation, security.
- **Har reverse proxy load balance kar SAKTA hai, but har load balancer poora reverse proxy NAHI hai.**

### API Gateway kya hai?
**API Gateway = APIs ke liye specialized reverse proxy** — saare clients (web, mobile, partners) ke liye **ek front door** jo microservices ke saamne baithta hai aur API-related cross-cutting kaam karta hai:

| Gateway ka kaam | Example |
|---|---|
| **Routing** service-wise | `/orders/*` → order-service, `/users/*` → user-service |
| **Authentication** | JWT / API key verify, invalid → 401 (service tak pahunchne hi nahi diya) |
| **Rate limiting / quotas** per user, API key ya plan | Free plan 100 req/min, Pro 10K |
| **Request/response transformation** | Header jodna, field rename, version mapping `/v1` → `/v2` |
| **Aggregation** (kabhi-kabhi) | Ek call → 3 services → merged response |
| **Analytics, logging, API keys portal** | Kis partner ne kitni calls ki |

**Kaise banta hai:** usually khud nahi likhte — ready tools: **AWS API Gateway**, **Kong** (Nginx/OpenResty pe bana!), **Apigee**, **Spring Cloud Gateway**, **Envoy-based** (Gloo, Emissary), Tyk. Andar se ye ek reverse proxy hi hai jisme plugins/config se upar wale features jode jaate hai.

> ⚠️ **"Gateway" word ke aur matlab bhi hai** — confuse mat hona: tumhare router ka **default gateway**, AWS **Internet Gateway** / **NAT Gateway** — ye networking devices hai (packets ko ek network se doosre me bhejna), API gateway nahi.

### AWS LB kya hai?
- **AWS ALB = L7 reverse proxy jo load balancing karta hai** — haan, dono. TLS termination, path/host routing, health checks, WAF integration, OIDC login bhi. **Caching/compression nahi** karta (woh CloudFront karta hai).
- **AWS NLB = L4 load balancer** — packets ko tezi se aage bhejta hai, client IP preserve, static IPs. HTTP samajhta hi nahi, toh caching/path routing nahi. (TLS terminate kar sakta hai, but L7 kaam nahi.)
- **AWS API Gateway** = managed API gateway (auth, throttling, API keys, Lambda integration).

### RP aur FP ka aapas me role — ek real request me dono
```
[Employee laptop] → Forward proxy (company ka outbound)
   → Internet
   → CDN / Cloudflare (reverse proxy — edge)
   → ALB (reverse proxy + LB)
   → API Gateway (reverse proxy + auth/rate limit)
   → order-service (+ sidecar proxy)
   → Egress proxy (forward proxy — fixed IP) → Bank / payment API
```
Har proxy **apni side** ko protect karta hai: forward proxies **andar ke clients** ko (outbound), reverse proxies **andar ke servers** ko (inbound). Ek hi request me dono ho sakte hai.

---

## 6. Eureka — kya ye proxy hai? (Nahi)

**Eureka = Netflix ka service registry (service discovery)** — Spring Cloud me bahut use hota hai. Ye **proxy nahi hai**, kyunki **traffic iske through jaata hi nahi.**

- Har service start hote hi Eureka me **register** karti hai: *"main `order-service` hu, `10.0.3.7:8080` pe"*, aur har ~30s heartbeat bhejti hai (band hui → list se hata do).
- Koi service ko order-service call karni ho, woh Eureka se **poochti hai** "order-service kahan hai?" → IPs ki list milti hai → phir **directly** us IP ko call karti hai (client-side load balancing — Spring Cloud LoadBalancer / purana Ribbon).

```
Proxy:    Caller ──request──▶ Proxy ──request──▶ Service      (proxy beech me)
Eureka:   Caller ──"kahan hai?"──▶ Eureka ──"10.0.3.7"──▶ Caller
          Caller ──request──────────────────────────▶ Service  (direct)
```
**Analogy:** Reverse proxy = receptionist (call usi ke through jaati hai). Eureka = **phone directory** (number deta hai, baat tum khud karte ho).

Typical Spring Cloud stack: **Eureka** (registry) + **Spring Cloud Gateway** (gateway — yeh proxy hai) + client-side LB. Kubernetes me ye kaam built-in hai: **DNS + Service** (`order-service` naam se hi IP mil jaata hai). Similar tools: Consul, etcd/ZooKeeper based registries.

---

## 7. Kaunsa kaam kahan karte hai — aur kyu wahin?

**Principle:** har kaam us **sabse pehli layer** pe karo jiske paas uske liye **zaroori jaankari** ho — taaki bekaar traffic andar tak aaye hi na. Aur common kaam (cross-cutting) **ek jagah** karo, taaki 30 services me 30 baar na likhna pade.

| Kaam | Best jagah | Kyu wahan |
|---|---|---|
| DDoS protection, WAF | **CDN / edge** (Cloudflare) | Attack tumhare infra tak pahunchne se pehle hi rok do |
| TLS termination | **Edge / LB / RP** | User ke paas handshake (kam RTT), certs ek jagah |
| Static file caching | **CDN** | User ke paas, origin pe load zero |
| Compression (gzip/brotli) | **RP / CDN** | App ko ye kaam nahi karna padta |
| Per-IP rate limiting | **Edge / RP** | Sirf IP chahiye, sasta, jaldi block |
| Path → service routing | **API gateway / L7 LB / Ingress** | Inko poori HTTP request dikhti hai |
| Auth (token valid hai?) | **API gateway** | Ek baar verify, invalid request services tak nahi pahunchti |
| Per-user / per-plan rate limit, API keys | **API gateway** | Isko pata hai user/plan kaunsa hai |
| UI ke liye data jodna / shape karna | **BFF** (section 10) | Frontend ki zarurat frontend ki team ke paas |
| Service-to-service retries, timeouts, mTLS, tracing | **Sidecar / service mesh** | Har service me same code likhne ki zarurat nahi |
| **Authorization** ("kya ye user ye order edit kar sakta?"), validation, business logic | **Service (app) khud** | Iske liye business data chahiye jo sirf service ke paas hai |

> ⚠️ **Gateway me business logic mat daalo** — "god gateway" ban jaata hai: har team ko gateway change karwana padta hai, woh bottleneck aur single point of failure ban jaata hai.

---

## 8. Transparent Proxy

Client ki **jaankari ya configuration ke bina** traffic intercept karta hai — client ko pata bhi nahi ki proxy hai. Client ne request **asli destination** ko bheji thi; network ne beech me pakad li.

```
Client (sochta hai directly server se baat kar raha)
   → Transparent Proxy (in-path intercept) → Destination Server
```

**Use cases (1-1 line):**
- **ISP caching** — ISP chupke se popular content (purane HTTP din me videos/downloads) apne paas cache kar leta tha, taaki bahar ki bandwidth bache.
- **Corporate content filtering (bina client config)** — office ka router saara traffic ek filter box se guzaarta hai; employee ne kuch set nahi kiya, phir bhi blocked sites nahi khulti.
- **CDN edge caching** — loose example: user ne koi proxy configure nahi kiya, use lagta hai woh "site" se baat kar raha, asal me edge cache jawab de raha (technically CDN reverse proxy hai — upar dekho).
- **Government censorship** — national-level boxes saare traffic ko dekh ke kuch domains block karte hai.

**Limitation — HTTPS:** traffic encrypted hai, toh proxy ko sirf **IP aur domain (SNI)** dikhta hai, page ya data nahi. Isliye woh poora domain block kar sakta hai, ek specific page nahi. Andar dekhna ho toh **MITM** karna padta hai: proxy apna fake certificate dikhaye → browser turant reject karega ("not private"), jab tak device pe proxy ka root CA install na ho. **Company laptops me exactly yahi hota hai** — IT ne unka CA daal rakha hota hai, isliye office proxy HTTPS bhi padh sakta hai.

### Kya reverse proxy bhi transparent proxy hai?
Ek loose sense me — client ko **backend servers** ka pata nahi. But technical term me **nahi**:

| | Client kisko address karta hai? | Client ko proxy ka pata? |
|---|---|---|
| **Forward proxy** | Proxy ko (configure kiya) | Haan, jaan-boojh ke |
| **Transparent proxy** | **Asli server** ko — network beech me pakad leta hai | Nahi |
| **Reverse proxy** | Proxy ko hi — **use hi server samajhta hai** (DNS proxy ka IP deta hai) | Nahi — use lagta hai proxy hi server hai |

Transparent = "chupke se raaste me pakda". Reverse = "client ne khud proxy ka darwaza khatkhataya, bas use pata nahi ki woh receptionist hai".

---

## 9. Sidecar Proxy (Service Mesh), Pod, mTLS

### Pod kya hai?
**Pod = Kubernetes ki sabse chhoti deploy hone wali unit** — ek ya zyada **containers** jo ek saath chalte hai, **same network share** karte hai (same IP, ek doosre se `localhost` pe baat) aur same machine (node) pe rehte hai. Usually ek pod = tumhara app container (+ helper containers jaise sidecar). Kubernetes me lesson 04 ka "instance" ≈ ek pod.

### Sidecar proxy
**Sidecar proxy** = har service ke pod me app ke **bagal me** ek proxy container (usually Envoy), jo us service ka saara inbound/outbound network traffic transparently handle karta hai. Same pod hai, isliye app `localhost` pe sidecar se baat karti hai; sidecar service discovery, TLS, retries, observability sambhalta hai.

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
(Yahan sidecar ek tarah ka **transparent proxy** hai — app ko pata bhi nahi ki uska traffic intercept ho raha.)

**Sidecar (Envoy) features:** L7 load balancing (HTTP/2, gRPC native — lesson 05 ka gRPC catch yahi solve karta hai), automatic retries + circuit breaking, **mTLS** between all services, distributed tracing, rate limiting (Topic 3).

> **Service mesh kya hai:** ek pattern jahan **har** service ke saath ek sidecar proxy chalti hai, aur ye saare sidecars milke ek "mesh" banate hai jo service-to-service communication (retries, security, observability) ko application code ke bahar handle karta hai. **Control plane** (Istio) saare sidecars ko config aur certificates baant'ta hai. Matlab: developers business logic likhe, mesh networking-concerns sambhaale.

### mTLS (mutual TLS) kya hai?
- **Normal TLS:** sirf **server** apna certificate dikhata hai, client verify karta hai ("ye sach me bank.com hai"). Client ki pehchaan baad me password/JWT se hoti hai.
- **mTLS:** **dono** apne certificates dikhate hai. Server bhi client ka cert verify karta hai — *"tum sach me order-service ho?"*

**Kahan use hota hai:** service-to-service (microservices), banks/partner APIs (sirf known clients connect kar sake), IoT devices. Mesh me certificates control plane khud banata aur ghanton me rotate karta rehta hai — developers ko kuch nahi karna.

> **Zero-trust kaise:** "network ke andar ho toh trusted" wala assumption hata do. Har pod ka sidecar mTLS terminate/originate karta hai, saara pod-to-pod traffic sidecars ke through jaata hai, har sidecar ki ek cryptographic identity hoti hai. Network compromise ho jaye toh bhi attacker eavesdrop/impersonate nahi kar sakta.

**Used by:** Istio (Envoy sidecar), Linkerd, Consul Connect.

**Frontend analogy:** Sidecar = har component ke saath ek automatic wrapper (jaise ek HOC ya middleware) jo cross-cutting concerns (auth, logging, retry) handle kare, taaki component khud sirf apna kaam kare.

---

## 10. Nginx — the workhorse

Nginx sabse popular reverse proxy hai. Ek hi tool me: load balancing (round-robin, least-conn, IP hash), SSL/TLS termination + HTTP/2, caching (configurable TTL), compression (gzip/brotli), rate limiting (per-IP/per-URI), health checks, WebSocket upgrade forwarding, static file serving, 100K+ concurrent connections modest hardware pe.

### "Jiska apna HTTP server ho" matlab?
Tumhara app **khud bhi HTTP bol sakta hai** — FastAPI `uvicorn` ke saath port 8000 pe sun sakta hai, Express Node ke `http` server se port 3000 pe, Spring Tomcat se 8080 pe. Browser seedha unse baat kar sakta hai. **Toh beech me Nginx kyu?** Kyunki app ka server tumhara **code chalane** me accha hai, Nginx **networking ke chore** me:

1. **SSL termination** — Nginx TLS handshake + decrypt kare (section 4), app sirf plain HTTP le
2. **Static files** — JS/CSS/images Nginx disk se seedha serve kare, app tak aaye hi nahi
3. **Connection buffering** — slow mobile clients ka data Nginx dheere-dheere le, phir app ko ek jhatke me de (slow-loris attack se app workers bache)
4. **Rate limiting + security** — app-code ke bina
5. **Load balancing + caching + gzip** — kai app instances ke saamne

---

## 11. BFF — Backend For Frontend ⭐

**BFF = ek backend jo kisi ek frontend ke liye specially banaya jaata hai** — web app ka apna BFF, mobile app ka apna. Usually **frontend team** hi isko own karti hai. Browser sirf BFF se baat karta hai; BFF peeche ki services se data laake **UI ke shape me** deta hai.

```
Without BFF:                         With BFF:
Browser ──▶ user-service             Browser ──▶ Web BFF ──▶ user-service
Browser ──▶ orders-service                        (1 call)  ──▶ orders-service
Browser ──▶ notifications-service                           ──▶ notifications-service
(3 round trips slow network pe)      (fast datacenter ke andar 3 calls, browser ko 1 jawab)
```

### Kyu (problems jo solve karta hai)
1. **Kam round trips** — dashboard ke liye 5 services ka data chahiye; browser se 5 calls (mobile/slow network pe har call ka RTT — lesson 01) ki jagah ek call, baaki datacenter ke andar.
2. **UI ke hisaab se data** — services generic data deti hai (bade objects). BFF sirf woh fields deta hai jo screen ko chahiye; mobile BFF aur bhi kam.
3. **Security** — OAuth access/refresh tokens **server pe** (BFF me) rehte hai; browser ko sirf `httpOnly` session cookie milti hai → XSS se token chori nahi. Internal services bahar expose nahi hoti.
4. **Frontend team ki azaadi** — naya screen = BFF me naya endpoint, backend teams ka wait nahi.
5. **SSR** — Next.js/SSR server aksar BFF ka kaam bhi karta hai (server pe data fetch karke HTML).

### Kab use karo / kab nahi
- **Haan:** kai microservices, kai alag clients (web, iOS, Android, partner) jinki zarurat alag, frontend-heavy teams.
- **Nahi:** ek monolith API jo already UI ke shape me data deti hai; chhoti team (ek aur service maintain karni padegi, ek extra network hop).
- **Risk:** business logic BFF me ghus jaata hai, ya 3 BFFs me same logic copy hota hai. Rule: BFF = **jodna aur shape karna**, decide karna (business rules) services ka kaam.

### BFF vs API Gateway
| | API Gateway | BFF |
|---|---|---|
| Kitne | Ek, sab clients ke liye | Har frontend ka ek |
| Kaam | Generic: auth, rate limit, routing | UI-specific: aggregate, shape, session |
| Kaun own karta | Platform/infra team | Frontend team |
| Code | Mostly config/plugins | Asli code (Node/Next.js/GraphQL) |
Dono saath bhi chalte hai: `Browser → Gateway → BFF → services`.

### Kaise implement karte hai (chhota Express example)
```js
// web-bff/routes/dashboard.js
// WHY: Browser ek call kare, BFF datacenter ke andar 3 services ko PARALLEL call kare.
// WHAT-IF-NOT: browser khud 3 calls karta → 3 RTT slow network pe + 3 baar auth + bada payload.
app.get("/bff/dashboard", async (req, res) => {
  // Session cookie se user nikala (token browser me kabhi nahi gaya — XSS safe)
  const token = await sessionStore.getAccessToken(req.cookies.sid);
  if (!token) return res.status(401).end(); // session expire → frontend login pe bheje

  const headers = { Authorization: `Bearer ${token}` };

  // allSettled (not all): ek service fail ho toh poora dashboard na gire.
  // FAILURE CASE: notifications down → baaki dashboard dikhao, bas woh widget khaali.
  const [user, orders, notifs] = await Promise.allSettled([
    fetch(`${USER_SVC}/users/me`, { headers, signal: AbortSignal.timeout(800) }),
    fetch(`${ORDER_SVC}/orders?limit=5`, { headers, signal: AbortSignal.timeout(800) }),
    fetch(`${NOTIF_SVC}/notifications/unread`, { headers, signal: AbortSignal.timeout(500) }),
  ]);
  // Timeout zaroori: bina iske ek slow service poore dashboard ko apni speed pe le doobti (P99 — lesson 04).

  // Sirf UI ke kaam ke fields (services poore objects bhejti hai — browser ko kam bytes)
  res.json({
    name: user.status === "fulfilled" ? (await user.value.json()).firstName : null,
    recentOrders: orders.status === "fulfilled"
      ? (await orders.value.json()).items.map(o => ({ id: o.id, status: o.status, total: o.total }))
      : [],
    unreadCount: notifs.status === "fulfilled" ? (await notifs.value.json()).count : null,
  });
});
```
Common tech: Node/Express, **Next.js API routes / server components**, GraphQL server (ek GraphQL BFF = frontend khud bataye kya fields chahiye).

---

## 12. Real case — React + Nginx local setup (kya ye BFF hai?)

Typical company local setup (tumhare jaisa) aisa dikhta hai. *(Mujhe tumhari actual files nahi dikhi — neeche common pattern hai, apni files se match karna.)*

| Tum kya karte ho | Ye kya hota hai |
|---|---|
| `.env` me QA / pre-prod / prod chunna | React app ke env vars (`REACT_APP_*` / `VITE_*`). Ye **build/dev start ke time JS bundle me chhap jaate hai** — isliye inme secrets kabhi mat rakhna, browser me sab dikhte hai |
| `local.backend.conf` me server pointing | Most likely ek **Nginx include file** — `nginx.conf` ke andar `include local.backend.conf;` — jisme `upstream` / `proxy_pass` likha hai ki `/api` kis backend env (QA/pre-prod) pe jaaye |
| `nginx.conf` me pointing | Main config: kaunse local domain/port pe suno, `/` kahan bhejo, `/api` kahan bhejo |
| `nginx -s stop` → `nginx` | Purana Nginx band karke naya config load karna. (`nginx -t` se pehle config test karo, aur `nginx -s reload` bina band kiye naya config load kar deta hai) |
| `yarn run dev` | React **dev server** (Webpack/Vite) — e.g. `localhost:3000`, hot reload |
| `yarn run server` (doosra terminal) | Ek **Node server** — ya toh SSR server, ya BFF, ya mock/proxy server. Isi pe depend karta hai ki BFF hai ya nahi |

**Local pe Nginx kyu?**
- **Ek origin** — browser ko UI aur API dono ek hi domain/port se milte hai → **CORS ki problem nahi**.
- **Cookies** — company ki auth cookies aksar `.company.com` domain pe set hoti hai; isliye local pe bhi `local.company.com` jaisa domain (`/etc/hosts` me `127.0.0.1` pe mapped) chahiye.
- **Prod jaisi routing** — prod me bhi same paths (`/`, `/api`) ek reverse proxy ke peeche hote hai.

### Kya ye BFF hai?
- **Nginx khud BFF nahi hai** — woh reverse proxy hai (routing, maybe gzip/cache). Woh data jodta ya shape nahi karta.
- **`yarn run server` wala Node server BFF hai agar:** uske andar route handlers hai jo **backend services ko call karke data jodte/badalte** hai, ya session/tokens sambhalte hai. Check karo: kya browser ke Network tab me calls Node server ke paths pe jaati hai, aur Node server code me `fetch/axios` se backend calls hai?
- Agar Node server bas HTML render karta hai → **SSR server**. Agar Nginx `/api` seedha QA backend ko `proxy_pass` karta hai → **koi BFF nahi**, bas reverse proxy.

### Kya SSL termination, caching, gzip, rate limiting `nginx.conf` me hi hote hai?
Ho **sakte** hai — Nginx ye sab karta hai (`listen 443 ssl` + `ssl_certificate`, `proxy_cache`, `gzip on`, `limit_req`). Apni file me ye keywords dhundo. **Local pe** aksar sirf routing hoti hai (kabhi local HTTPS ke liye self-signed cert). **Prod pe** ye kaam aksar aage ki layers karti hai: TLS → ALB/CloudFront, rate limit → CDN/gateway, caching → CDN. Nginx tab bhi gzip/static/routing karta hai.

### Nginx config ko kahan se padhta hai? `/etc`, `/usr` kyu?
- Nginx ek **system-wide program** hai — tumhare project ka hissa nahi, machine pe ek baar install, aur port 80/443 jaise system ports sambhalta hai. Isliye uska config **system ki standard jagah** pe rehta hai, project folder me nahi.
- Install ke time Nginx me ek **default config path** compile hota hai. Start hote hi wahi file padhta hai:
  - Linux: `/etc/nginx/nginx.conf`
  - Mac Homebrew: `/opt/homebrew/etc/nginx/nginx.conf` (Apple Silicon) ya `/usr/local/etc/nginx/nginx.conf` (Intel)
  - Check karo: `nginx -t` (path + config valid hai ya nahi) ya `nginx -V` me `--conf-path`
- `include local.backend.conf;` jaise relative includes isi folder ke relative dhoonde jaate hai.
- **Unix convention (FHS):** `/etc` = system configs, `/usr` (ya `/usr/local`, `/opt/homebrew`) = installed software, `/var/log/nginx` = logs, `/var/run/nginx.pid` = chalte hue Nginx ka process ID.
- `nginx -s stop` kaise kaam karta hai? Woh pid file se master process ka ID padhta hai aur use **signal** bhejta hai. `nginx -c /path/to/project/nginx.conf` se kisi bhi config file ke saath start bhi kar sakte ho.

### Prod pe `nginx.conf` kaun aur kahan banata hai?
- **Kaun:** DevOps / platform / SRE team (kabhi frontend team, apne frontend container ke liye). File **git me version-controlled** hoti hai — infra repo ya app repo me.
- **Kahan/kaise pahunchti hai** (common tarike):
  - **Docker image:** `FROM nginx` → `COPY build/ /usr/share/nginx/html` (React ka `yarn build` output) → `COPY nginx.conf /etc/nginx/conf.d/default.conf`. Har deploy pe naya image.
  - **Kubernetes:** config **ConfigMap** se mount hoti hai, ya **Ingress-NGINX controller** tumhari Ingress YAML se `nginx.conf` **khud generate** karta hai.
  - **VMs:** Ansible/Chef/Terraform templates se file server pe likhi jaati hai.
- **Prod me `yarn dev` nahi chalta** — `yarn build` static files banata hai, jo Nginx ya **S3 + CloudFront** serve karte hai. Env (QA/prod) ki values build ke time ya template se (`envsubst`) daali jaati hai.

---

## 13. Proxy protocols (awareness)

| Protocol | Layer | Use case |
|---|---|---|
| HTTP Proxy | L7 | Web, API routing (Nginx, HAProxy) |
| SOCKS5 | L5 | General TCP, anonymity (Tor, SSH tunnels) |
| TCP Proxy | L4 | Database connections, raw TCP (HAProxy, NLB) |
| UDP Proxy | L4 | DNS, gaming, streaming (Envoy) |

---

## Quick Self-Check
1. Proxy kya hai — forward vs reverse ka assistant analogy? "Kiski taraf se kaam karta hai" wala sawaal?
2. Forward proxy kaise configure hota hai? HTTPS pe usko kya dikhta hai (`CONNECT`)? Egress proxy kyu?
3. SSL/TLS termination kya hai? Proxy pe kyu karte hai (4 reasons)? `X-Forwarded-Proto` kyu?
4. **Reverse proxy vs load balancer vs API gateway** — family tree? AWS ALB vs NLB vs API Gateway?
5. Eureka proxy kyu nahi hai? Directory vs receptionist?
6. Auth, authorization, rate limiting, TLS, retries — har ek kis layer pe aur kyu?
7. Transparent proxy kya hai, aur HTTPS ke saath kya problem? Reverse proxy transparent kyu nahi kehlata?
8. Pod kya hai? Sidecar + service mesh kya hai? mTLS normal TLS se kaise alag?
9. "Apna HTTP server" hone ke baad bhi Nginx saamne kyu?
10. BFF kya hai, kyu, kab nahi? BFF vs API gateway?
11. Local React + Nginx setup — Nginx config kahan se padhta hai, aur prod pe `nginx.conf` kaise pahunchti hai?

---

## Flow 1 — tumhara local setup end-to-end
1. `.env` me env chuna → `yarn run dev` React dev server (e.g. `:3000`) chalu karta hai, env values JS bundle me chhap jaati hai. `yarn run server` Node server (SSR/BFF/mock) doosre port pe.
2. `nginx` start hote hi apne **compiled default path** (`/etc/nginx/` ya `/opt/homebrew/etc/nginx/`) se `nginx.conf` padhta hai, `include local.backend.conf` jodta hai, aur port 80/443 pe sunta hai.
3. Browser `local.company.com` kholta hai → `/etc/hosts` usko `127.0.0.1` deta hai → request **Nginx** pe aati hai.
4. Nginx path dekhta hai: `/` → React dev server, `/api` → Node server ya seedha QA/pre-prod backend (jo `local.backend.conf` me likha hai). Ek origin, isliye CORS nahi aur cookies chalti hai.
5. Agar Node server backend services ko call karke data jodta/shape karta hai → woh tumhara **BFF** hai. Warna Nginx bas reverse proxy hai.

## Flow 2 — prod me ek request (saare proxies ke saath)
1. Browser → DNS → **CDN/WAF** (reverse proxy at edge: DDoS block, static cache, often TLS termination).
2. → **ALB** (L7 reverse proxy + LB: TLS terminate, path routing, health checks) → **API gateway** (auth verify, per-user rate limit) → (optional) **BFF** (UI ke liye data jodna).
3. → service pod: pehle **sidecar** (mTLS, retries, tracing) → app (authorization + business logic) → Redis/DB. Service discovery K8s DNS ya Eureka se.
4. Service ko bahar (bank/payment) call karni ho → **egress forward proxy** (fixed IP, whitelisting).
5. Response wahi raasta wapas: sidecar → gateway → ALB (gzip) → CDN → browser.

---
**Next:** `07-CDN.md` — content ko user ke paas laana (edge delivery).