# 02 — Real-Time Communication (Polling → SSE → WebSocket)

> **Is lesson me:** Client ko server se **live updates** bhejne ke 4 tarike — short polling, long polling, SSE, WebSocket — har ek ka mechanism (sequence diagrams ke saath), comparison, kab kaunsa, aur kab WebSocket NAHI. Topic 1 lesson 02 me intro tha; ye full depth hai.

Prerequisites: Topic 1 lesson 01-02 (HTTP, TCP, real-time intro)

---

## 1. Real-time communication kya hai

**Real-time communication** = server se client ko data **jaise hi available ho** pahunchana, bina client ke explicitly maange.

**Analogy — package delivery ka wait:**
- **Polling:** har 5 min me khud mailbox check karne jaate ho. Zyadatar khaali trips.
- **Long polling:** mailbox pe khaDe ho jaate ho jab tak mailman na aaye, phir andar. Repeat.
- **SSE:** ek doorbell laga li jo bajti hai jab package aaye (one-way notification).
- **WebSocket:** delivery driver se seedhi phone line — wo live updates deta hai, aur tum wapas sawaal bhi puch sakte ho.

Problem ye hai ki HTTP by default **client-pull** hai (client maange tab milta hai). Live updates ke liye humein kuch aur chahiye.

---

## 2. Short Polling (regular polling)

Client fixed intervals pe baar-baar server se poochta hai.
```
Client                              Server
  │── GET /messages?since=100 ─────▶│
  │◀── 200 OK (no new messages) ───│  ← WASTED request
  │    ... 5 sec wait ...           │
  │── GET /messages?since=100 ─────▶│
  │◀── 200 OK (no new) ────────────│  ← WASTED request
  │    ... 5 sec wait ...           │
  │── GET /messages?since=100 ─────▶│
  │◀── 200 OK [{msg:"hello"}] ─────│  ← Finally data!
```
**Problem:** zyadatar requests khaali return hoti hai — bandwidth + server resources waste. Delay = up to polling interval (5s tak purana).

**Frontend analogy:** `setInterval(() => fetch('/messages'), 5000)`. Kaam karta hai, simplest, but wasteful.

---

## 3. Long Polling

Client request bhejta hai, server usse **khula rakhta hai** jab tak data na aaye (ya timeout).
```
Client                              Server
  │── GET /messages?since=100 ─────▶│
  │         ... waiting ...         │  Server connection khula rakhta (30s timeout)
  │                                 │  Naya message aaya!
  │◀── 200 OK [{msg:"hello"}] ─────│
  │── GET /messages?since=101 ─────▶│  ← turant reconnect
  │         ... waiting ...         │
```
**Polling se better:** koi wasted request nahi, data available hote hi aata hai. **But** har response ke baad nayi connection.

---

## 4. Server-Sent Events (SSE)

Server ek **single long-lived HTTP connection** pe data **push** karta rehta hai. **One-way only** (server → client).
```
Client                              Server
  │── GET /events                   │
  │   (Accept: text/event-stream) ─▶│
  │◀── data: {"msg":"hello"} ──────│  Push 1
  │◀── data: {"msg":"world"} ──────│  Push 2
  │◀── data: {"typing":true} ──────│  Push 3
  │    (connection khula rehta hai) │
```
```js
// Client-side (browser) — built-in API
const evtSource = new EventSource('/events/notifications');
evtSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  showNotification(data);
};
// Connection drop ho toh AUTO-reconnect! (built-in)
```
**Key features:** browser me **built-in auto-reconnect** (with `Last-Event-ID`), standard HTTP (proxies/CDNs se kaam karta hai — Topic 1), one-way only, text only.

---

## 5. WebSocket

**Full-duplex, bidirectional** communication ek single TCP connection pe. Dono taraf data bhej sakte hai.
```
Client                              Server
  │── HTTP Upgrade: websocket ─────▶│  Handshake
  │◀── 101 Switching Protocols ────│
  │═══ WebSocket Connection ═════════  (persistent TCP)
  │──▶ {"type":"send_msg","text":"hi"}    Client → Server
  │◀── {"type":"new_msg","from":"Bob"}    Server → Client
  │◀── {"type":"typing","user":"Bob"}     Server → Client
  │──▶ {"type":"read_receipt","id":"x"}   Client → Server
```
HTTP se "upgrade" hoke ek persistent `ws://` connection banti hai jispe dono taraf messages behte hai. Chat, collaborative editing, games ke liye.

> **Recap (Topic 1 lesson 02):** WebSocket HTTP se upgrade hoke bidirectional persistent connection banata hai. Yahan hum uska full flow + scaling (lesson 03) dekh rahe hai.

---

## 6. Comparison table (the decision core)

| Feature | Short Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| Direction | Client→Server | Client→Server | **Server→Client** | **Bidirectional** |
| Latency | Up to interval | Near real-time | Real-time | Real-time |
| Connection | New each poll | New each response | Single long-lived | Single long-lived |
| HTTP compatible | Yes | Yes | Yes | No (upgrades) |
| Proxies/CDN se | Yes | Sometimes issues | **Yes** | Sometimes issues |
| Auto-reconnect | N/A | Manual | **Built-in** | Manual |
| Scalability | Easy | Medium | Easy | **Hard** (lesson 03) |
| Binary data | No | No | No (text only) | **Yes** |
| Max connections | N/A | Limited | ~6 per domain | ~65K per server |

---

## 7. Kab kya use karo

| Use case | Best choice | Kyu |
|---|---|---|
| Chat application | **WebSocket** | Bidirectional, low latency, typing indicators |
| Live sports scores | **SSE** | Server push only, auto-reconnect, simple |
| Stock ticker | WebSocket | High-frequency, bidirectional (subscribe/unsubscribe) |
| Email inbox | Long Polling | Infrequent updates, simpler infra |
| Social feed refresh | Short Polling | Low frequency, simple, cacheable |
| Ride tracking (Uber) | WebSocket / SSE | Continuous location updates |
| Notifications | **SSE** | One-way server push, reliable |
| Online gaming | WebSocket | Low latency, bidirectional, binary |
| Dashboard metrics | SSE | Server push, auto-reconnect |

**Real examples:** Slack (WebSocket — messaging, presence), Uber (WebSocket + gRPC streaming — location), Discord (WebSocket — voice+text), Robinhood (WebSocket — stock prices), Twitter (SSE streaming API), Facebook (Long Polling / MQTT internally).

---

## 8. Kab WebSocket NAHI (important — over-engineering rok)

WebSocket sexy lagta hai but complexity laata hai (connection management, scaling, sticky sessions — lesson 03). **Use mat karo jab:**
- Infrequent updates (once per minute) → polling ya SSE simpler
- **One-way server push only** → SSE lighter (WebSocket ki bidirectional zaroorat nahi)
- Restrictive corporate proxies ke peeche → WebSocket upgrade aksar block hota hai
- HTTP caching matter karti ho → WebSocket responses cache nahi hote
- Simple CRUD → REST kaafi hai
- Sticky sessions afford nahi kar sakte → WebSocket scaling complex

**Interview line:** *"Agar updates sirf server→client behte hai, SSE — simpler, CDNs/proxies se kaam karta hai, auto-reconnect. WebSocket sirf tab jab client bhi high rate pe messages bheje (chat/collab/games). Polling suffice kare toh WebSocket over-engineering hai."*

**Golden rule:** *SSE for one-way push, WebSocket for bidirectional, polling for infrequent/simple. Simplest jo kaam kare, wahi use karo.*

---

## Quick Self-Check
1. 4 approaches (polling → WebSocket) ko package-delivery analogy se samjhao.
2. Short polling ka core problem kya (wasted requests)?
3. Long polling short polling se kaise better hai?
4. SSE ka killer feature kya (auto-reconnect + HTTP-compatible)? Direction?
5. WebSocket SSE se kaise alag (duplex)? HTTP upgrade kya hai?
6. Chat ke liye WebSocket, notifications ke liye SSE — kyu alag?
7. Kab WebSocket NAHI use karna chahiye (3 cases)?

---
**Next:** `03-Scaling-Real-Time.md` — WebSockets ko millions tak scale karna (the hard part).