# 03 — SQL Language Basics (From Scratch)

> **Is lesson me:** SQL ka poora practical basics — data padhna (SELECT, WHERE, ORDER BY), likhna/badalna (INSERT, UPDATE, DELETE), tables jodna (JOIN), aur grouping/counting (GROUP BY, aggregates). Har cheez frontend `array` methods se compare karke.

Prerequisites: `02-RDBMS-Tables-Keys-Relationships.md`

---

## 1. SQL kya hai

**SQL (Structured Query Language)** = wo language jisse tum relational database ko instructions dete ho. SQL **declarative** hai — matlab tum batate ho "**kya** chahiye", na ki "**kaise** laao". Database khud figure out karta hai kaise (ye "kaise" wala kaam query planner karta hai — lesson 04).

```sql
-- "Mujhe age > 25 wale users ke naam chahiye"
SELECT name FROM users WHERE age > 25;
```
Tum ye nahi likhte "loop through rows, check each age, collect names" — bas result describe karte ho. Database wo loop khud, optimally chalata hai.

**Frontend analogy:** SQL declarative hai jaise React declarative hai. React me tum batate ho "UI aisa dikhna chahiye" (declarative), na ki "ye DOM node yaha add karo, wo hataao" (imperative). SQL me tum batate ho "result aisa hona chahiye", na ki "aise-aise fetch karo".

SQL commands 4 broad categories me:
- **Query/Read:** `SELECT`
- **Write:** `INSERT`, `UPDATE`, `DELETE`
- **Schema:** `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`
- **Control:** `BEGIN`, `COMMIT` (transactions — lesson 08)

---

## 2. CREATE TABLE — table banana

```sql
CREATE TABLE users (
  id       INTEGER PRIMARY KEY,       -- primary key
  username TEXT NOT NULL UNIQUE,      -- required, no duplicates
  email    TEXT NOT NULL UNIQUE,
  age      INTEGER,                    -- optional (can be null)
  created_at TIMESTAMP DEFAULT now()   -- auto-fills current time
);
```

Har column: naam + data type + optional constraints (`NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `DEFAULT`, `CHECK`, etc.). Ye "fixed schema" hai — columns aur unke types pehle se define.

---

## 3. INSERT — naya data daalna

```sql
INSERT INTO users (username, email, age)
VALUES ('bob_123', 'bob@mail.com', 25);
```
(id auto-generate ho jayega agar auto-increment set ho; created_at default se aa jayega.)

**Frontend analogy:** `users.push({ username: 'bob_123', email: '...', age: 25 })`.

---

## 4. SELECT — data padhna (sabse important)

### Basic
```sql
SELECT * FROM users;              -- saare columns, saari rows
SELECT username, email FROM users; -- sirf ye 2 columns
```
`*` = "all columns". Production me specific columns maangna better hai (less data transfer).

### WHERE — filtering
```sql
SELECT * FROM users WHERE age > 25;
SELECT * FROM users WHERE username = 'bob_123';
SELECT * FROM users WHERE age >= 18 AND age <= 30;   -- multiple conditions
SELECT * FROM users WHERE age > 25 OR username = 'eve_99';
SELECT * FROM users WHERE age BETWEEN 18 AND 30;      -- shorthand for range
SELECT * FROM users WHERE username IN ('bob_123', 'eve_99');  -- multiple values
SELECT * FROM users WHERE email LIKE '%@gmail.com';   -- pattern match (% = wildcard)
```

**Frontend analogy:** `WHERE` = `.filter()`.
```js
users.filter(u => u.age > 25);  // ≈ SELECT * FROM users WHERE age > 25
```

### ORDER BY — sorting
```sql
SELECT * FROM users ORDER BY age;         -- ascending (default)
SELECT * FROM users ORDER BY age DESC;     -- descending
SELECT * FROM users ORDER BY age DESC, username ASC;  -- age se, phir tie pe username se
```
Analogy: `.sort((a,b) => a.age - b.age)`.

### LIMIT / OFFSET — pagination
```sql
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;           -- top 10
SELECT * FROM users ORDER BY created_at DESC LIMIT 10 OFFSET 20; -- skip 20, then 10 (page 3)
```
Analogy: `.slice(20, 30)`. Ye exactly wahi pagination hai jo tum "page 1, 2, 3" ke liye use karte ho.

---

## 5. UPDATE — existing data badalna

```sql
UPDATE users SET age = 26 WHERE id = 1;              -- ek specific row
UPDATE users SET age = age + 1 WHERE age < 18;       -- multiple rows, computed value
```

> ⚠️ **`WHERE` bhoolna = disaster.** `UPDATE users SET age = 0;` (no WHERE) **saari rows** ka age 0 kar dega. Ye ek famous production mistake hai. Hamesha `WHERE` double-check karo.

Analogy: `users.forEach(u => { if (u.id === 1) u.age = 26; })`.

---

## 6. DELETE — data hataana

```sql
DELETE FROM users WHERE id = 3;         -- ek row
DELETE FROM users WHERE age < 13;        -- multiple rows
```
Same warning: `DELETE FROM users;` (no WHERE) = **poori table khali**. Careful.

Analogy: `users = users.filter(u => u.id !== 3)`.

---

## 7. JOIN — tables ko combine karna (the powerful part)

Yaad karo lesson 02 — `posts.user_id` foreign key se `users.id` ko point karta hai. JOIN in dono tables ko **combine** karke ek result banata hai.

```
Table: users              Table: posts
┌────┬──────────┐        ┌─────┬─────────┬───────────┐
│ id │ username │        │ id  │ user_id │ caption   │
├────┼──────────┤        ├─────┼─────────┼───────────┤
│ 1  │ bob_123  │        │ 101 │ 1       │ "Sunset"  │
│ 2  │ alice_x  │        │ 102 │ 2       │ "Coffee"  │
└────┴──────────┘        └─────┴─────────┴───────────┘
```

```sql
-- "Har post ke saath, uske author ka username bhi dikhao"
SELECT posts.caption, users.username
FROM posts
JOIN users ON posts.user_id = users.id;
```
Result:
```
┌───────────┬──────────┐
│ caption   │ username │
├───────────┼──────────┤
│ "Sunset"  │ bob_123  │   ← post 101 + Bob (kyunki user_id=1)
│ "Coffee"  │ alice_x  │   ← post 102 + Alice (kyunki user_id=2)
└───────────┴──────────┘
```

`ON posts.user_id = users.id` = "in dono tables ki rows ko match karo jaha ye condition true ho".

**Frontend analogy:** JOIN = manually do arrays ko merge karna:
```js
posts.map(post => ({
  caption: post.caption,
  username: users.find(u => u.id === post.user_id).username
}));
```
JOIN yehi kaam database ke andar, optimized tarike se (indexes ke saath) karta hai.

### JOIN types (awareness)
```sql
INNER JOIN  -- sirf wo rows jinka dono taraf match ho (default, sabse common)
LEFT JOIN   -- left table ki SAARI rows + jo match ho right se (na match ho toh NULL)
RIGHT JOIN  -- ulta (right ki saari rows)
FULL JOIN   -- dono taraf ki saari rows
```
Example — `LEFT JOIN` se "saare users, chahe unke post ho ya na ho":
```sql
SELECT users.username, posts.caption
FROM users
LEFT JOIN posts ON posts.user_id = users.id;
-- Jis user ka koi post nahi, uske liye caption = NULL aayega (row phir bhi dikhegi)
```

*(JOIN internally kaise chalta hai aur uski complexity — lesson 07 me.)*

---

## 8. GROUP BY + Aggregates — grouping aur counting

**Aggregate functions** poori group of rows pe ek single value compute karte hai: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

```sql
SELECT COUNT(*) FROM users;           -- total kitne users
SELECT AVG(age) FROM users;            -- average age
SELECT MAX(age) FROM users;            -- sabse zyada age
```

`GROUP BY` rows ko groups me baant ke, har group pe aggregate lagata hai:

```sql
-- "Har user ke kitne posts hai?"
SELECT user_id, COUNT(*) AS post_count
FROM posts
GROUP BY user_id;
```
Result:
```
┌─────────┬────────────┐
│ user_id │ post_count │
├─────────┼────────────┤
│ 1       │ 2          │   ← Bob ke 2 posts
│ 2       │ 1          │   ← Alice ka 1 post
└─────────┴────────────┘
```

**Frontend analogy:** `GROUP BY` = lodash ka `groupBy` + phir har group pe count/sum:
```js
_.groupBy(posts, 'user_id')  // { 1: [...2 posts], 2: [...1 post] }
// phir har group ki length nikaalo
```

`HAVING` = `GROUP BY` ke baad groups pe filter (jaise `WHERE`, but groups ke liye):
```sql
-- "Sirf wo users jinke 5+ posts hai"
SELECT user_id, COUNT(*) AS cnt
FROM posts
GROUP BY user_id
HAVING COUNT(*) >= 5;
```

---

## 9. Query ka logical order (important mental model)

Tum SQL is order me likhte ho: `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.

But database ise is **logical order** me evaluate karta hai:
```
1. FROM      → kaunsi table(s), JOINs
2. WHERE     → rows filter karo
3. GROUP BY  → groups banao
4. HAVING    → groups filter karo
5. SELECT    → columns choose karo
6. ORDER BY  → sort karo
7. LIMIT     → top N lo
```
Isliye `WHERE` me tum `SELECT` wale alias use nahi kar sakte (WHERE pehle chalta hai) — ye ek common beginner confusion hai. Bas yaad rakho: **FROM/WHERE pehle, SELECT baad me, LIMIT sabse last.**

---

## Quick Self-Check
1. SQL "declarative" hone ka kya matlab hai — React se analogy do.
2. `WHERE`, `ORDER BY`, `LIMIT` — teeno ke frontend array-method equivalents kya hai?
3. `UPDATE`/`DELETE` me `WHERE` bhoolne se kya hota hai?
4. JOIN kya karta hai? `INNER JOIN` aur `LEFT JOIN` me difference?
5. `GROUP BY` + `COUNT(*)` ka ek example — kya karta hai?
6. `WHERE` aur `HAVING` me difference kya hai?
7. Query ka logical evaluation order kya hai (FROM se LIMIT tak)?

---
**Next:** `04-How-A-Query-Travels.md` — jab tum ye SQL run karte ho, andar exactly kya hota hai.