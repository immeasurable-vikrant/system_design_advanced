# 04 — Availability & Reliability Math

> **Is lesson me:** The "nines" (99.9% = kitna downtime), kyu serial dependencies availability multiply karti hai, redundancy kaise nines add karti hai (aur "independent" ka catch), SLI/SLO/SLA/error budget, aur failure/recovery metrics (MTTR, RPO, RTO). Ye reliability ko quantitative banata hai.

Prerequisites: `03-Littles-Law-And-Queueing.md`, Topic 5 (availability/CAP), Topic 2 lesson 12 (replication)

---

## 1. The nines ⭐

"High availability" vague hai — nines me socho:

| Availability | Downtime/year | Downtime/month | Downtime/week |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% (three nines) | 8.77 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5.04 min |
| 99.99% (four nines) | 52.6 min | 4.38 min | 1.01 min |
| 99.999% (five nines) | 5.26 min | 26.3 sec | 6.05 sec |

> ⚠️ **Five nines = less than one bad deploy per year.** Tum ise human-in-recovery-path ke saath reach nahi kar sakte — ise automated failover chahiye jo ek page se fast ho. Interview me five nines claim karo toh justify karne ko ready raho.

> **Recap (Topic 7 lesson 02):** har extra "nine" exponentially mehenga (more redundancy, regions, ops). Poocho "kitni nines ACTUALLY chahiye?" — banking = five; internal tool = 99% theek.

---

## 2. Serial dependencies MULTIPLY ⚠️⭐

Ye deep synchronous call chains ke against strongest technical argument hai:
```
Chain: gateway → auth → orders → inventory → payments
Har ek 99.9% pe:
  0.999^5 = 0.995  →  99.5%  →  1.8 DAYS downtime/year!
```
Paanch "very reliable" (99.9%) services series me = ek **mediocre** system (99.5%). Har additional synchronous hop availability ko multiply-down karta hai.

> **Recap (Topic 6 lesson 01, Topic 4):** ye microservices ka hidden cost hai — aur async (queues) + fewer hops ka argument. Critical path chhota rakho; non-critical kaam async karo.

**Say this:** *"Main read path do hops tak rakhunga — paanch 99.9% services series me 99.5% hai, saal me deDh din downtime. Critical path se bahar sab kuch async via queue."*

---

## 3. Redundancy ADDS nines (if independent) ⭐

```
Do independent instances, har ek 99% pe:
  P(both down) = 0.01 × 0.01 = 0.0001  →  availability 99.99%
```
Redundancy availability badhata hai — **but sirf agar failures independent ho.**

> ⚠️ **"Independent" is doing enormous work.** Same AZ, same deploy, same config push, same cert expiry, same poisoned cache entry, same dependency — **correlated failure** benefit kha jaata hai.

**Say this:** *"Redundancy sirf utni nines add karta hai jitni failures actually independent hai — isliye main AZs ke across spread karunga aur deploys stagger karunga."*

> **Recap (Topic 5, Topic 2 lesson 12):** replication + multi-AZ. But correlated failures (ek bad deploy sab replicas pe) real hai — isliye staggered rollout (lesson 08 canary).

---

## 4. SLI vs SLO vs SLA vs Error budget ⭐

| Term | Definition | Example |
|---|---|---|
| **SLI** (Indicator) | The measurement | "% requests < 300ms, measured at LB" |
| **SLO** (Objective) | Internal target | "99.9% requests < 300ms over 30 days" |
| **SLA** (Agreement) | External contract with penalties | "99.5% or 10% credit" |
| **Error budget** | 100% − SLO — allowed failure | 0.1% of 30 days = 43 minutes |

**Three rules:**
```
1. SLA hamesha SLO se LOOSER. Money at stake hone se pehle internal headroom chahiye.
2. Error budget ek BUDGET hai — spend it. Untouched budget = over-investing in
   reliability, under-investing in features. Exhausted = freeze deploys till recovery.
3. Good SLI USER's experience measure kare, server's comfort nahi.
   "CPU < 80%" ek SLI NAHI hai.
```

**Say this:** *"SLO main user-facing indicator pe define karunga — p99 latency + success rate at the edge — aur error budget se release velocity govern karunga. Isse reliability conversation quantitative rehti hai, 'site feels slow' ke bajaye."*

---

## 5. Failure & recovery metrics

| Metric | Meaning |
|---|---|
| **MTBF** | Mean Time Between Failures (repairable systems) |
| **MTTF** | Mean Time To Failure (non-repairable — a disk) |
| **MTTD** | Mean Time To Detect — usually biggest slice, cheapest to improve |
| **MTTA** | Mean Time To Acknowledge (paging/on-call health) |
| **MTTR** | Mean Time To Repair/Recover |

```
Availability = MTBF / (MTBF + MTTR)
```

> ⭐ **The key insight:** MTTR aadha karna availability ko utna hi improve karta hai jitna MTBF double karna — aur usually **10× sasta** hai. Better alerting, faster rollback, rehearsed runbooks > heroic engineering to prevent all failure. **Recovery pe invest karo, sirf prevention pe nahi.**

### RPO vs RTO ⭐
| | RPO | RTO |
|---|---|---|
| Question | Kitna data lose kar sakta hu? | Kitni der down reh sakta hu? |
| Set by | Backup/replication frequency | Recovery process speed |
| RPO = 0 | Synchronous replication (latency cost) | — |
| RTO = 0 | — | Active-active with auto failover |

> **Recap (Topic 2 lesson 12, Topic 5):** sync replication = RPO ~0 but latency cost; async = fast but some data loss window. RPO/RTO decide karte hai replication mode + DR strategy (lesson 09).

---

## 6. Interview application

```
Q: "What's the difference between an SLO and an SLA?"
A: "SLO internal target, SLA contractual promise with penalties. SLO hamesha
    stricter — problem SLA credit se pehle pata chal jaata."

Q: "What's your RPO and RTO?"
A: "RPO = kitna data lose (replication mode se — async cross-region = seconds).
    RTO = kitni der down (promotion automation se — scripted + fencing = minutes).
    Aur dono sirf real hai agar hum drill karein."
```

**Interview line:** *"Availability ko nines me quantify karunga aur serial-dependency math check karunga (5 hops × 99.9% = 99.5%). Redundancy across AZs (independent failures). SLO user-facing indicator pe, error budget se release velocity govern. MTTR pe invest (recovery > prevention — 10× cheaper). RPO/RTO explicitly define + drill karunga."*

---

## Quick Self-Check
1. 99.9% vs 99.99% — downtime/year? Five nines ka catch?
2. Serial dependencies availability ko kaise affect (5 × 99.9% = ?)? Argument kya?
3. Redundancy nines kaise add karta? "Independent" ka catch?
4. SLI vs SLO vs SLA — teeno? Error budget kya, aur "spend it" kyu?
5. Ek good SLI kis cheez ko measure kare (user, not server)?
6. MTTR aadha karna vs MTBF double — kaunsa sasta aur kyu?
7. RPO vs RTO — kaunsa "data loss", kaunsa "downtime"? Replication se relation?

---
**Next:** `05-Observability-Metrics-Logs-Traces.md` — system healthy hai ye kaise pata.