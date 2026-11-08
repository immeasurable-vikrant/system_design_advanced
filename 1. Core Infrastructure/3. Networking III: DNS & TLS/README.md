# 03 — Networking III: DNS & TLS

> **Is lesson me:** DNS (naam → IP kaise resolve hota hai — har cache aur resolver step-by-step, records kya hote hai, TTL, aur global routing — DNS load balancing vs anycast vs GeoDNS), aur TLS (certificate kya hai aur kaise verify hota hai, handshake, termination vs passthrough vs mTLS, aur famous cert-expiry outage). Ye woh do cheezein hai jo har HTTPS request ke shuru me chupke se hoti hai.

Prerequisites: `01-Networking-I-Layers-And-Transport.md`

---

## 1. DNS — naam se IP tak

**Problem:** Insaan `google.com` yaad rakh sakte hai, `142.250.183.14` nahi. But machines IP addresses se hi baat karti hai (lesson 01). **DNS (Domain Name System)** = internet ki "phone book" jo naam ko IP me convert karti hai.

DNS ek **distributed** system hai — koi ek server saare naam nahi jaanta. Har level sirf apna hissa jaanta hai aur agle ka pata batata hai.

### 1.1 Resolution chain — har step kya hai

```
browser cache → OS cache → recursive resolver (ISP ya 8.8.8.8) → root server
   → TLD server (.com) → authoritative nameserver → A record: 93.184.216.34
```

**Pehle 3 steps = caches** (jinke paas answer pehle se pada ho sakta hai):

| Step | Kya hai | Cache me kya rehta hai |
|---|---|---|
| **Browser cache** | Chrome/Firefox ka apna chhota DNS cache | Haal hi me visit kiye naam → IP, kuch seconds-minutes ke liye (Chrome me dekho: `chrome://net-internals/#dns`) |
| **OS cache** | Operating system ka DNS client (macOS `mDNSResponder`, Windows DNS Client, Linux `systemd-resolved`) | Saari apps (browser, curl, Node) ke lookups. Isse pehle OS **`/etc/hosts`** file bhi dekhta hai — wahan likha naam sabse pehle jeet ta hai |
| **Recursive resolver** | Ek server jo **tumhari jagah poora chase karta hai** — ISP ka (Jio/Airtel), ya public: `8.8.8.8` (Google), `1.1.1.1` (Cloudflare). Tumhara laptop isko DHCP/router se jaanta hai | Lakhon users ke answers. `google.com` jaisa popular naam almost hamesha yahan cached milta hai |

**Resolver kyu hai?** Tumhara laptop khud root → TLD → authoritative tak 3-4 servers se baat kare toh slow hoga, aur har device yahi kaam alag-alag karega. Resolver ye kaam ek baar karta hai aur result **sab users ke liye cache** kar leta hai. "Recursive" isliye kehte hai kyunki ye answer milne tak khud peecha karta hai; tumhe bas final IP milta hai.

**Agle 3 steps = asli source of truth** (sirf tab, jab resolver ke cache me nahi mila):

| Step | Kya jaanta hai | Kya jawab deta hai |
|---|---|---|
| **Root server** | Sirf ye ki har TLD (`.com`, `.in`, `.org`) kaun sambhalta hai. 13 root naam hai (`a`–`m.root-servers.net`), anycast se duniya bhar me 1000+ copies | "`.com` ke liye in servers se poocho" (referral) |
| **TLD server** | `.com` ke saare domains ke **nameservers** (NS records) — `.com` Verisign chalata hai | "`example.com` ke authoritative nameservers ye hai: `ns1.cloudflare.com`…" |
| **Authoritative nameserver** | Domain owner ne jo records khud set kiye — ye **final truth** hai (Route 53, Cloudflare DNS, GoDaddy DNS) | "`example.com` → A `93.184.216.34`, TTL 300" |

Wapsi me resolver answer cache karta hai (TTL tak), OS cache karta hai, browser cache karta hai. **Cache hit = ~1-10ms; poora miss = ~50-200ms.** Zyada tar lookups cache se hi nipat jaate hai — isliye DNS itna scale karta hai.

**Ye chain set kaun karta hai?** Tum domain **registrar** (GoDaddy, Namecheap) se khareedte ho aur wahan batate ho ki tumhare nameservers kaun hai (e.g., Cloudflare). Registrar ye `.com` TLD tak pahuncha deta hai. Phir tum apne DNS provider (Cloudflare/Route 53) pe actual records daalte ho.

**Frontend analogy:** Ye bilkul ek multi-layer cache lookup hai (Topic 3 lesson 02 ke cache layers jaisa) — browser cache → OS cache → resolver cache → phir "origin" (authoritative server). Resolver = ek shared CDN cache jaisa; authoritative = tumhara origin DB.

### 1.2 DNS records — "record" hota kya hai?

**Record = domain ki DNS settings ki ek row:** `naam + type + value + TTL`. Authoritative nameserver pe tumhare domain ki saari rows ek "zone" kehlati hai. Example zone:

```
NAAM                 TYPE   VALUE                                        TTL
example.com.         A      93.184.216.34                                300
example.com.         AAAA   2606:2800:220:1::248                         300
www.example.com.     CNAME  example.com.                                 300
api.example.com.     CNAME  my-alb-123.ap-south-1.elb.amazonaws.com.     300
example.com.         MX     10 aspmx.l.google.com.                       3600
example.com.         TXT    "v=spf1 include:_spf.google.com ~all"        3600
example.com.         NS     ns1.cloudflare.com.                          86400
```

| Record | Kaam (ek line) | Gotcha |
|---|---|---|
| **A** | Naam → **IPv4** address. Final answer jis pe browser connect karta hai | — |
| **AAAA** | Naam → **IPv6** address | — |
| **CNAME** | Naam → **doosra naam** (alias). Resolver phir us doosre naam ko resolve karta hai (ek extra lookup) | Zone apex (`example.com`) pe allowed nahi — kyunki apex pe NS/SOA records hote hi hai, aur CNAME kisi aur record ke saath nahi reh sakta |
| **ALIAS/ANAME** | Apex-safe CNAME — provider khud andar resolve karke A record jaisa jawab deta hai | Provider-specific (Route 53, Cloudflare "CNAME flattening") |
| **MX** | Is domain ki **email** kaun receive karta hai | Priority number — chhota number pehle try hota hai |
| **TXT** | Free text — domain ownership verify karna, email anti-spoofing (SPF/DKIM) | — |
| **NS** | Is domain ke **authoritative nameservers** kaun hai (delegation) | — |
| **SRV** | Service ka host **+ port** | Service discovery, Kubernetes |

**CNAME kab use karte hai (real life):** Jab target ka IP tumhare control me nahi aur badal sakta hai. AWS load balancer ke IPs badalte rehte hai, isliye `api.example.com` ko ALB ke **hostname** pe CNAME karte hai, IP pe A record nahi. Vercel/Netlify bhi tumse yahi karwate hai.

### 1.3 TTL — the thing interviewers probe
**TTL (Time To Live)** = kitni der resolvers is DNS answer ko cache rakhe. 24-hour TTL matlab: tumne IP change kiya, phir bhi resolvers **poore din tak purana IP** serve karte reh sakte hai (aur kuch short TTL ignore bhi kar dete hai).

> ⚠️ **Migration trap:** *"Cutover ke time TTL kam karna help nahi karta"* — kyunki resolvers ne pehle hi purana record purane long-TTL ke saath cache kar liya. **Sahi tarika:** migration se **ek din pehle** (purane TTL jitna pehle) TTL ko 60 seconds tak gira do, phir migrate karo, phir wapas badha do.

### 1.4 DNS-based load balancing aur uski limit

**Idea:** Load balancer machine lagaye bina, **DNS khud traffic baant de.** Authoritative server ek naam ke liye kai IPs rakhta hai aur alag-alag users ko alag IP deta hai (ya list ko ghuma ke deta hai — "round-robin DNS"):
```
api.example.com → user A ko: 10.0.0.1
                → user B ko: 10.0.0.2
                → user C ko: 10.0.0.3
```
Sasta aur simple — koi extra box nahi. **But 4 problems:**

1. **Health ka pata nahi** — server `10.0.0.2` mar gaya, DNS usko deta rahega jab tak tum record hatao. (Route 53 jaise providers health check karke dead IP dena band kar dete hai, but problem 2 phir bhi rehti hai.)
2. **Caching failover rok deti hai** — record hata bhi diya, toh jinke resolver/OS/browser me purana IP cached hai, woh **TTL khatam hone tak** mare hue server pe jaate rahenge.
3. **Uneven load** — ek resolver ek answer lakhon users ke liye cache karta hai. Jio ke ek resolver ke saare users ek hi IP pe gaye → ek server pe pahaad, baaki khaali.
4. **Client ki marzi** — kai IPs milne pe kaunsa use karna hai ye client decide karta hai, aur kai clients hamesha pehla hi pakadte hai.

**Asli load balancer se compare:** LB ka ek **stable IP** hota hai, aur woh har connection ko us waqt ke **healthy** backend pe bhejta hai — failover turant, cache ka koi jhanjhat nahi (lesson 05). DNS LB bas "rough baatna" hai.

**Isliye CDNs anycast use karte hai:**
> **Anycast kya hai:** ek hi IP address ko **kai locations** se announce kiya jaata hai, aur internet ka routing (BGP) har user ko **nearest** location pe bhej deta hai. Failover ek routing change hai (invisible + instant), koi cache expire nahi karni padti kyunki IP badla hi nahi. (Regular "unicast" = ek IP, ek location.)

> **GeoDNS** = DNS user (asal me uske resolver) ki location ke hisaab se alag IP deta hai (India user → India server). Simple global routing, but upar wale caching lag ke saath.

**Interview line:** *"Fast failover ke liye main DNS pe depend nahi karunga — TTLs cached hote hai aur kuch resolvers short TTL ignore karte hai. Anycast with health-aware BGP, ya ek stable-DNS-name wala load balancer jo peeche failover handle kare."*

---

## 2. TLS — encryption (HTTPS ka "S")

**TLS (Transport Layer Security)** = woh protocol jo HTTP ko HTTPS banata hai — data ko **encrypt** karta hai taaki beech me koi padh/badal na sake. Teen cheezein achieve karta hai:
1. **Authentication** — certificate prove karta hai server wahi hai jo claim kar raha (fake server nahi)
2. **Key agreement** — ek shared secret key derive karte hai
3. **Integrity** — koi tamper kare toh pata chal jaye

### 2.1 Certificate wala scene

**Problem:** Encrypt toh kar loge, but **kis ke saath?** Agar beech me koi fake server (attacker) baitha ho aur khud ko `bank.com` bole, toh tum attacker ke saath "securely" baat kar rahe hoge. Certificate isi ko rokta hai.

**Certificate = domain ka digital ID card.** Isme hota hai:
- **Domain name(s)** jiske liye ye valid hai (`example.com`, `*.example.com`)
- Server ki **public key**
- **Validity dates** (kab se kab tak)
- **Issuer** — kisne issue kiya (CA)
- Issuer ka **digital signature** — taaki koi cert me kuch badal na sake

**CA (Certificate Authority)** = trusted issuer, jaise **Let's Encrypt, DigiCert, Google Trust Services.** Har browser/OS ke andar trusted root CAs ki ek list aati hai ("**trust store**"). Browser kisi ko trust nahi karta jab tak chain kisi trusted root tak na pahunche:
```
Root CA (browser/OS me pehle se trusted)
  └─ signs → Intermediate CA
               └─ signs → example.com ka certificate (leaf)
```
Server handshake me leaf + intermediate bhejta hai; root browser ke paas pehle se hota hai.

**Cert milta kaise hai:**
1. Server pe ek **key pair** banao — private key server pe secret rehti hai, public key cert me jaayegi.
2. CA ko prove karo ki domain tumhara hai — ACME/Let's Encrypt me ya toh ek file `http://example.com/.well-known/acme-challenge/...` pe rakho, ya ek **DNS TXT record** daalo (upar wala TXT yahi kaam aata hai).
3. CA cert ko apni private key se **sign** karke de deta hai (Let's Encrypt cert ~90 din valid, isliye auto-renew zaroori).

**Browser verify kya karta hai:**
- Signature chain ek **trusted root** tak jaati hai?
- Cert ka domain = jo URL khola (SNI wala hostname)?
- **Expire** toh nahi hua?
- Revoke toh nahi hua?
Ek bhi fail → "Your connection is not private" page.

**"Cert toh public hai, attacker copy kyu nahi kar leta?"** — Copy kar sakta hai, but handshake me server ko apni **private key** se ek signature banana padta hai jo sirf cert ki public key se verify hota hai. Private key ke bina attacker ye nahi bana sakta → pakda jaata hai. **Cert = ID card, private key = asli chehra.**

*Try karo:* browser me lock icon → "Connection is secure" → certificate dekho — issuer, expiry, chain sab dikhega.

### 2.2 Handshake (aur RTT cost — lesson 01 se)
```
TLS 1.2:  ClientHello → ServerHello+cert → key exchange → Finished   (2 RTT)
TLS 1.3:  ClientHello+keyshare → ServerHello+cert+Finished           (1 RTT)
TLS 1.3 resumption / QUIC 0-RTT: pehle packet me hi data             (0 RTT)
```
TLS **asymmetric crypto** (slow — public/private key wala) sirf **identity prove** karne aur ek shared **symmetric key** agree karne ke liye use karta hai, phir bulk data symmetric (fast, e.g. AES) se encrypt hota hai. Yehi poora design hai — slow crypto se key set karo, fast crypto se baaki sab.

**Frontend analogy:** Do log ek secret code decide karne ke liye pehle formally introduce hote hai aur ID card dikhate hai (slow, asymmetric), phir us code me tezi se baat karte hai (fast, symmetric).

### 2.3 TLS termination — kahan decrypt hota hai (infra-relevant)
| Term | Matlab |
|---|---|
| **TLS termination** | LB/CDN pe decrypt; backend traffic plain HTTP (fast, but trusted network chahiye) |
| **TLS passthrough** | LB encrypted bytes waise hi aage bhejta hai (end-to-end, but LB L7 routing nahi kar sakta — woh HTTP padh hi nahi sakta, sirf L4) |
| **Re-encryption** | Terminate, inspect, phir backend ko dobara encrypt (compliance ka usual answer) |
| **mTLS** (mutual TLS) | Dono taraf certificates — server bhi client ka cert verify karta hai. Service-to-service auth ka standard (service mesh — lesson 06) |

> **Kyu terminate at edge:** TLS handshake user ke *paas* (CDN edge) complete karo → RTT chhota → fast (lesson 01 ka RTT concept). Phir origin tak re-encrypt. Iska matlab: jahan TLS terminate hota hai, **cert + private key wahan rakhni padti hai** (CDN/LB pe).

> ⚠️ **The classic outage:** **expired certificate.** Cert expire → poori site down (browsers connection refuse karte hai). **Fix:** renewal automate karo (ACME/Let's Encrypt, AWS ACM) aur **30 din pehle** alert lagao, 3 din nahi.

**SNI (Server Name Indication):** client ClientHello me hostname batata hai, taaki ek IP kai sites serve kar sake aur server sahi cert chune. **Forward secrecy:** har connection ke liye temporary (ephemeral) keys, taaki baad me server ki private key chori ho toh bhi purana recorded traffic decrypt na ho.

---

## 3. Ye sab milke — ek HTTPS request ka shuruaat

```
1. DNS resolve: example.com → 93.184.216.34   (browser/OS/resolver cache, TTL honor)
2. TCP handshake: 1 RTT
3. TLS handshake: 1-2 RTT (cert verify, key agree)
4. HTTP request finally jaata hai
```
Pehle byte se pehle **2-3 RTT** ka cost (DNS miss ho toh aur bhi) — isliye connection reuse (lesson 01), edge termination (CDN — lesson 07), aur newer protocols (HTTP/3) itne matter karte hai. Ye poora flow lesson 10 (capstone) me end-to-end assemble karenge.

---

## Quick Self-Check
1. DNS kya solve karta hai? Browser cache, OS cache, recursive resolver — har ek kya cache karta hai, aur resolver kyu exist karta hai?
2. Root → TLD → authoritative — har ek kya jaanta hai aur kya jawab deta hai?
3. DNS "record" kya hai? A vs AAAA vs CNAME vs MX vs TXT vs NS — ek-ek line. Apex pe CNAME kyu nahi?
4. TTL kya hai, aur migration se pehle TTL kyu (aur kab) girate hai?
5. DNS-based load balancing kaise kaam karta hai, aur uski 4 problems kya hai?
6. Anycast kya hai, aur CDN ise fast failover ke liye kyu use karta hai (vs DNS)?
7. Certificate me kya hota hai, CA kya hai, aur browser kya-kya verify karta hai?
8. Cert public hai — phir bhi attacker fake server kyu nahi bana sakta?
9. TLS termination vs passthrough vs mTLS — teeno kab?
10. Sabse classic TLS outage kya hai aur fix?

---

## Flow — DNS + TLS end-to-end kaise chalta hai
1. Browser `https://example.com` kholta hai → browser cache → OS cache (+ `/etc/hosts`) → recursive resolver. Resolver ke paas nahi hai toh woh root → `.com` TLD → authoritative nameserver se **A record** laata hai, aur har layer usse **TTL** tak cache karti hai.
2. Us IP ke port 443 pe **TCP handshake** (1 RTT).
3. Browser **ClientHello** bhejta hai (hostname via SNI + apna key share). Server **certificate chain** + apna key share bhejta hai, aur **private key se signature** karke prove karta hai ki cert sach me uska hai.
4. Browser cert check karta hai (trusted CA tak chain, domain match, expire nahi). Dono side same **symmetric key** derive karte hai.
5. Ab HTTP request us symmetric key se **encrypted** jaati hai. Agar beech me LB/CDN hai, toh TLS wahan terminate hota hai (aur zarurat ho toh backend tak re-encrypt).

---
**Next:** `04-Scalability.md` — ab ek server kaafi kyu nahi, aur scale kaise karte hai.