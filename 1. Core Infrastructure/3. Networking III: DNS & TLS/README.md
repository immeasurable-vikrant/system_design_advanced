# 03 — Networking III: DNS & TLS

> **Is lesson me:** DNS (naam → IP kaise resolve hota hai, records, TTL, aur global routing — anycast vs GeoDNS), aur TLS (encryption ka handshake, termination vs passthrough vs mTLS, aur famous cert-expiry outage). Ye woh do cheezein hai jo har HTTPS request ke shuru me chupke se hoti hai.

Prerequisites: `01-Networking-I-Layers-And-Transport.md`

---

## 1. DNS — naam se IP tak

**Problem:** Insaan `google.com` yaad rakh sakte hai, `142.250.env.number` nahi. But machines IP addresses se hi baat karti hai (lesson 01). **DNS (Domain Name System)** = internet ki "phone book" jo naam ko IP me convert karti hai.

### Resolution kaise hota hai (the lookup chain)
Jab tum `example.com` type karo, IP dhundhne ke liye ye chain chalti hai:
```
browser cache → OS cache → resolver (ISP ya 8.8.8.8) → root server
   → TLD server (.com) → authoritative nameserver → A record: 93.184.216.34
```
Har step pehle apne cache me dekhta hai (fast), miss pe aage poochta hai. Ye layered caching hi DNS ko scalable banata hai.

**Frontend analogy:** Ye bilkul ek multi-layer cache lookup hai (Topic 3 lesson 02 ke cache layers jaisa) — browser cache → OS cache → resolver cache → phir "origin" (authoritative server). Zyada tar lookups cache se hi resolve ho jaate hai.

### DNS record types (jo matter karte hai)
| Record | Kaam | Gotcha |
|---|---|---|
| **A** | Naam → IPv4 | — |
| **AAAA** | Naam → IPv6 | — |
| **CNAME** | Ek naam ka doosre naam pe alias | Zone apex (`example.com`) pe allowed nahi |
| **ALIAS/ANAME** | Apex-safe CNAME | Provider-specific (Route 53) |
| **MX** | Mail servers | Priority field matters |
| **TXT** | Verification, SPF/DKIM | — |
| **NS** | Nameserver delegation | — |
| **SRV** | Service + port | Service discovery, Kubernetes |

### TTL — the thing interviewers probe
**TTL (Time To Live)** = kitni der resolvers is DNS answer ko cache rakhe. 24-hour TTL matlab: tumne IP change kiya, phir bhi resolvers **poore din tak purana IP** serve karte reh sakte hai (aur kuch short TTL ignore bhi kar dete hai).

> ⚠️ **Migration trap:** *"Cutover ke time TTL kam karna help nahi karta"* — kyunki resolvers ne pehle hi purana record purane long-TTL ke saath cache kar liya. **Sahi tarika:** migration se **ek din pehle** TTL ko 60 seconds tak gira do, phir migrate karo, phir wapas badha do.

### DNS-based load balancing aur uski limit
Multiple A records ya geo-specific answers return karna = sasta global routing. But DNS me client-side **health awareness nahi**, caching fast failover ko rok deti hai, aur clients unpredictably pick karte hai.

**Isliye CDNs anycast use karte hai:**
> **Anycast kya hai:** ek hi IP address ko **kai locations** se announce kiya jaata hai, aur internet ka routing (BGP) har user ko **nearest** location pe bhej deta hai. Failover ek routing change hai (invisible + instant), koi cache expire nahi karni padti. (Regular "unicast" = ek IP, ek location.)

**Interview line:** *"Fast failover ke liye main DNS pe depend nahi karunga — TTLs cached hote hai aur kuch resolvers short TTL ignore karte hai. Anycast with health-aware BGP, ya ek stable-DNS-name wala load balancer jo peeche failover handle kare."*

> **GeoDNS** = DNS user ki location ke hisaab se alag IP deta hai (India user → India server). Simple global routing, but DNS caching lag ke saath.

---

## 2. TLS — encryption (HTTPS ka "S")

**TLS (Transport Layer Security)** = woh protocol jo HTTP ko HTTPS banata hai — data ko **encrypt** karta hai taaki beech me koi padh/badal na sake. Teen cheezein achieve karta hai:
1. **Authentication** — certificate prove karta hai server wahi hai jo claim kar raha (fake server nahi)
2. **Key agreement** — ek shared secret key derive karte hai
3. **Integrity** — koi tamper kare toh pata chal jaye

### Handshake (aur RTT cost — lesson 01 se)
```
TLS 1.2:  ClientHello → ServerHello+cert → key exchange → Finished   (2 RTT)
TLS 1.3:  ClientHello+keyshare → ServerHello+cert+Finished           (1 RTT)
TLS 1.3 resumption / QUIC 0-RTT: pehle packet me hi data             (0 RTT)
```
TLS **asymmetric crypto** (slow) sirf ek shared **symmetric key** agree karne ke liye use karta hai, phir bulk data symmetric (fast) se encrypt hota hai. Yehi poora design hai — slow crypto se key set karo, fast crypto se baaki sab.

**Frontend analogy:** Do log ek secret code decide karne ke liye pehle formally introduce hote hai (slow, asymmetric), phir us code me tezi se baat karte hai (fast, symmetric).

### TLS termination — kahan decrypt hota hai (infra-relevant)
| Term | Matlab |
|---|---|
| **TLS termination** | LB/CDN pe decrypt; backend traffic plain HTTP (fast, but trusted network chahiye) |
| **TLS passthrough** | LB encrypted bytes waise hi aage bhejta hai (end-to-end, but LB L7 routing nahi kar sakta) |
| **Re-encryption** | Terminate, inspect, phir backend ko dobara encrypt (compliance ka usual answer) |
| **mTLS** (mutual TLS) | Dono taraf certificates — service-to-service auth ka standard (service mesh — lesson 06) |

> **Kyu terminate at edge:** TLS handshake user ke *paas* (CDN edge) complete karo → RTT chhota → fast (lesson 01 ka RTT concept). Phir origin tak re-encrypt.

> ⚠️ **The classic outage:** **expired certificate.** Cert expire → poori site down (browsers connection refuse karte hai). **Fix:** renewal automate karo (ACME/Let's Encrypt, AWS ACM) aur **30 din pehle** alert lagao, 3 din nahi.

**SNI (Server Name Indication):** client handshake me hostname batata hai, taaki ek IP kai certs/sites serve kar sake. **Forward secrecy:** ephemeral keys, taaki baad me private key chori ho toh bhi purana traffic decrypt na ho.

---

## 3. Ye sab milke — ek HTTPS request ka shuruaat

```
1. DNS resolve: example.com → 93.184.216.34   (browser/OS/resolver cache, TTL honor)
2. TCP handshake: 1 RTT
3. TLS handshake: 1-2 RTT (cert verify, key agree)
4. HTTP request finally jaata hai
```
Pehle byte se pehle **2-3 RTT** ka cost — isliye connection reuse (lesson 01), edge termination (CDN — lesson 07), aur newer protocols (HTTP/3) itne matter karte hai. Ye poora flow lesson 10 (capstone) me end-to-end assemble karenge.

---

## Quick Self-Check
1. DNS kya solve karta hai, aur resolution chain kya hai (browser se authoritative tak)?
2. DNS lookup kis Topic-3 concept (multi-layer cache) jaisa hai?
3. TTL kya hai, aur migration se pehle TTL kyu (aur kab) girate hai?
4. Anycast kya hai, aur CDN ise fast failover ke liye kyu use karta hai (vs DNS)?
5. TLS 3 cheezein kya achieve karta hai? Asymmetric vs symmetric crypto ka role?
6. TLS termination vs passthrough vs mTLS — teeno kab?
7. Sabse classic TLS outage kya hai aur fix?

---
**Next:** `04-Scalability.md` — ab ek server kaafi kyu nahi, aur scale kaise karte hai.