# 02 — Networking II: HTTP & Real-Time Communication

> **Is lesson me:** HTTP ke versions (1.1 vs 2 vs 3/QUIC) aur kyu "naya hamesha better nahi", important HTTP status codes (502/503/504), aur real-time communication ke options — polling, long-polling, SSE, WebSocket, webhook — kab kaunsa. Ye woh L7 protocols hai jinpe web chalta hai.

Prerequisites: `01-Networking-I-Layers-And-Transport.md`

---

## 1. HTTP kya hai (quick)

**HTTP (HyperText Transfer Protocol)** = L7 protocol jispe web chalta hai — client ek **request** bhejta hai (method + path + headers + optional body), server ek **response** deta hai (status code + headers + body). TCP ke upar chalta hai (usually).

```
Request:                          Response:
GET /api/users/123 HTTP/1.1       HTTP/1.1 200 OK
Host: api.example.com             Content-Type: application/json
Authorization: Bearer xyz         
                                  { "id": 123, "name": "Bob" }
```
Tumne ye frontend me DevTools Network tab me hazaar baar dekha hai — yahi HTTP hai.

---

## 2. HTTP versions — 1.1 vs 2 vs 3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC (UDP)** |
| Concurrency | Ek request per connection at a time | Multiplexed streams | Multiplexed streams |
| Browser workaround | 6 connections/host chahiye | Nahi | Nahi |
| Header compression | None (plaintext, repeated) | HPACK | QPACK |
| HOL blocking | HTTP level pe | TCP level pe | **None** |
| Handshake RTTs | 1 (TCP) + 2 (TLS) | same | 1, ya 0 on resumption |
| Connection migration | No | No | **Yes** (wifi→cellular survive) |
| Format | Text | Binary | Binary |

### HTTP/1.1 — one-at-a-time problem
Ek connection pe ek time ek hi request. Isliye browsers ek host ke liye **6 parallel connections** kholte the (taaki 6 cheezein saath aaye). Workaround, elegant nahi.

### HTTP/2 — multiplexing
Ek hi connection pe **kai requests/responses ek saath** (multiplexed "streams"). Plus header compression (HPACK). Datacenter/clean networks pe excellent. **gRPC** isi pe chalta hai.

> **gRPC kya hai (quick):** Google ka ek high-performance RPC framework jo HTTP/2 pe chalta hai. "RPC" = Remote Procedure Call — ek service doosri service ka function directly call kar rahi ho jaise local ho. Internal service-to-service communication me popular (REST se fast, binary protocol "protobuf" use karta hai). Yaad rakho: **gRPC = internal microservices ke beech, HTTP/2 pe.**

### HTTP/2 ka subtle catch (interview gold)
HTTP/2 ne HTTP-level HOL blocking solve ki (multiplexing se) — **but sab kuch ek TCP connection pe** hai, toh ek lost packet **saari streams** ko rok deta hai (TCP-level HOL blocking — lesson 01). Clean datacenter link pe great. **Lossy mobile link pe HTTP/2 kabhi HTTP/1.1 se bhi bura** ho sakta hai — kyunki 6 connections me loss sirf 1/6th requests rokta hai, jabki HTTP/2 me ek loss sabko rokta hai.

### HTTP/3 — QUIC (the real fix)
QUIC UDP pe reliability dobara build karta hai, **per-stream** loss recovery ke saath — ek stream ka lost packet baaki streams ko nahi rokta. Plus fewer handshake RTTs aur **connection migration** (phone wifi se LTE switch kare toh same connection chalti rehti hai, dobara handshake nahi).

**Interview line:** *"HTTP/2 internal services ke beech (datacenter me loss negligible, multiplexing + header compression sasta milta hai — gRPC bhi yahi use karta hai). HTTP/3 edge pe mobile clients ke liye, jahan packet loss aur network switching normal hai — per-stream recovery aur 0-RTT resumption matter karte hai."*

---

## 3. HTTP status codes — jo interview me aate hai

Basics tumhe pata honge (200 OK, 404 Not Found, 401 Unauthorized). Ye 3 error codes specifically infra me important hai (LB/gateway ke context me):

| Code | Matlab | Kya point karta hai |
|---|---|---|
| **502** Bad Gateway | Upstream ne kuch **invalid** return kiya | Broken backend |
| **503** Service Unavailable | Service **unavailable** ya load shed kar rahi | Saturated backend (ya intentional) |
| **504** Gateway Timeout | Upstream ne gateway ke timeout me **respond nahi kiya** | Slow backend |

Teeno alag problems batate hai: **502 = tuta hua, 503 = bhara hua, 504 = slow.** Ye distinction debugging me crucial hai (gateway/proxy ke logs me ye dikhte hai).

Aur `429 Too Many Requests` — rate limiting (Topic 3 lesson 07-08) — yaad hai.

---

## 4. Real-time communication — server se client updates

Aksar tumhe server se client ko **live updates** bhejni hoti hai (chat message, notification, live score, progress bar). HTTP normally client-pull hai (client maange tab milta hai). Real-time ke 5 options:

| | Polling | Long polling | SSE | WebSocket | Webhook |
|---|---|---|---|---|---|
| Direction | Client pulls | Client pulls (held) | Server → client | **Bidirectional** | Server → server |
| Protocol | HTTP | HTTP | HTTP | Upgraded (ws://) | HTTP POST |
| Connection | New each time | Held until data | One long-lived | One long-lived | Per event |
| Data | Any | Any | Text only | Text or binary | Any |
| Auto-reconnect | n/a | Manual | **Built-in** | Manual | n/a |
| Best for | Slow-changing, simplest | Legacy real-time | Notifications, feeds, token streaming | Chat, collab, games | 3rd-party integrations |

### Samjho har ek:
- **Polling** — client har X seconds pe "kuch naya?" poochta hai. Simple but wasteful (zyada tar "nahi" milta). Slow-changing data ke liye theek.
- **Long polling** — client poochta hai, server **rok ke rakhta hai** jab tak data na aaye, phir respond. Data aate hi client dobara poochta hai. Legacy real-time.
- **SSE (Server-Sent Events)** — ek long-lived HTTP connection jispe **server push karta rehta hai** (text). Auto-reconnect built-in. Plain HTTP — har proxy se kaam karta hai.
- **WebSocket** — ek persistent, **bidirectional** connection (`ws://`), HTTP se "upgrade" hoke. Dono taraf high-rate messages ke liye (chat, collaborative editing, games).
- **Webhook** — server-to-server: "ye event hua" ka HTTP POST doosre server ko (jaise Stripe tumhe payment-success webhook bhejta hai).

**Interview line:** *"Agar updates sirf server→client behte hai, main SSE use karunga — plain HTTP, auto-reconnect with event IDs, har proxy se kaam karta hai. WebSocket sirf tab jab client bhi high rate pe messages bheje (chat/collab), warna main ek stream + ek POST path dono ka cost de raha hunga."*

> ⚠️ **Long-lived connections scale karna (SSE/WebSocket):** ek connection = server pe **state**. Matlab sticky routing chahiye, ya ek shared pub/sub (Redis/Kafka) taaki koi bhi server kisi bhi user ko push kar sake. Aur budget ~10-50KB memory per connection — 100K connections = real memory (~30GB across fleet), jo tumhare server count ko decide karta hai. (Ye Topic 3 ka Redis pub/sub + Topic 2 ka replication connect karta hai.)

**Frontend analogy:** Tumne shayad `EventSource` (SSE) ya `new WebSocket()` use kiya hai. SSE = ek-taraffa live feed (notifications). WebSocket = do-taraffa (chat). Polling = `setInterval(() => fetch(...), 5000)` — kaam karta hai but wasteful.

---

## Quick Self-Check
1. HTTP request aur response me kya-kya hota hai (parts)?
2. HTTP/1.1 me 6 connections/host kyu khulte the?
3. HTTP/2 ka multiplexing kya deta hai, aur uska subtle catch (TCP HOL) kya?
4. HTTP/3 (QUIC) kaunse 2 bade problems fix karta hai?
5. gRPC kya hai aur kis layer pe chalta hai?
6. 502 vs 503 vs 504 — teeno kya batate hai?
7. SSE vs WebSocket — kab kaunsa? Long-lived connection scale karne me kya dhyan?

---
**Next:** `03-Networking-III-DNS-And-TLS.md` — naam se IP tak (DNS), aur encryption (TLS).