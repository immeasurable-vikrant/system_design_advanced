# 06 — Authorization Models (RBAC, ABAC, ReBAC)

> **Is lesson me:** Authorization ko kaise structure karo — RBAC (roles), ABAC (attributes/policies), ReBAC (relationships), scopes, aur least privilege. AuthN (lesson 03-05) ne "kaun ho" solve kiya; ye "kya kar sakte ho" ko scalably model karta hai.

Prerequisites: `01-Security-Fundamentals-AuthN-vs-AuthZ.md`

---

## 1. The problem — permissions at scale

Ek chhote app me: `if user.is_admin: allow`. But grow hote hi ye mess ban jaata hai — 50 features, 10 user types, "editors can edit but not delete", "managers see their team's data only". Ad-hoc `if` checks maintainable nahi. Chahiye ek **model.**

> **Recap (lesson 01):** AuthZ hamesha **server-side** (never trust client). Ye models server-side enforcement ko structure dete hai.

---

## 2. RBAC — Role-Based Access Control ⭐ (most common)

Users ko **roles** assign karo; roles ke paas **permissions** hote hai. User ki permissions = uske roles ki permissions.
```
Roles → Permissions:
  admin  → [read, write, delete, manage_users]
  editor → [read, write]
  viewer → [read]

Users → Roles:
  Alice → admin
  Bob   → editor

Check: "kya Bob delete kar sakta?" → Bob=editor → editor has [read,write] → NO delete
```

**Pros:** simple, intuitive, easy manage (role badlo, saare users update), most apps ke liye kaafi.
**Cons:** "role explosion" — fine-grained needs (per-resource, conditional) pe bahut roles ban jaate hai ("editor_team_A_readonly_weekends"). Context-aware nahi (e.g., "sirf apna data").

**Kab:** zyadatar apps. **Default choice.** Clear role hierarchy ho toh perfect.

**Frontend analogy:** tumne `user.role === 'admin'` se conditionally render kiya hoga. RBAC wahi, but server-side + structured (roles → permissions table, not hardcoded checks).

---

## 3. ABAC — Attribute-Based Access Control

Access **attributes + policies** se decide hota hai — user attributes, resource attributes, aur context (time, location).
```
Policy: "Allow IF user.department == resource.department
         AND user.clearance >= resource.sensitivity
         AND time is business_hours"

Check har request pe policy evaluate kare against attributes.
```
**Pros:** very flexible, fine-grained, context-aware (time/location/relationship), no role explosion.
**Cons:** complex to implement + reason about, harder to audit ("kaun kya access kar sakta" ek query nahi).

**Kab:** complex/regulated domains (healthcare, finance, government), context matters (time-based, location-based, data-classification).

**Example:** "ek doctor sirf apne current patients ke records dekh sake, sirf apni shift ke daoran" — RBAC me mushkil, ABAC me natural (attributes: doctor-patient relationship + shift time).

---

## 4. ReBAC — Relationship-Based Access Control

Access **relationships** se (X ka Y ke saath kya relation hai). Google Zanzibar (jo Google Docs/Drive permissions power karta) ise popular kiya.
```
"Alice CAN edit doc_5 IF Alice is an editor of doc_5"
"Bob CAN view folder_3 IF Bob is a member of a group that owns folder_3"

Relationships graph: user → relation → object
  alice --editor--> doc_5
  doc_5 --parent--> folder_3   (inherited permissions)
```
**Pros:** natural for sharing/hierarchies (docs, folders, orgs), handles inheritance, scales to complex graphs.
**Cons:** complex infra (a permission graph/service — Zanzibar-style).

**Kab:** sharing-heavy apps (Google Docs, GitHub repos, social graphs), nested resources with inherited permissions.

**Frontend analogy:** Google Docs me "share with", "anyone with link", "inherited from folder" — wo ReBAC hai. Permissions relationships (owner/editor/viewer + folder hierarchy) se aati hai.

---

## 5. Comparison + decision

| Model | Based on | Complexity | Best for |
|---|---|---|---|
| **RBAC** | Roles | Low | Most apps (default) |
| **ABAC** | Attributes + policies | High | Context-aware, regulated |
| **ReBAC** | Relationships | High | Sharing, hierarchies (docs) |

```
Clear roles, straightforward?        → RBAC (start here)
Context matters (time/location/data)? → ABAC
Sharing + nested resources?           → ReBAC
```
Real systems **combine** — e.g., RBAC for coarse roles + ABAC for fine conditions ("editor role, but only for own department").

**Interview line:** *"Main RBAC se start karunga — roles → permissions, simple aur most needs cover karta. Agar context matter kare (time-based, data-classification, 'own data only'), ABAC policies add karunga. Sharing-heavy (docs/folders with inherited perms) ke liye ReBAC (Zanzibar-style). AuthZ hamesha server-side, har request pe checked."*

---

## 6. Scopes (OAuth authorization)

OAuth/JWT me, **scopes** = ek token ke permissions (lesson 05).
```
Token scope: "read:profile write:posts"
  → is token se profile read + posts write kar sakte, delete NAHI
API endpoint check kare: "is action ke liye required scope token me hai?"
```
Scopes = token-level coarse authorization (RBAC-ish for API access). Fine-grained AuthZ (resource-level) app logic me.

---

## 7. Least Privilege in practice (recap + apply)

> **Recap (lesson 01):** minimum permissions only.
```
- Default DENY (whitelist, not blacklist) — jo explicitly allowed wahi
- Roles ko minimum permissions do
- Time-bound elevated access (JIT — just-in-time admin, not permanent)
- Regular audits — "kaun kya access kar sakta" review karo
- Separate roles for separate concerns (no god-mode)
```

> ⚠️ **Broken Access Control OWASP #1 hai (lesson 08).** Sabse common serious bug = AuthZ check bhool jaana, ya client pe trust karna. **Har protected action pe, server-side, explicitly check karo.** IDOR (apna ID badalke doosre ka data) classic example.

---

## Quick Self-Check
1. Ad-hoc `if is_admin` checks scale pe kyu fail hote (role explosion → need a model)?
2. RBAC kaise kaam karta (users → roles → permissions)? Kab default?
3. ABAC RBAC se kaise alag (attributes + policies + context)? Doctor example?
4. ReBAC kya hai (relationships)? Google Docs analogy?
5. RBAC vs ABAC vs ReBAC — kab kaunsa?
6. Scopes kya hai (OAuth/JWT token permissions)?
7. Least privilege ke practices? Broken Access Control kyu #1?

---
**Next:** `07-Service-To-Service-Security.md` — services ek dusre ko kaise trust karein.