# 08 — Deployment Strategies

> **Is lesson me:** Deploy vs release (the decoupling insight), deployment strategies (rolling/blue-green/canary/shadow), aur feature flags. Ye "how do you ship without downtime?" ka answer hai — jo deep design questions ka natural end hai.

Prerequisites: `04-Availability-And-Reliability-Math.md`, Topic 6 lesson 09 (migration), Topic 1 lesson 05 (LB)

---

## 1. Deploy vs Release ⭐ (the key decoupling)

```
DEPLOYMENT: naya code production servers pe daalna (running)
RELEASE:    us code ko users ke saamne exposing (traffic)
```
Inhe **decouple** karna modern delivery ka core insight hai. Code deployed but dark (feature flag off) ho sakta hai — production me chal raha, but koi user use nahi kar raha. Phir tum **deploy ke risk ko release ke risk se alag** kar dete ho.
```
Deploy (code out) → verify healthy → Release (flag on, gradually) → verify metrics
  Agar release me problem → flag OFF (instant, no redeploy) — separate lever
```

**Frontend analogy:** tumne feature flags use kiye honge (LaunchDarkly, ya ek simple config flag) — code merged + deployed hai, but feature `if (flag.newCheckout)` ke peeche hidden, gradually rolled out. Ye exactly deploy≠release hai.

---

## 2. Deployment strategies ⭐

### Rolling deployment (the default)
Instances ko batches me replace karo, kuch purane + kuch naye chalte hue.
```
[v1][v1][v1][v1]  → [v2][v1][v1][v1] → [v2][v2][v1][v1] → ... → [v2][v2][v2][v2]
```
**Pros:** no extra infra, gradual. **Cons:** dono versions ek saath live (backward compatibility zaroori! — lesson 09), rollback slow (roll forward/back batch by batch).

### Blue-Green
Do identical environments. Blue (current) live; green (new) prep. Traffic ek saath switch.
```
Blue (v1) ← 100% traffic     Green (v2) idle/testing
  → switch →
Blue (v1) idle (rollback ready)  Green (v2) ← 100% traffic
```
**Pros:** instant switch, instant rollback (traffic wapas blue). **Cons:** 2× infra (briefly), DB migrations tricky (dono environments ek DB share karte — lesson 09).

### Canary ⭐ (the safest for risk)
Naye version ko **chhote % traffic** pe release karo, metrics watch karo, phir gradually badhao.
```
v2 ← 1% traffic  → metrics OK? → 5% → 25% → 50% → 100%
                 → metrics BAD? → 0% (rollback), only 1% users affected
```
**Pros:** real-traffic validation with minimal blast radius, automated rollback on metric regression. **Cons:** needs good metrics + automation, slower.

> **Recap (lesson 04):** canary "correlated failure" (ek bad deploy sab instances pe) ke against defense hai — staggered rollout se ek bad version poore fleet ko ek saath nahi girata.

### Shadow (mirror)
Real traffic ko naye version pe **copy** karo, but response **discard** karo (users ko purana milta).
```
Request → v1 (real response to user)
        ↘ (mirrored copy) → v2 (response discarded, just measured)
```
**Pros:** production traffic se test bina user risk. **Cons:** complex; ⚠️ **side effects** dhyan (v2 duplicate emails na bheje, DB double-write na kare — idempotency/dry-run mode).

### Comparison
| Strategy | Rollback | Infra cost | Blast radius | Best for |
|---|---|---|---|---|
| Rolling | Slow | None | Medium | Default, stateless services |
| Blue-Green | Instant | 2× (brief) | All-or-nothing | Fast rollback critical |
| Canary | Fast (metric-triggered) | Small | Tiny (1%) | Risky changes, high traffic |
| Shadow | N/A (no release) | Extra | Zero (users) | Perf testing new version |

---

## 3. Feature flags ⭐

Code ko runtime behavior se decouple karta hai — deploy≠release ka mechanism.
```python
if feature_flags.enabled("new_checkout", user):   # per-user/%/attribute
    return new_checkout()
return old_checkout()
```
**Uses:**
```
- Gradual rollout (1% → 100%, like canary but at code level)
- Instant kill switch (problem → flag off, no redeploy — seconds not minutes)
- A/B testing (variant per user cohort)
- Trunk-based dev (unfinished features merged but flagged off)
- Ops toggles (disable expensive feature under load — load shedding, lesson 09)
```

> ⚠️ **Flag debt:** flags accumulate → code ek maze of conditionals ban jaata. **Stale flags remove karo** (cleanup tickets on rollout complete). Aur flag state = config (Topic 6 lesson 03) — audited, not hardcoded.

**Say this:** *"Main deploy ko release se decouple karunga — code dark deploy, phir feature flags se gradually release. Risky changes canary pe (1% → watch metrics → ramp, auto-rollback on regression). Flags kill switches bhi dete hai — problem pe seconds me off, redeploy ke bina."*

---

## 4. Rollback — the most important capability

```
Fastest recovery = rollback, not forward-fix.
  → "Can I roll back in one click?" har deploy strategy ka acid test.
  → Blue-green: instant (traffic switch). Canary: instant (0%). Rolling: slower.
  → Feature flag: instant (flag off) even without touching deploy.
```
> **Recap (lesson 04 — MTTR):** rollback speed = MTTR ka biggest lever. Ek rehearsed one-click rollback heroic forward-debugging se 10× value deta. **"How do I undo this fast?" design karo, sirf "how do I ship" nahi.**

---

## Quick Self-Check
1. Deploy vs release — difference aur decouple kyu (dark deploy)?
2. Rolling deployment — pros/cons? Backward compatibility kyu zaroori?
3. Blue-green — instant rollback kaise? DB migration ka issue?
4. Canary kya hai aur kyu safest (blast radius)? Correlated failure se relate?
5. Shadow deployment — kya karta, aur side-effect ka warning?
6. Feature flags ke 3 uses (rollout/kill-switch/A-B)? Flag debt?
7. Rollback MTTR ka biggest lever kyu (lesson 04)?

---
**Next:** `09-Zero-Downtime-Migrations-And-Resilience.md` — schema changes + failure handling.