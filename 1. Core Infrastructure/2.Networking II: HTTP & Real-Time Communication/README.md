# 02 — Networking II: HTTP & Real-Time Communication

> **Is lesson me:** HTTP ke versions (1.1 vs 2 vs 3/QUIC) aur kyu "naya hamesha better nahi", important HTTP status codes (502/503/504), aur real-time communication ke options — short polling, long polling, SSE, WebSocket, webhook — kab kaunsa, real-life examples ke saath, aur ye OS-level sockets se kaise jude hai. Ye woh L7 protocols hai jinpe web chalta hai.

Prerequisites: `01-Networking-I-Layers-And-Transport.md`

---

## 1. HTTP kya hai (quick)

**HTTP (HyperText Transfer Protocol)** = L7 protocol jispe web chalta hai — client ek **request** bhejta hai (method + path + headers + optional body), server ek **response** deta hai (status code + headers + body). TCP ke upar chalta hai (HTTP/1.1, HTTP/2), ya QUIC/UDP ke upar (HTTP/3).

```
Request:                          Response:
GET /api/users/123 HTTP/1.1       HTTP/1.1 200 OK
Host: api.example.com             Content-Type: application/json
Authorization: Bearer xyz         
                                  { "id": 123, "name": "Bob" }
```
Tumne ye frontend me DevTools Network tab me hazaar baar dekha hai — yahi HTTP hai.

> **REST kya hai phir?** REST HTTP ke upar ek **API design style** hai (resources + methods: `GET /users/123`, `POST /orders`). REST alag protocol nahi hai — HTTP hi hai, bas ek convention ke saath.

---

## 2. HTTP versions — 1.1 vs 2 vs 3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC (UDP)** |
| Concurrency | Ek request per connection at a time | Multiplexed streams | Multiplexed streams |
| Browser workaround | 6 connections/host chahiye | Nahi | Nahi |
| Header compression | None (plaintext, repeated) | HPACK | QPACK |
| HOL blocking | HTTP level pe | TCP level pe | **None** |
| Handshake RTTs | 1 (TCP) + 1-2 (TLS 1.3 / 1.2) | same | 1, ya 0 on resumption |
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
| **502** Bad Gateway | Upstream ne kuch **invalid** return kiya (ya connection tod diya) | Broken backend |
| **503** Service Unavailable | Service **unavailable** ya load shed kar rahi | Saturated backend (ya intentional) |
| **504** Gateway Timeout | Upstream ne gateway ke timeout me **respond nahi kiya** | Slow backend |

Teeno alag problems batate hai: **502 = tuta hua, 503 = bhara hua, 504 = slow.** Ye distinction debugging me crucial hai (gateway/proxy ke logs me ye dikhte hai).

Aur `429 Too Many Requests` — rate limiting (Topic 3 lesson 07-08) — yaad hai.

---

## 4. Real-time communication — server se client updates

Aksar tumhe server se client ko **live updates** bhejni hoti hai (chat message, notification, live score, progress bar). HTTP normally **client-pull** hai (client maange tab milta hai). Real-time ke 5 options:

### 4.1 Inko kehte kya hai? (naming & topic)

System design me ye sab ek hi topic ke andar aate hai: **"Real-time communication patterns"** (ya **"push vs pull"**, ya **"client-server communication patterns"**). Interview me sawaal aksar aise aata hai: *"Client ko live updates kaise doge?"*

| Group | Kaun | Idea |
|---|---|---|
| **Pull-based** (client poochta hai) | Short polling, Long polling | Client hi baar-baar request bhejta hai |
| **Push-based, persistent connection** | SSE, WebSocket | Ek connection khula rehta hai, server jab chahe bhej deta hai |
| **Server-to-server push / callback** | Webhook | Event hone pe ek server doosre server ko HTTP POST karta hai |

Naming notes:
- "Polling" ko aksar **short polling** kehte hai (long polling se alag dikhane ke liye).
- Long polling ka purana naam **"Comet"** tha (2000s me, jab WebSocket nahi tha).
- "WebSocket" aur "websockets" ek hi cheez hai — protocol ka naam **WebSocket** hai, URL `ws://` / `wss://` (secure).

### 4.2 Comparison

| | Short polling | Long polling | SSE | WebSocket | Webhook |
|---|---|---|---|---|---|
| Direction | Client pulls | Client pulls (held) | Server → client | **Bidirectional** | Server → server |
| Protocol | HTTP | HTTP | HTTP | Upgraded (ws://) | HTTP POST |
| Connection | New request each time | Held until data | One long-lived | One long-lived | Per event |
| Data | Any | Any | Text only | Text or binary | Any |
| Auto-reconnect | n/a | Manual | **Built-in** | Manual | n/a (sender retries) |
| Best for | Slow-changing, simplest | Legacy real-time, queues | Notifications, feeds, token streaming | Chat, collab, games | 3rd-party integrations |

### 4.3 Har ek — kya hai + real-life example

**Short polling** — client har X seconds pe "kuch naya?" poochta hai. Simple but wasteful (zyada tar "nahi" milta).
- *Real life:* UPI/payment ka **"Checking payment status…"** screen — frontend har 2-3 sec `GET /payments/:id/status` maarta hai. **Async job status** — "Export CSV" dabaya → server `jobId` deta hai → frontend `GET /jobs/:id` poll karta hai jab tak `done` na aaye. React Query ka `refetchInterval` = short polling.
- *Tumhe kyu "dikha nahi":* tumne shayad kiya hai, bas naam nahi diya — `setInterval(() => fetch(...))` hi short polling hai.

**Long polling** — client poochta hai, server **jawab rok ke rakhta hai** jab tak data na aaye (ya timeout, e.g. 20-30s), phir respond. Client turant dobara poochta hai.
- *Real life:* **Telegram Bot API `getUpdates`**, **AWS SQS** `ReceiveMessage` with `WaitTimeSeconds=20` (empty queue pe baar-baar khaali response se bachne ke liye). Browser apps me aaj kal rare — SSE/WebSocket ne replace kar diya, isliye tumhe kam dikha.

**SSE (Server-Sent Events)** — ek long-lived HTTP response jo **khatam hi nahi hota**; server usme events likhta rehta hai (text). Auto-reconnect built-in. Plain HTTP — har proxy se kaam karta hai.
- *Real life:* **ChatGPT/Claude ka token-by-token streaming** (OpenAI/Anthropic API `stream: true` = SSE), live notifications, stock/score tickers, deploy logs live. DevTools me request open karo → **"EventStream"** tab dikhega.

**WebSocket** — ek persistent, **bidirectional** connection, HTTP se "upgrade" hoke. Dono taraf high-rate messages.
- *Real life:* **Slack, WhatsApp Web, Figma/Google Docs live collab, multiplayer games, crypto exchange live order book.** DevTools Network → **"WS"** filter → **Messages** tab me frames dikhte hai.

**Webhook** — server-to-server: "ye event hua" ka HTTP POST doosre server ke URL pe.
- *Real life:* **Stripe/Razorpay** payment success, **GitHub** push event → CI trigger, Shopify order created.

### 4.4 Tumhari company ka case (REST + incoming webhooks)

```
Business X (unka server) ──HTTP POST /webhooks/x──→ Tumhara server
                                                    → 2xx turant return
                                                    → kaam queue me daalo
```
- Ye **push** hai, but **server-to-server**. Tumhara server yahan "receiver" hai — X ko tumhe poll nahi karna padta, aur tumhe X ko poll nahi karna padta. Isliye tumhe polling ki zarurat hi nahi padi.
- Agar webhook na hota, toh tumhe X ko **short poll** karna padta: `GET /x/orders?since=...` har minute — wasteful aur late.
- Webhook ke 3 must-haves (bas yaad rakho): **signature verify karo** (HMAC header se — koi bhi tumhare URL pe fake POST na kar de), **jaldi 2xx do aur kaam background me karo** (warna sender timeout samajh ke retry karega), **idempotent raho** (same event dobara aa sakta hai — `event_id` se dedupe).
- Aur jab tumhare **frontend** ko webhook ka result dikhana ho (e.g. "payment received"), tab wahi frontend-side choice wapas aati hai: short poll, SSE ya WebSocket. **Webhook backend tak update laata hai; browser tak le jaane ke liye inme se ek chahiye.**

### 4.5 OS-level socket se kya rishta hai?

**Sab ke sab TCP socket ke upar chalte hai** (lesson 01). Farak sirf ye hai ki **socket kitni der khula rehta hai aur us pe kaun kab likhta hai:**

| Pattern | Socket pe kya hota hai |
|---|---|
| Short polling | Har poll ek naya HTTP request — keep-alive ho toh **wahi socket reuse**, warna har baar naya |
| Long polling | Socket pe request gayi, server **response likhne me der** karta hai; socket beech me idle khula |
| SSE | Ek socket, ek response jo kabhi "end" nahi hota — server usme chunks likhta rehta hai |
| WebSocket | Socket pe pehle HTTP "Upgrade" request; uske baad **wahi socket** HTTP chhod ke WebSocket frames bolta hai — dono side likhte hai |
| Webhook | Ulta direction: **X ka server** tumhare server ke port 443 pe socket kholta hai |

Isliye: "socket" = OS-level pipe (har cheez ke neeche). "WebSocket" = us pipe pe chalne wala ek L7 protocol.

**Interview line:** *"Agar updates sirf server→client behte hai, main SSE use karunga — plain HTTP, auto-reconnect with event IDs, har proxy se kaam karta hai. WebSocket sirf tab jab client bhi high rate pe messages bheje (chat/collab), warna main ek stream + ek POST path dono ka cost de raha hunga."*

> ⚠️ **Long-lived connections scale karna (SSE/WebSocket):** ek connection = server pe **state** (ek khula socket). Matlab sticky routing chahiye, ya ek shared pub/sub (Redis/Kafka) taaki koi bhi server kisi bhi user ko push kar sake. Aur budget ~10-50KB memory per connection — 100K connections = real memory (~30GB across fleet), jo tumhare server count ko decide karta hai. (Ye Topic 3 ka Redis pub/sub + Topic 2 ka replication connect karta hai.)

**Frontend analogy:** Tumne shayad `EventSource` (SSE) ya `new WebSocket()` use kiya hai. SSE = ek-taraffa live feed (notifications). WebSocket = do-taraffa (chat). Polling = `setInterval(() => fetch(...), 5000)` — kaam karta hai but wasteful.

---

## Flow — ek request end-to-end kaise chalti hai
1. **Choose pattern:** normal REST call = ek request/response; live updates chahiye toh poll / SSE / WebSocket; server-to-server event = webhook.
2. **Connection:** DNS → OS socket → TCP + TLS handshake (HTTP/3 me QUIC ek saath karta hai) — ya pehle se khula keep-alive socket reuse.
3. **HTTP request** us socket pe likhi jaati hai (HTTP/2/3 me ek stream ke roop me, baaki streams ke saath multiplexed).
4. **Server respond karta hai:** turant (REST/short poll), der se (long poll), chunks me lagataar (SSE), ya `101 Switching Protocols` deke socket ko WebSocket bana deta hai.
5. **Beech me koi gateway/LB** ho aur backend toot/bhar/slow ho jaye → client ko 502 / 503 / 504 milta hai.



## Quick Self-Check
1. HTTP request aur response me kya-kya hota hai (parts)? REST aur HTTP me farak?
2. HTTP/1.1 me 6 connections/host kyu khulte the?
3. HTTP/2 ka multiplexing kya deta hai, aur uska subtle catch (TCP HOL) kya?
4. HTTP/3 (QUIC) kaunse 2 bade problems fix karta hai?
5. gRPC kya hai aur kis layer pe chalta hai?
6. 502 vs 503 vs 504 — teeno kya batate hai?
7. Short polling / long polling / SSE / WebSocket / webhook — kaun pull, kaun push? Har ek ka ek real example?
8. Webhook receive karte waqt kaunse 3 cheezein zaroori hai?
9. In sab ka OS-level socket se kya rishta hai? WebSocket aur socket me farak?
10. SSE vs WebSocket — kab kaunsa? Long-lived connection scale karne me kya dhyan?

---
**Next:** `03-Networking-III-DNS-And-TLS.md` — naam se IP tak (DNS), aur encryption (TLS).