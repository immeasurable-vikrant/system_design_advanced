# 09 — Migration & Evolution Patterns

> **Is lesson me:** Ek existing monolith ko microservices me (ya koi legacy system ko naye me) **safely, incrementally** kaise migrate karo bina "big bang rewrite" ke risk ke — strangler fig pattern, anti-corruption layer, aur decomposition strategy. Ye real-world architecture evolution ka pattern hai (systems greenfield nahi hote).

Prerequisites: `01-Monolith-vs-Microservices.md`, `08-Saga-Pattern.md`

---

## 1. The problem — big bang rewrite = disaster

Ek working monolith ko microservices me le jaana hai. Tempting approach: sab kuch scratch se microservices me rewrite karo, phir ek din switch. Ye **almost hamesha fail** hota hai:
```
Big Bang Rewrite:
  - Mahino/saal naya system banao (purana bhi maintain karte hue)
  - Feature parity mushkil (purana system me hidden edge cases)
  - "Switch day" pe sab kuch ek saath toot sakta hai
  - Business is beech naye features nahi de paata
  → High risk, often abandoned midway
```

**Golden rule:** *incrementally migrate, never big-bang rewrite.* Ek-ek piece nikaalo, verify karo, aage badho.

---

## 2. Strangler Fig Pattern ⭐

Naam ek strangler fig vine se aaya — jo ek ped ke around badhti hai, dheere-dheere use replace karti hai, jab tak original ped na ho aur vine khud khadi rahe.

**Strangler Fig pattern** = naye system ko purane ke around incrementally banao, ek-ek functionality nikaal ke, jab tak purana "strangled" (fully replaced) na ho jaye.

```
Step 1: Facade (proxy/gateway) purane monolith ke saamne rakho
  Client → Facade → Monolith (sab requests abhi monolith ko)

Step 2: Ek feature nikaalo (e.g. "notifications") naye service me
  Client → Facade → /notifications → New Notification Service ✅
                  → everything else → Monolith

Step 3: Aur features nikaalte jao, ek-ek
  Client → Facade → /notifications → Notification Service
                  → /payments     → Payment Service
                  → /orders       → Order Service
                  → rest          → Monolith (shrinking)

Step N: Monolith khali ho gaya → retire
  Client → Facade → saare microservices (monolith gone)
```

> **Recap (Topic 1 lesson 06, 08):** ye facade ek **reverse proxy / API gateway** hai (Topic 1). Wo routing decide karta hai — kaunsa path naye service ko, kaunsa purane monolith ko. Client ko pata bhi nahi chalta migration ho raha hai.

**Pros:** low risk (ek-ek piece, verify karke), no downtime, business chalti rehti hai, rollback easy (ek route wapas monolith pe point karo), incremental learning.

**Frontend analogy:** ye ek legacy jQuery app ko React me incrementally migrate karne jaisa hai — ek-ek component/route ko React me convert karo, baaki jQuery chalta rahe, jab tak poora React na ho. Tumne shayad ye kiya ho. Same idea, backend services pe.

---

## 3. Decomposition strategy — kaunsa piece pehle nikaalo

Sab pieces equal nahi. Smart order:
```
1. Pehle nikaalo: LEAF services (kam dependencies)
   - Notifications, email, reporting (standalone, low risk)
2. High-value: alag scaling ya team ownership wale
   - Payment (compliance), search (scaling)
3. Sabse baad: CORE, tangled pieces (highest risk)
   - Order/user core (sab pe depend, careful extraction)
```

**Criteria for what to extract:**
- **Low coupling** — kam dependencies = easy extraction
- **High value** — alag scaling/deployment need = worth the effort
- **Clear boundary** — well-defined bounded context (lesson 01)
- **Data separability** — uska data cleanly alag ho sakta hai

⚠️ **Data decomposition sabse mushkil hai.** Ek shared DB (monolith) ko per-service DBs me toDna — foreign keys, JOINs, transactions sab break hote hai. Approach: pehle code separate karo (shared DB pe), phir data separate karo (dedicated DB + events/saga for cross-service consistency — lesson 08).

---

## 4. Anti-Corruption Layer (ACL)

Jab naya service purane monolith (ya legacy system) se baat kare, purane system ka "ganda" model naye clean model me leak ho sakta hai. **Anti-Corruption Layer** = ek translation layer jo dono ke beech baithta hai aur models ko translate karta hai, taaki legacy design naye service ko "corrupt" na kare.

```
New Service (clean model) ←→ [Anti-Corruption Layer] ←→ Legacy Monolith (messy model)
                                (translate/adapt)
```
ACL legacy ke weird data formats, naming, structures ko naye service ke clean domain model me convert karta hai. Naya service legacy ki mess se **insulated** rehta hai.

**Frontend analogy:** ye ek adapter/mapper layer jaisa hai jo ek messy third-party API response ko tumhare clean internal types me convert karta hai — taaki tumhara app us API ke weird shape se coupled na ho. ACL wahi services ke beech.

---

## 5. Migration best practices

```
1. INCREMENTAL — ek piece at a time, never big bang
2. FACADE FIRST — proxy/gateway saamne, taaki routing control ho (strangler)
3. LEAF FIRST — low-coupling services pehle (low risk se seekho)
4. VERIFY EACH STEP — har extraction ke baad monitor, rollback ready
5. DUAL-RUN — kabhi purana + naya dono chalao, results compare (parallel run), phir switch
6. DATA LAST — code pehle separate, data (hardest) baad me
7. ACL — legacy se insulate naye services ko
8. EVENTS for consistency — cross-service data sync events/saga se (lesson 05, 08)
```

> **"Dual run" (parallel run):** ek risky migration me, naya aur purana dono ko same input do, outputs compare karo (bina naye ka result use kiye). Jab confident ho ki naya sahi hai, switch. Ye financial/critical systems me common.

**Interview line:** *"Main monolith ko strangler fig se migrate karunga — ek facade/gateway saamne, phir leaf services (notifications, search) pehle nikaalunga, verify karke aage. Data ko last me separate karunga (hardest — shared DB toDna), cross-service consistency events/saga se. Anti-corruption layer se naye services ko legacy mess se insulate. Big-bang rewrite kabhi nahi — incremental, reversible, verified har step pe."*

---

## Quick Self-Check
1. Big bang rewrite kyu fail hota hai (3 reasons)?
2. Strangler fig pattern kya hai — vine analogy se? Facade ka role (Topic 1 recap)?
3. Strangler fig me rollback easy kyu?
4. Decomposition me kaunsa piece pehle nikaalo (leaf/low-coupling) aur kyu?
5. Data decomposition sabse mushkil kyu? Approach kya (code first, data last)?
6. Anti-corruption layer kya karta hai (legacy se insulate)? Frontend adapter analogy?
7. "Dual run" kya hai aur kab (critical migrations)?

---
**Next:** `10-Architecture-Playbook.md` — sab patterns tie together, full curriculum synthesis.