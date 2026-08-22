# 03 — Scaling Real-Time (WebSockets at Scale)

> **Is lesson me:** Real-time systems ka sabse mushkil challenge — WebSockets ko millions of connections tak kaise scale karo. Pub/sub backplane, connection registry, sticky sessions, heartbeats + reconnection, aur memory/capacity math. Ye Topic 1 (LB, consistent hashing) + Topic 3 (Redis pub/sub) ko real-time ke liye combine karta hai.

Prerequisites: `02-Real-Time-Communication.md`, Topic 1 lesson 05 (load balancing)

---

## 1. The core problem — connections are server state

Regular HTTP stateless hai (Topic 1 lesson 04) — koi bhi server koi bhi request handle kar sakta hai. But ek WebSocket **ek specific server pe live rehta hai**. Ye problem create karta hai:

```
Server A ke paas User 1 ki WebSocket hai
Server B ke paas User 2 ki WebSocket hai

User 1 → User 2 ko message bhejta hai
Server A ko mila... but User 2 to Server B pe hai!

Server A, Server B tak kaise forward kare??
```

Ye hi real-time scaling ka dil hai: **connections distributed hai across servers, but messages kisi bhi user ko pahunchne chahiye.**

---

## 2. Solution — Pub/Sub backplane ⭐

Servers ke beech ek **message bus** (Redis Pub/Sub, ya Kafka/NATS) daalo. Har server us bus pe apne connected users ke channels **subscribe** karta hai.

```
        ┌──────────── Redis Pub/Sub (message bus) ────────────┐
        │                                                      │
   Server A                Server B                Server C
   Users: 1,3,5            Users: 2,4,6            Users: 7,8,9
```

**Flow — User 1 (Server A) → User 2 (Server B):**
```
1. User 1 apne WebSocket (Server A) pe message bhejta hai
2. Server A Redis channel "user:2" pe PUBLISH karta hai
3. Server B (jo "user:2" pe SUBSCRIBED hai) message receive karta hai
4. Server B User 2 ki WebSocket pe push karta hai
```

**Frontend analogy:** ye bilkul ek event bus / pub-sub pattern hai jo tumne frontend state management me dekha hoga (`EventEmitter`, Redux middleware). Har server ek subscriber, message bus central dispatcher. Server ko doosre server ka address jaanne ki zarurat nahi — bas channel pe publish/subscribe.

> **Recap (Topic 3):** Redis Pub/Sub — ek `PUBLISH channel message` sab subscribers tak pahunchta hai. Yahi mechanism cross-server message delivery deta hai. (Detail pub-sub pattern — lesson 04.)

---

## 3. Connection routing — 2 approaches

Message sahi server tak kaise pahunche? Do tarike:

### (a) Sticky Sessions (Load Balancer level)
Ek baar WebSocket establish ho jaye, us user ka **saara traffic same server pe** route karo.
```
LB IP hash / cookie-based routing se same user → same server
```
> **Recap (Topic 1 lesson 05):** IP hash / consistent hashing se session affinity. WebSocket me ye zaroori ho jaata hai kyunki connection ek server pe live hai.
**Problem:** uneven load agar kuch users bahut "chatty" ho (kuch servers overloaded).

### (b) Connection Registry (Redis)
Ek mapping store karo: `userId → serverId`.
```
Redis:  user:1 → serverA,  user:2 → serverB,  ...
Message aaye → registry me dekho user kis server pe → us server ko route
```
Pub/sub ke saath combine hota hai — publish channel se sahi server subscribe karta hai.

**Real systems dono use karte hai:** sticky sessions connection ko ek server pe rakhne ke liye + pub/sub (ya registry) cross-server delivery ke liye.

---

## 4. Heartbeats & reconnection

Ek WebSocket "chup" ho sakti hai bina proper close ke (network drop, phone sleep). Server ko kaise pata connection abhi zinda hai? **Heartbeats (ping/pong):**
```
Client                              Server
  │◀── PING ───────────────────────│  Har 30s
  │──▶ PONG ───────────────────────│
  │◀── PING ───────────────────────│
  │    (10s me PONG nahi aaya)     │
  │    Connection DEAD maana        │
  │    Resources clean up           │
```
Agar PONG time pe na aaye → connection dead maano, cleanup karo (warna "ghost" connections memory khaate rahenge).

**Reconnection (client-side):**
- Connection drop → **exponential backoff ke saath auto-reconnect** (lesson 07 me backoff detail)
- Reconnect pe: client apna **last received message ID** bhejta hai → server missed messages **replay** karta hai (Redis sorted set ya DB short-term buffer se)
- **At-least-once delivery + client-side dedup** (lesson 05 idempotency) — taaki koi message miss ya double na ho

**Frontend analogy:** tumne shayadd socket.io ya ek reconnecting-websocket wrapper use kiya ho — wo exactly ye karta hai (auto-reconnect + backoff + missed-message recovery).

---

## 5. Capacity math (interview me impress karta hai)

Long-lived connections **memory** khaate hai — ye tumhare server count ko decide karta hai:
```
Per WebSocket connection: ~10-50KB (kernel + app memory)
Ek server (properly tuned): ~50,000-100,000 connections handle kar sakta

1 million concurrent connections chahiye?
  → 1M / 100K per server = 10-20 servers
  → Memory: 1M × 30KB = ~30GB across fleet
```

**Interview line:** *"Har server ~50-100K WebSocket connections handle kar sakta hai proper tuning ke saath. 1M ke liye 10-20 servers. Consistent hashing se users distribute karo, Redis Pub/Sub ya Kafka cross-server communication ke liye. Connection count per server monitor karke auto-scale karo. Regional deployment se latency kam."*

> **Recap (Topic 1 lesson 04):** ye woh "stateless nahi hai" wali baat hai — WebSocket state hai, isliye ye extra planning (registry, pub/sub, memory budget) chahiye jo regular stateless HTTP me nahi chahiye.

---

## 6. Full scaling architecture

```
                    Clients (millions)
                          │
                  ┌───────▼───────┐
                  │  Load Balancer │  (sticky sessions / IP hash)
                  └───────┬───────┘
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   WS Server A       WS Server B       WS Server C
   (100K conns)      (100K conns)      (100K conns)
        │                 │                 │
        └────────┬────────┴────────┬────────┘
                 ▼                 ▼
          Redis Pub/Sub      Connection Registry
          (message bus)      (userId → serverId)
                 │
                 ▼
          Message DB (persistence, replay buffer)
```

---

## Quick Self-Check
1. WebSocket scaling ka core problem kya (connection = server state)?
2. Pub/sub backplane cross-server message delivery kaise deta hai (User 1→User 2 flow)?
3. Sticky sessions vs connection registry — dono ka role?
4. Heartbeat (ping/pong) kyu zaroori? PONG na aaye toh?
5. Reconnection pe missed messages kaise recover hote hai (last message ID)?
6. 1M WebSocket connections ke liye kitne servers + kitni memory (math)?
7. Ye Topic 1 ke "stateless" concept se kaise differ karta hai?

---
**Next:** `04-Message-Queues.md` — ab service-to-service async messaging.