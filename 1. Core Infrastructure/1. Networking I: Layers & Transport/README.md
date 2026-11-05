# 01 — Networking I: Layers & Transport

> **Is lesson me:** Sabse foundational cheezein — client/server/IP/port/socket kya hai, OSI model (practical L1-L7 map), **L4 vs L7** (ye ek distinction poore module ko explain karta hai), TCP vs UDP, handshakes aur RTT ka "tax", multiplexing aur head-of-line blocking, aur connection reuse. Yahan se sab shuru hota hai.

Prerequisites: none — ye curriculum ki sabse pehli file hai.

---

## 1. Pehle basics — client, server, IP, port, socket

Chalo woh words define karein jo baaki sab jagah use honge:

- **Server** = ek program (aur machine) jo requests ka **wait** karta hai aur respond karta hai. (Express/Django backend, database, sab servers hai.)
- **Client** = jo request **bhejta** hai (browser, mobile app, ya ek server jo doosre server ko call kare).
- **IP address** = internet pe kisi machine (exactly bolein toh uske network interface) ka "address" — jaise ghar ka address. Har packet pe "from" aur "to" IP likha hota hai, routers isi ko dekh ke packet aage bhejte hai.
  - **IPv4** = 32-bit, 4 numbers dots se (`93.184.216.34`) → sirf ~4.3 billion addresses, jo khatam ho chuke hai.
  - **IPv6** = 128-bit, hex colons se (`2606:2800:220:1::248`) → practically unlimited addresses; IPv4 ki kami fix karne ke liye bana.
  - *(Bonus: ghar/office me tumhara laptop ek **private IP** rakhta hai jaise `192.168.1.5`; router saare devices ko ek public IP ke peeche chhupata hai — isko **NAT** kehte hai. IPv4 ki kami isi jugaad se chal rahi hai.)*
- **Port** = ek machine pe kaunsa **program** — jaise ghar me kaunsa flat number. Ek machine pe kai servers chal sakte hai, har ek apne port pe:
  ```
  93.184.216.34:443   → HTTPS web server (port 443)
  93.184.216.34:5432  → PostgreSQL (port 5432)  [Topic 2 me dekha]
  93.184.216.34:6379  → Redis (port 6379)       [Topic 3 me dekha]
  ```
  Port ek 16-bit number hai (**0–65535**); 0–1023 "well-known" ports hai (80 HTTP, 443 HTTPS, 22 SSH). **Client side bhi ek port use karta hai** — OS ek random temporary ("ephemeral") port de deta hai, jaise `54321`, taaki response wapas sahi program tak pahunche.
- **Socket** = ek connection ka endpoint jo OS banata hai = `IP + port`. Ek TCP connection ko ye **4-tuple** uniquely pehchanta hai: `(client IP, client port, server IP, server port)`.
  - **Kab aur kaise bana?** Har `fetch()` ke neeche browser OS se ek TCP socket khulwata hai (ya ek purana khula hua reuse karta hai), aur HTTP request usi socket pe bytes ki tarah likhi jaati hai. Network tab ye level nahi dikhata kyunki woh **requests** dikhata hai, sockets nahi — but Chrome DevTools Network tab me right-click on column header → **"Connection ID"** enable karo: same ID wali requests same socket pe gayi (connection reuse live dikh jayega).
  - **"ws" (WebSocket) alag cheez hai** — naam me "socket" hai but ye ek **L7 protocol** hai: ek normal HTTP request se shuru hoke ek long-lived, two-way connection ban jaata hai (TCP socket ke upar). Plain "socket" = OS-level concept jo **har** network call ke neeche hai.

**Frontend analogy:** Jab tum `fetch("https://api.example.com/users")` karte ho — browser `api.example.com` ko ek IP me resolve karta hai (DNS — lesson 03), phir us IP ke port 443 pe ek socket connection banata hai, aur HTTP request bhejta hai. Ye poora invisible plumbing yahi module hai.

---

## 2. Layers — OSI model aur jo practically matter karta hai

Network communication "layers" me organized hai — har layer apna ek kaam karti hai aur upar-neeche wali se baat karti hai. **OSI model** ki saari 7 layers, ek-ek line me:

| Layer | Naam | Ek line me kaam | Example |
|---|---|---|---|
| **L7** | Application | App ka actual "language" / protocol | HTTP, gRPC, WebSocket, DNS |
| **L6** | Presentation | Data ka format — encoding, compression, encryption | TLS, gzip, UTF-8 |
| **L5** | Session | Do sides ke beech "session" open/maintain/close karna | TLS sessions, RPC sessions |
| **L4** | Transport | Program-to-program delivery (ports), reliability, ordering | TCP, UDP, QUIC |
| **L3** | Network | Machine-to-machine delivery across networks (IP addresses, routing) | IP, routers |
| **L2** | Data link | Same local network pe ek device se agle device tak (MAC addresses) | Ethernet, Wi-Fi frames, switches |
| **L1** | Physical | Raw bits ko signal banana — bijli, light, radio waves | Cables, fiber, Wi-Fi radio |

> **Reality check:** Real internet **TCP/IP model** pe chalta hai jisme L5 aur L6 alag nahi hote — woh L7 me mil jaate hai. Isliye engineers practically sirf **L7, L4, L3, L2** bolte hai. TLS ko "L6-ish" kehte hai; "TLS terminates at L7" bolna galat nahi.

Practical map (jo interview me chahiye):

| Layer | Unit | Kahan milta hai |
|---|---|---|
| **L7** | Message | API gateway, Nginx, ALB |
| **L4** | Segment/Datagram | NLB, firewalls, ports |
| **L3** | Packet | Routers, subnets, anycast |
| **L2** | Frame | Switches, VLANs |

> **Ratne ki zarurat nahi.** Bas ye ek line yaad rakho: **"L4 devices sirf IP + port dekhte hai. L7 devices poori HTTP request padh sakte hai."** Yehi ek difference load balancers, proxies, gateways ki saari capabilities explain karta hai (aage lessons me baar-baar aayega).

**L4 vs L7 — concrete:**
```
L4 (transport level): "port 443 pe traffic aaya → kisi backend server ko de do"
                       (pata nahi /api/users hai ya /static/logo.png)

L7 (application level): "GET /api/users aaya → API server pool ko
                        GET /static/logo.png → static/CDN pool ko"
                        (poori request padh ke smart routing)
```
L4 fast hai (kam parsing), L7 flexible hai (poora HTTP samajhta hai, but zyada CPU). Ye trade-off har intermediary (LB/proxy/gateway) me aata hai.

**Frontend analogy:** L4 = ek courier jo sirf envelope ka pin-code dekh ke sort karta hai (fast, dumb). L7 = ek assistant jo envelope khol ke, letter padh ke, sahi department ko bhejta hai (smart, slow).

---

## 3. TCP vs UDP — data bhejne ke 2 tarike (L4)

Transport layer pe 2 main protocols hai. Ye choice har real-time/streaming design me aata hai:

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Handshake zaroori (3-way) | Connectionless, bas bhej do |
| Reliability | Lost data retransmit karta hai | None — loss silent (gaya toh gaya) |
| Ordering | Guaranteed (order me deliver) | None (koi bhi order) |
| Congestion control | Yes (network ko flood nahi karta) | No |
| Header overhead | 20-60 bytes | 8 bytes (light) |
| Use for | **Bytes lose nahi hone chahiye** | **Late data bekaar hai** |

**TCP** = reliable, ordered, but slower (guarantees ki keemat). Web (HTTP/1.1, HTTP/2), databases, file transfer — sab TCP.
**UDP** = fast, no guarantees. Video calls, live games, DNS, metrics.

**Kab UDP sahi hai (ye samajhna important):**
- **Video/voice call** — ek 200ms-late audio frame **bekaar** hai (baat aage badh chuki). Retransmit karne se koi fayda nahi — drop karo, gap chhupao. Isliye UDP.
- **Games** — newest player position bhejo, purani stale position replay mat karo.
- **DNS** — ek chhota request/response, retry app-layer pe sasta hai handshake se. Matlab: DNS query ek hi packet me fit ho jaati hai, toh agar jawab nahi aaya, client bas timeout ke baad wahi packet **dobara bhej deta hai** — ye TCP ke reliability machinery jaisa hi kaam karta hai, bina har lookup pe 1 RTT ka handshake diye. *(Bada response ho toh DNS TCP pe bhi switch kar leta hai.)*

**Interview line:** *"Video frames UDP pe jaate hai kyunki retransmitted frame use hone ke baad aata hai — better to drop aur conceal. Signalling aur API TCP pe, kyunki 'call ended' message kho jaana unacceptable hai."*

**2026 me kaunsa version?**
- **TCP ka koi "version number" nahi hota** jaise HTTP ka — ek hi TCP hai (original 1981 spec, 2022 me RFC 9293 me consolidate hua), bas upar se extensions aur congestion-control algorithms badalte rehte hai (Linux default **CUBIC**, Google jaise log **BBR**).
- **HTTP:** browser traffic ka majority **HTTP/2** pe hai, **HTTP/3** (QUIC) tezi se badh raha hai (bade sites/CDNs pe default), aur **HTTP/1.1** abhi bhi internal services, bots aur purane clients me common hai. (Live numbers: Cloudflare Radar.)

---

## 4. The handshake tax — RTT kyu matter karta hai

Yahan ek super-important concept: **har network round-trip time leta hai, aur ye add up hota hai.**

> **RTT (Round-Trip Time)** = ek message ko client se server jaake, wapas aane me lagne wala time. Same city ~1-5ms, cross-country ~50-150ms. Network kitna bhi fast ho, light-speed ki limit hai — physics.

TCP connection banane se pehle ek **3-way handshake** hota hai:
```
Client                          Server
  │────── SYN ───────────────────→│   "connect karna hai"         (0.5 RTT)
  │←───── SYN-ACK ────────────────│   "ok, ready"                 (1.0 RTT)
  │────── ACK + data ────────────→│   "confirmed" + actual request
         ↑ yahan tak 1 RTT "waste" hua
```

**"1.5 RTT nahi hona chahiye?"** — Handshake ke messages total 1.5 RTT ke hai, haan. But teesra message (ACK) **akela nahi jaata — request ka data usi ke saath jaata hai** (piggyback). Wo aakhri 0.5 RTT toh request ko waise bhi chalna hi tha, handshake ho ya na ho. Isliye **extra cost = 1 RTT**. Full timeline: 1 RTT handshake + 1 RTT request/response = pehla response byte **2 RTT** pe.

Ab isme **TLS handshake** add karo.

> **TLS (Transport Layer Security)** = TCP ke upar ek layer jo connection ko **encrypt** karti hai (beech me koi padh na sake), server ko **verify** karti hai (certificate se — "ye sach me google.com hai"), aur data ko **tamper-proof** banati hai. **HTTPS = HTTP over TLS.** (Detail lesson 03 me.)

```
TCP handshake:     1 RTT
TLS 1.2 handshake: 2 RTT
─────────────────────────
= 3 RTT before your request's first byte even goes!  (response = 4 RTT pe)
```
Ek 100ms link pe = **300ms pure protocol overhead**, actual kaam se pehle. Cross-country user ke liye ye bahut hai.

**Isi liye ye teen cheezein exist karti hai (poore module me dikhengi):**
1. **Connection reuse (keep-alive)** — ek baar handshake karo, phir usi connection pe kai requests (naya handshake har baar mat karo).
2. **CDN edge** (lesson 07) — TLS handshake user ke *paas* complete karo (RTT chhota).
3. **Newer protocols** — TLS 1.3 (sirf 1 RTT ka TLS → TCP+TLS total 2 RTT), HTTP/3 / QUIC (transport + TLS ek saath: 1 RTT, repeat visit pe 0 RTT — lesson 02).

**Frontend analogy:** Har `fetch` pe naya connection = har baar phone number dial karna, ring hone ka wait, "hello hello" karna, phir baat. Keep-alive = line khuli rakho, baar-baar dial mat karo. Isliye connection pooling (lesson 09) aur keep-alive itne important hai.

---

## 5. Simplex, duplex, multiplex — aur head-of-line (HOL) blocking

Pehle teen confusing words clear kar lo — ye do **alag** sawaalon ke jawab hai:

**Duplex = data kis direction me ja sakta hai?**
- **Simplex** = sirf ek direction (TV/radio broadcast — tum wapas nahi bol sakte).
- **Half-duplex** = dono direction, but ek time pe ek hi (walkie-talkie — "over").
- **Full-duplex** = dono direction ek saath (phone call). **TCP aur WebSocket full-duplex hai.** Classic HTTP request→response pattern half-duplex jaisa behave karta hai.

**Multiplexing = ek connection pe kitni alag conversations?**
- **Multiplexing** = kai independent streams (requests) ek hi connection share karte hai — har stream ke chhote-chhote tukde (frames) aapas me mix hoke jaate hai, har frame pe "stream ID" laga hota hai taaki dusri side alag kar sake.
  ```
  HTTP/1.1: ek connection pe ek waqt ek request  → browser 6 connections kholta hai per host
  HTTP/2:   ek connection, kai streams mixed     → [A1][B1][C1][A2][B2]...
  ```
- Toh **duplex ≠ multiplex**: duplex = *direction*, multiplex = *sharing*. HTTP/2 ek full-duplex TCP connection pe multiplexing karta hai.

**Ab HOL blocking.** TCP bytes **in order** deliver karta hai. Agar beech ka ek piece (segment #3) kho jaye:
```
received: [1][2][ ✗ ][4][5][6]
app ko mila: 1, 2, ...phir kuch nahi jab tak #3 dobara na aaye
```
Segment 4, 5, 6 already aa chuke hai (kernel buffer me), but **use nahi ho sakte** kyunki #3 missing hai. Ye "head-of-line blocking" hai — ek atka hua item peeche walon ko rok deta hai.

**HTTP/2 me ye kyu aur bura hai:** HTTP/2 ki streams A, B, C alag hai — but **TCP ko ye pata hi nahi**, TCP ke liye sab ek hi byte-stream hai. Toh agar stream A ka ek packet kho gaya, TCP **B aur C ke ready data ko bhi rok leta hai** jab tak A ka packet retransmit na ho. Lossy mobile network pe (jahan packet loss common hai) ye baar-baar hota hai — kabhi HTTP/2 ek connection pe HTTP/1.1 ke 6 connections se bhi slow ho jaata hai.

**HTTP/3 (QUIC) fix:** QUIC UDP ke upar bana hai aur **har stream ki ordering alag** track karta hai. Stream A ka packet kho jaye toh sirf A rukta hai; B aur C chalte rehte hai. (Detail lesson 02.)

**Frontend analogy:** Ek single-file queue me pehla banda atak gaya (form bhar raha) → poori line ruk gayi, chahe peeche walon ka kaam ready ho. HTTP/3 = har kaam ke liye alag counter, ek atka toh baaki nahi rukte.

---

## 6. Keep-alive & connection reuse (practical)

```
Without keep-alive: handshake + TLS + request + response + close  (HAR baar)
With keep-alive:    handshake ONCE, phir request/response × N
```

Internal service calls me, fresh connection per request aksar request se **3× mehenga** ho sakta hai. Isliye **hamesha reuse karo.**

**Keep-alive kahan set hota hai, kisko batate hai?**
- **HTTP/1.1 me ye by default ON hai** — client `Connection: keep-alive` header bhejta hai (ya bas default man liya jaata hai); `Connection: close` bhejo toh band. HTTP/2, HTTP/3 me ek connection reuse karna design me hi hai.
- **Browser:** khud handle karta hai, tumhe kuch nahi karna.
- **Server:** kitni der idle connection khula rakhe — e.g. Nginx `keepalive_timeout`, Node `server.keepAliveTimeout` (default 5s).
- **Server-to-server client:** tumhara HTTP client/agent — e.g. Node `new http.Agent({ keepAlive: true })` (Node 19+ me default on), Python `requests.Session()` / `httpx.Client()` (har call pe `requests.get()` naya connection = no reuse).
- **Load balancer:** apna idle timeout hota hai (e.g. AWS ALB default 60s).
- *(Naam same, cheez alag: **TCP keepalive** (`SO_KEEPALIVE`) = OS-level "zinda ho?" probe packets dead connection detect karne ke liye. HTTP keep-alive = connection reuse.)*

> ⚠️ **Failure mode:** ek idle connection jise server ne band kar diya but client ko lagta hai abhi khuli hai → intermittent "connection reset" errors. **Fix:** client ka idle-timeout server aur load balancer ke idle-timeout se **kam** rakho. *(Classic bug: Node backend ALB ke peeche, Node ka 5s timeout < ALB ka 60s → ALB band connection pe request bhejta hai → 502s. Fix: Node ka `keepAliveTimeout` ALB se zyada, jaise 65s.)*

Ye connection reuse ka concept directly connection pooling (lesson 09) me expand hota hai — jahan hum pool of reusable connections rakhte hai database ke liye.

---

## Flow — ek request end-to-end kaise chalti hai
1. `fetch("https://api.example.com/users")` → browser DNS se `api.example.com` ka IP nikalta hai.
2. OS ek **socket** kholta hai (client IP + random ephemeral port → server IP:443) aur **TCP 3-way handshake** karta hai (1 RTT), phir **TLS handshake** (1-2 RTT) se connection encrypt hota hai.
3. HTTP request (L7) us socket me bytes ki tarah likhi jaati hai → TCP use segments me todta hai (L4), IP packets banata hai (L3), aur woh Wi-Fi/Ethernet frames → signals ban ke wire/hawa me jaate hai (L2/L1).
4. Server pe ulta hota hai: L1 → L7 tak data jud ke HTTP request banti hai. App response likhta hai, jo wahi raasta pakad ke wapas aata hai. Lost segment TCP retransmit karta hai, order bhi wahi sambhalta hai.
5. Response ke baad socket **keep-alive** se khula rehta hai, taaki agli request bina naye handshake ke chale.

---

## Quick Self-Check
1. IP, port, socket — teeno ka difference, ek analogy se? IPv4 vs IPv6?
2. Network tab me socket kyu nahi dikhta, aur WebSocket "socket" se kaise alag hai?
3. L1 se L7 — har layer ek line me. Practically L5/L6 kyu ignore hoti hai?
4. L4 aur L7 device me core difference kya (ek line)? Ye kyu itna important hai?
5. TCP vs UDP — kaunsa kab? Video call ke media aur signalling ke liye alag kyu? DNS UDP kyu?
6. RTT kya hai? 3-way handshake 1 RTT ka kyu gina jaata hai, 1.5 ka kyu nahi? TCP + TLS 1.2 = kitne RTT?
7. Simplex / half-duplex / full-duplex / multiplex — farak?
8. HTTP/2 me ek lost packet saari streams kyu rokta hai, aur HTTP/3 kaise fix karta hai?
9. Keep-alive kahan-kahan configure hota hai, aur "connection reset" / ALB 502 kaise aata hai?

---
**Next:** `02-Networking-II-HTTP-And-RealTime.md` — HTTP versions aur real-time communication.