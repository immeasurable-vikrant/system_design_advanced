# 06 — Consistent Hashing (Distribution + Replication)

> **Is lesson me:** Consistent hashing ka quick recap (Topic 2 me poora dekha), phir naye distributed-systems angles — replication placement (N consecutive nodes), rendezvous hashing comparison, aur implementation. Ye lesson 09 (leaderless replication) ka direct building block hai.

Prerequisites: Topic 2 lesson 11 (consistent hashing basics — ideally padha ho)

---

## 1. Quick recap — the problem & the ring

> **Recap (Topic 2 lesson 11):** naive `hash(key) % N` me server add/remove pe **almost saari keys** remap ho jaati hai (~100% cache miss = stampede). **Consistent hashing** ise fix karta hai — sirf **~K/N keys** move hoti hai.

Ek-line reminder:
```
Ring (0 to 2^32 circular):
  Servers ko hash karke ring pe place karo
  Keys ko hash karke ring pe place karo
  Key → clockwise pehla server jo mile, wahi assign

Server add/remove → sirf ek segment ki keys move (rest untouched)
```

**Virtual nodes:** har physical server ko ring pe **multiple positions** (100-200) pe rakho → even distribution + failure load evenly baant.

```
Mod hashing:        ~N/(N+1) keys move (nearly everything)
Consistent hashing: ~1/N keys move
```

Agar Topic 2 lesson 11 padha hai, ye tumhe pata hai. Ab **naya material.**

---

## 2. NEW: Replication with Consistent Hashing ⭐

Fault tolerance ke liye, har key ko sirf ek node pe nahi — **N consecutive nodes** (clockwise) pe store karte hai.

```
Replication factor = 3
Key at ring position 150:
  Primary:   position 180 pe server (first clockwise)
  Replica 1: position 200 pe server (second clockwise)
  Replica 2: position 250 pe server (third clockwise)
```

**Kyu ye elegant hai:** ring pehle se ordering deta hai. "Agle N nodes" naturally replicas ban jaate hai. Primary die → next clockwise node automatically naya primary (data already uske paas hai).

> **Recap (Topic 2 lesson 12):** replication = same data multiple copies. Yahan consistent hashing + replication combine hote hai — ring pe consecutive nodes = replica set. Ye Dynamo-style systems (lesson 09) ka core hai.

**Ye "preference list" kehlata hai** (Dynamo terminology) — ek key ke liye pasandida nodes ki list, ring order me.

---

## 3. NEW: Uneven distribution & virtual nodes (deeper)

Sirf 3 physical servers ring pe → distribution uneven ho sakta hai:
```
Server A owns: 10 to 300 (290 units) → ~80% traffic!
Server B owns: 300 to 350 (50 units) → ~14%
Server C owns: 350 to 10 (20 units) → ~6%
```
**Virtual nodes fix:** har server 100-200 baar ring pe (`hash("A_1")`, `hash("A_2")`, ...). Ab har server ke chhote segments evenly scattered → load balance.

**Bonus (important):** server remove hone pe, uska load **saare remaining servers me evenly** baant jaata hai (sirf ek neighbor pe dump nahi). Ye virtual nodes ka underrated benefit hai — graceful failure absorption.

---

## 4. Implementation (the TreeMap trick)

Lookup efficient hona chahiye — "clockwise next server" ko O(log N) me dhundo, using a sorted structure (Java `TreeMap`, Python `sortedcontainers`):
```java
class ConsistentHash<T> {
    TreeMap<Integer, T> ring = new TreeMap<>();
    int virtualNodes;

    void addNode(T node) {
        for (int i = 0; i < virtualNodes; i++)
            ring.put(hash(node + "_" + i), node);   // vnodes
    }

    T getNode(String key) {
        int h = hash(key);
        var entry = ring.ceilingEntry(h);   // clockwise next (>= h)
        if (entry == null) entry = ring.firstEntry();  // wrap around
        return entry.getValue();
    }
}
```
`ceilingEntry(h)` = "h ke baraabar ya usse bada pehla key" = **clockwise next server**. Wrap-around ke liye `firstEntry()`. TreeMap red-black tree hai → O(log N) lookup.

> **Hash function choice:** MurmurHash3 ya xxHash (fast, good distribution). **NOT** Java's `hashCode()` (poor distribution, clusters). **NOT** MD5/SHA (too slow — cryptographic security yaha chahiye nahi).

---

## 5. NEW: Consistent Hashing vs Rendezvous Hashing

Ek alternative approach — **Rendezvous (Highest Random Weight) hashing:**
```
Consistent hashing: ring + virtual nodes; key → clockwise next
Rendezvous: har (key, server) pair ko hash karo, HIGHEST weight wala server pick
```
```
Key "user_5" ke liye:
  hash("user_5" + "ServerA") = 0.83
  hash("user_5" + "ServerB") = 0.91  ← highest → ServerB
  hash("user_5" + "ServerC") = 0.45
```

| | Consistent Hashing | Rendezvous |
|---|---|---|
| How | Ring + virtual nodes | Har (key,server) hash, highest pick |
| Add/Remove | ~K/N keys move | ~K/N keys move (same) |
| Lookup | O(log N) (TreeMap) | O(N) (sab servers check) |
| Memory | Ring store karo | Sirf server list |
| Kab | Large N (many servers) | Small N (< 20 servers) |

Dono ~K/N keys move dete hai (dono ka core benefit). Rendezvous simpler (no ring), but O(N) lookup — isliye small clusters ke liye. Consistent hashing O(log N) — large clusters ke liye.

---

## 6. Real-world usage (recap + extend)

| System | Kaise |
|---|---|
| DynamoDB | Partition placement; node add = ek partition split |
| Cassandra | Token ring + virtual nodes (vnodes) |
| Redis Cluster | 16384 hash slots nodes me distributed |
| CDN (Akamai) | Nearest cache node routing (Topic 1) |
| Discord | Guilds ko specific servers pe route |

> **Recap (Topic 1 lesson 05):** load balancers bhi consistent hashing use karte hai session affinity ke liye. Same technique, alag context — LB, cache, DB sharding, sab.

---

## Quick Self-Check
1. Consistent hashing naive `hash % N` se better kyu (K/N vs ~all keys move)?
2. Virtual nodes 2 problems kaise solve karte hai (even distribution + failure spread)?
3. **Replication with consistent hashing** — N consecutive nodes ka idea (preference list)?
4. Primary die hone pe next primary automatically kyu ban jaata hai (ring ordering)?
5. TreeMap ka `ceilingEntry` kya deta hai (clockwise next)? Lookup complexity?
6. Kaunsa hash function use karo aur kyu nahi (MurmurHash vs hashCode vs MD5)?
7. Consistent hashing vs rendezvous — lookup complexity aur kab kaunsa?

---
**Next:** `07-Vector-Clocks.md` — concurrent writes me conflict detect karna.