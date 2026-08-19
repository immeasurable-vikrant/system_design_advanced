# 01 — Networking I: Layers & Transport

> **Is lesson me:** Sabse foundational cheezein — client/server/IP/port/socket kya hai, OSI model thoda (aur practical L1-L7 map), **L4 vs L7** (ye ek distinction poore module ko explain karta hai), TCP vs UDP, handshakes aur RTT ka "tax", head-of-line blocking, aur connection reuse. Yahan se sab shuru hota hai.

Prerequisites: none — ye curriculum ki sabse pehli file hai.

---

## 1. Pehle basics — client, server, IP, port, socket

Chalo woh words define karein jo baaki sab jagah use honge:

- **Server** = ek program (aur machine) jo requests ka **wait** karta hai aur respond karta hai. (Express/Django backend, database, sab servers hai.)
- **Client** = jo request **bhejta** hai (browser, mobile app, ya ek server jo doosre server ko call kare).
- **IP address** = internet pe kisi machine ka "address" — jaise ghar ka address. E.g., `93.184.216.34` (IPv4) ya `2606:2800:...` (IPv6). Data isi address pe deliver hota hai.
- **Port** = ek machine pe kaunsa **program** — jaise ghar me kaunsa flat number. Ek machine pe kai servers chal sakte hai, har ek apne port pe:
  ```
  93.184.216.34:443   → HTTPS web server (port 443)
  93.184.216.34:5432  → PostgreSQL (port 5432)  [Topic 2 me dekha]
  93.184.216.34:6379  → Redis (port 6379)       [Topic 3 me dekha]
  ```
- **Socket** = ek specific connection ka endpoint = `IP + port` ka combination. Jab client server se baat karta hai, ek "socket" bana connection banta hai jispe data behta hai.

**Frontend analogy:** Jab tum `fetch("https://api.example.com/users")` karte ho — browser `api.example.com` ko ek IP me resolve karta hai (DNS — lesson 03), phir us IP ke port 443 pe ek socket connection banata hai, aur HTTP request bhejta hai. Ye poora invisible plumbing yahi module hai.

---

## 2. Layers — OSI thoda, aur jo practically matter karta hai

Network communication "layers" me organized hai — har layer apna ek kaam karti hai aur upar-neeche wali se baat karti hai. Poora **OSI model 7 layers** ka hai, but interview me tumhe sirf ye practical map chahiye:

| Layer | Naam | Unit | Yahan kya rehta hai | Kahan milta hai |
|---|---|---|---|---|
| **L7** | Application | Message | HTTP, gRPC, WebSocket, DNS, TLS* | API gateway, Nginx, ALB |
| **L4** | Transport | Segment/Datagram | TCP, UDP, QUIC | NLB, firewalls, ports |
| **L3** | Network | Packet | IP, routing | Routers, subnets, anycast |
| **L2** | Data link | Frame | Ethernet, MAC | Switches, VLANs |

*(TLS technically L4 aur L7 ke beech — "L6-ish" — hai; "TLS terminates at L7" bolna galat nahi.)*

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

**TCP** = reliable, ordered, but slower (guarantees ki keemat). Web (HTTP), databases, file transfer — sab TCP.
**UDP** = fast, no guarantees. Video calls, live games, DNS, metrics.

**Kab UDP sahi hai (ye samajhna important):**
- **Video/voice call** — ek 200ms-late audio frame **bekaar** hai (baat aage badh chuki). Retransmit karne se koi fayda nahi — drop karo, gap chhupao. Isliye UDP.
- **Games** — newest player position bhejo, purani stale position replay mat karo.
- **DNS** — ek chhota request/response, retry app-layer pe sasta hai handshake se.

**Interview line:** *"Video frames UDP pe jaate hai kyunki retransmitted frame use hone ke baad aata hai — better to drop aur conceal. Signalling aur API TCP pe, kyunki 'call ended' message kho jaana unacceptable hai."*

---

## 4. The handshake tax — RTT kyu matter karta hai

Yahan ek super-important concept: **har network round-trip time leta hai, aur ye add up hota hai.**

> **RTT (Round-Trip Time)** = ek message ko client se server jaake, wapas aane me lagne wala time. Same city ~1-5ms, cross-country ~50-150ms. Ye "network ki dsuper speed" pe bhi light-speed ki limit hai — physics.

TCP connection banane se pehle ek **3-way handshake** hota hai (1 RTT "waste" before koi actual data):
```
Client                          Server
  │────── SYN ───────────────────→│   "connect karna hai"
  │←───── SYN-ACK ────────────────│   "ok, ready"
  │────── ACK + data ────────────→│   "confirmed" + finally actual request
         (1 RTT already gaya)
```

Ab isme **TLS handshake** (encryption — lesson 03) add karo:
```
TCP handshake:     1 RTT
TLS 1.2 handshake: 2 RTT
─────────────────────────
= 3 RTT before your request's first byte even goes!
```
Ek 100ms link pe = **300ms pure protocol overhead**, actual kaam se pehle. Cross-country user ke liye ye bahut hai.

**Isi liye ye teen cheezein exist karti hai (poore module me dikhengi):**
1. **Connection reuse (keep-alive)** — ek baar handshake karo, phir usi connection pe kai requests (naya handshake har baar mat karo).
2. **CDN edge** (lesson 07) — TLS handshake user ke *paas* complete karo (RTT chhota).
3. **Newer protocols** — TLS 1.3 (2 RTT), HTTP/3 / QUIC (1 ya 0 RTT — lesson 02).

**Frontend analogy:** Har `fetch` pe naya connection = har baar phone number dial karna, ring hone ka wait, "hello hello" karna, phir baat. Keep-alive = line khuli rakho, baar-baar dial mat karo. Isliye connection pooling (lesson 09) aur keep-alive itne important hai.

---

## 5. Head-of-line (HOL) blocking

TCP bytes **in order** deliver karta hai. Agar beech ka ek piece (segment #3) kho jaye:
```
received: [1][2][ ✗ ][4][5][6]
app ko mila: 1, 2, ...phir kuch nahi jab tak #3 dobara na aaye
```
Segment 4, 5, 6 already aa chuke hai (kernel buffer me), but **use nahi ho sakte** kyunki #3 missing hai. Ye "head-of-line blocking" hai — ek atka hua item peeche walon ko rok deta hai.

Ye especially bura hai lossy mobile networks pe, aur HTTP/2 me (jahan sab ek TCP connection pe multiplexed hota hai — ek lost packet **saari** streams rok deta hai). HTTP/3 (QUIC) isko fix karta hai (lesson 02).

**Frontend analogy:** Ek single-file queue me pehla banda atak gaya (form bhar raha) → poori line ruk gayi, chahe peeche walon ka kaam ready ho. HOL blocking wahi hai network level pe.

---

## 6. Keep-alive & connection reuse (practical)

```
Without keep-alive: handshake + TLS + request + response + close  (HAR baar)
With keep-alive:    handshake ONCE, phir request/response × N
```

Internal service calls me, fresh connection per request aksar request se **3× mehenga** ho sakta hai. Isliye **hamesha reuse karo.**

> ⚠️ **Failure mode:** ek idle connection jise server ne band kar diya but client ko lagta hai abhi khuli hai → intermittent "connection reset" errors. **Fix:** client ka idle-timeout server aur load balancer ke idle-timeout se **kam** rakho.

Ye connection reuse ka concept directly connection pooling (lesson 09) me expand hota hai — jahan hum poola of reusable connections rakhte hai database ke liye.

---

## Quick Self-Check
1. IP, port, socket — teeno ka difference, ek analogy se?
2. L4 aur L7 device me core difference kya (ek line)? Ye kyu itna important hai?
3. TCP vs UDP — kaunsa kab? Video call ke media aur signalling ke liye alag kyu?
4. RTT kya hai? TCP + TLS 1.2 milke kitne RTT lagte hai request se pehle?
5. Head-of-line blocking kya hai — ek analogy se?
6. Keep-alive kyu zaroori, aur "connection reset" error kaise aata hai?

---
**Next:** `02-Networking-II-HTTP-And-RealTime.md` — HTTP versions aur real-time communication.