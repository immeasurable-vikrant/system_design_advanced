# 02 — Percentiles & Tail Latency

> **Is lesson me:** Kyu averages jhooth bolte hai, percentiles (p50/p99/p99.9) ka matlab, tail latency amplification (fan-out killer), aur kyu tum percentiles ko average nahi kar sakte (a classic interview follow-up). Ye performance measurement ka dil hai.

Prerequisites: `01-Latency-Throughput-And-Response-Time.md`

---

## 1. Averages actively mislead ⭐

Latency ke liye average (mean) **misleading** hai — ek 30s timeout hazaaron 5ms requests me chhup jaata hai.
```
Requests (ms): 5, 6, 6, 7, 7, 8, 8, 9, 10, 30000

average = 3,009 ms   ← "our average is 3 seconds!" — nonsense
p50     = 7 ms       ← the TRUTH for typical users
p99     = 30,000 ms  ← the TRUTH for the user who is suffering
```
Average na typical case batata hai (p50 batata), na pain (p99 batata). **Average latency ke liye lagbhag useless hai.**

---

## 2. Percentiles — the real language

| Metric | Meaning | Verdict |
|---|---|---|
| **Average** | Sum ÷ count | Actively misleading (ek timeout hide ho jaata) |
| **p50 / median** | Half faster | Typical case; pain ke baare me kuch nahi |
| **p90** | 1 in 10 slower | Jahan degradation pehli baar visible |
| **p99** | 1 in 100 slower | **The standard SLO target.** 10K rps pe = 100 users/sec suffering |
| **p99.9** | 1 in 1,000 slower | Aksar tumhare **biggest customers** (sabse zyada requests → tail zyada hit) |
| **max** | Worst one | Timeouts spot karne ko useful, SLO ke liye too noisy |

> **Insight:** p99.9 aksar tumhare **largest customers** ko hit karta hai — wo sabse zyada requests karte hai, toh tail ko sabse zyada baar hit karte hai. Isliye "sirf 0.1% users" actually tumhare most valuable users ho sakte hai.

**Frontend analogy:** tumne Web Vitals dekhe hai — LCP/INP "p75" pe report hote hai, average pe nahi. Google bhi jaanta hai average lie karta — wo tail (slow experiences) dekhta hai. Same principle backend pe, but p99/p99.9.

---

## 3. You CANNOT average percentiles ⚠️⭐ (the classic follow-up)

```
Host A's p99 aur Host B's p99 ko average karna ≠ fleet p99.
```
Ye ek **genuinely common interview follow-up** hai, aur sahi answer strong signal.

**Kyu galat:** p99 ek distribution se derived value hai. Do distributions ke derived values ko average karke tum original combined distribution reconstruct nahi kar sakte.

**Sahi tarika — mergeable structures:**
```
Har host ek HISTOGRAM export kare (fixed buckets) — ya HDR histogram / t-digest
→ distributions ko centrally MERGE karo
→ merged distribution se percentile compute karo
```
Histograms merge hote hai; quantiles nahi. (Isliye metrics me "summary" over "histogram" avoid karo — lesson 06.)

> ⚠️ **Percentiles time pe bhi average nahi hote.** 60 one-minute p99s ka mean ≠ us hour ka p99. Same reason.

**Say this:** *"You can't average percentiles. Har host ek histogram (ya t-digest) export kare, distributions ko merge karo, phir merged se percentile nikaalo. Per-instance p99s ko average karna common aur wrong shortcut hai."*

---

## 4. Tail latency amplification ⭐ (the fan-out killer)

Agar ek request N services me parallel fan-out kare aur **sabka wait** kare, **slowest ek** response time decide karta hai.
```
Har service: 99% fast, 1% slow (p99 tail)

N = 1   → P(no slow call) = 0.99        → 1%  requests slow
N = 10  → P = 0.99^10  ≈ 0.904          → ~10% requests slow
N = 100 → P = 0.99^100 ≈ 0.366          → ~63% requests slow (!)
```
**Ek service ka p99 problem, 100-way fan-out pe, user ke liye p37 problem ban jaata hai.** Isliye large-scale systems tail latency pe obsess karte hai, average pe nahi.

> **Recap (Topic 4 lesson 09 — fan-out):** feed fan-out, ya microservices me ek request kai services call kare (Topic 6) — har additional parallel call tail ki probability compound karti hai. Ye woh hidden cost hai.

**Mitigations (kitni baar right answer hai, us order me):**
| Technique | Kaise | Cost |
|---|---|---|
| **Reduce fan-out** | Denormalize taaki ek call answer de | Storage, write complexity |
| **Timeouts + partial results** | 100ms me jo aaya wo return, baaki degrade | Incomplete responses |
| **Hedged requests** | p95 pe ek duplicate doosre replica ko, pehla answer lo | ~5% extra load, big tail win |
| **Tied requests** | Do replicas ko bhejo; jo start kare doosre ko cancel | Thoda complex |
| **Request budgets** | Total deadline har hop tak propagate; jo fit na kare fail fast | Deadline propagation chahiye |
| **Avoid sequential hops** | 5 sequential 20ms hops = 100ms floor before any work | Architectural |

**Say this:** *"Fan-out ke saath main tail amplification worry karunga — 10 parallel calls, har ek 1% slow tail = ~1 in 10 user requests slow. Main ek deadline propagate karunga, 150ms ke baad partial results return karunga, aur read path pe hedged requests consider karunga jahan work idempotent + cheap ho."*

---

## 5. Interview application

```
Q: "Why p99 and not average?"
A: "Averages hide the tail — ek 30s timeout hazaaron 5ms me disappear ho jaata.
    10K rps pe, p99 = 100 users har second bad experience. Aur fan-out ke saath,
    1% per-service tail 10 parallel calls pe 10% user tail ban jaata."

Q: "How do you compute p99 across 50 servers?"
A: "Average nahi kar sakte. Har host histogram (ya t-digest) export kare,
    distributions centrally merge karo, phir percentile compute karo."
```

**Interview line:** *"Main SLOs p99 (ya p99.9 for critical) pe set karunga, average pe kabhi nahi — average tail hide karta hai jahan actual pain hai. Fan-out-heavy paths pe tail amplification ka dhyan (deadline propagation, partial results, hedging). Aur percentiles ko merge karke compute karunga (histograms), average karke nahi."*

---

## Quick Self-Check
1. Average latency ke liye kyu misleading hai (30s timeout example)?
2. p50 vs p99 vs p99.9 — kaun kya batata? p99.9 aksar kise hit karta (big customers)?
3. Percentiles average kyu nahi kar sakte? Sahi tarika (histograms merge)?
4. Tail amplification kya hai — 100-way fan-out pe 1% tail kya ban jaata?
5. Fan-out tail ki 3 mitigations (reduce/partial/hedge)?
6. Hedged requests kya hai aur cost?
7. "50 servers ka p99" — interview answer?

---
**Next:** `03-Littles-Law-And-Queueing.md` — pool sizing + the queueing wall.