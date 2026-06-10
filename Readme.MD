# AUDIT.md

# Zudio Incident - Production Audit Report

## Repository Information

* Project: Zudio Incident Backend
* Audit Date: 10 June 2026
* Auditor: Rishiraj Singh
* Environment: Node.js + PostgreSQL
* Branch: zudio-incident

---

# 1. Profiling Results (Before Fixes)

| Endpoint                                  | Response Time | Query Count | Observation                  |
| ----------------------------------------- | ------------- | ----------- | ---------------------------- |
| GET /api/products                         | 320ms         | 1           | Working normally             |
| GET /api/products?search=shirt            | 280ms         | 1           | Search works                 |
| GET /api/products?search=shirt' OR '1'='1 | 290ms         | 1           | SQL Injection vulnerability  |
| POST /api/auth/register                   | 45ms          | 1           | Password stored as plaintext |
| POST /api/auth/login                      | 40ms          | 1           | Login successful             |
| GET /api/orders/history                   | 14200ms       | 201         | Severe N+1 query issue       |
| POST /api/cart/checkout                   | 890ms         | 3           | Discount applied twice       |
| GET /api/products (after checkout)        | 300ms         | 1           | Stock unchanged              |

---

# Bug 1: SQL Injection in Product Search

**Severity:** CRITICAL

**File:** src/controllers/product.controller.js

**Line:** ~34

## Root Cause

User input is directly concatenated into SQL queries using string interpolation.

Example:

```js
const query = `
SELECT * FROM products
WHERE name LIKE '%${req.query.search}%'
`
```

This allows attackers to execute arbitrary SQL.

## Reproduction Steps

1. Open Postman.
2. Send:

GET /api/products?search=shirt' OR '1'='1

3. Observe that all products are returned.

### Expected

Only products matching the search string.

### Actual

Entire table is returned.

## Affected Users / Impact

* All customers
* Entire database exposed
* Possible data manipulation and deletion

## Fix Plan

Replace string concatenation with parameterized queries.

```js
const query = `
SELECT * FROM products
WHERE name ILIKE $1
`

await pool.query(query, [`%${req.query.search}%`])
```

---

# Bug 2: Plaintext Password Storage

**Severity:** CRITICAL

**File:** src/controllers/auth.controller.js

**Line:** ~25

## Root Cause

Passwords are inserted directly into the database without hashing.

Example:

```js
INSERT INTO users(email,password)
VALUES($1,$2)
```

where password is raw user input.

## Reproduction Steps

1. Register a new user.
2. Query database:

```sql
SELECT password FROM users;
```

3. Observe password stored in readable form.

### Expected

Hashed password.

### Actual

Plaintext password.

## Affected Users / Impact

* Complete credential compromise if DB leaks.
* Users may reuse passwords elsewhere.

## Fix Plan

Use bcrypt with 12 salt rounds.

```js
const hashedPassword = await bcrypt.hash(password,12)
```

Store hash instead of plaintext.

---

# Bug 3: Double Discount Application

**Severity:** HIGH

**File:** src/controllers/checkout.controller.js

**Line:** ~82

## Root Cause

Coupon validation and coupon usage update occur in separate operations.

Two simultaneous requests can apply the same coupon.

## Reproduction Steps

1. Checkout using coupon SAVE50.
2. Repeat checkout immediately.
3. Coupon is applied again.

### Expected

Coupon usable only once.

### Actual

Discount applied multiple times.

## Affected Users / Impact

* Revenue loss
* Negative order totals possible

## Fix Plan

Atomic update:

```sql
UPDATE coupons
SET used = true
WHERE code = $1
AND used = false
RETURNING *
```

If no row returned → coupon already used.

---

# Bug 4: Stock Not Decrementing After Purchase

**Severity:** HIGH

**File:** src/controllers/checkout.controller.js

**Line:** ~120

## Root Cause

Order creation succeeds but inventory update never executes.

Stock remains unchanged after purchases.

## Reproduction Steps

1. Check product stock.
2. Purchase product.
3. Check stock again.

### Expected

Stock decreases.

### Actual

Stock remains identical.

## Affected Users / Impact

* Overselling inventory
* Incorrect stock visibility
* Customer dissatisfaction

## Fix Plan

Use transaction:

```sql
BEGIN;

INSERT INTO orders ...

UPDATE products
SET stock = stock - $1
WHERE id = $2
AND stock >= $1;

COMMIT;
```

Rollback on failure.

---

# Bug 5: N+1 Query in Order History

**Severity:** MEDIUM

**File:** src/controllers/order.controller.js

**Line:** ~60

## Root Cause

Application loads orders first and then executes one additional query per order item.

Example:

```js
for (const order of orders) {
  await pool.query(...)
}
```

Creates hundreds of DB round trips.

## Reproduction Steps

1. Login as user with many orders.
2. Open order history.
3. Observe response delay.

### Expected

Single optimized query.

### Actual

201 database queries executed.

## Affected Users / Impact

* Extremely slow page load.
* Poor user experience.

## Fix Plan

Replace multiple queries with JOIN.

```sql
SELECT
o.id,
o.total,
o.status,
oi.quantity,
p.name
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.user_id = $1
```

---

# Verification Table

| Bug                 | Before                | After                              | Verification Method                       |
| ------------------- | --------------------- | ---------------------------------- | ----------------------------------------- |
| SQL Injection       | Returns all products  | Returns 0 results / literal search | GET /api/products?search=shirt' OR '1'='1 |
| Plaintext Passwords | Raw password visible  | bcrypt hash stored                 | SELECT password FROM users                |
| Double Discount     | Coupon reused         | Second attempt returns 400         | Checkout twice                            |
| Stock Decrement     | Stock unchanged       | Stock reduced correctly            | Before vs After inventory check           |
| N+1 Query           | 14200ms / 201 queries | 180ms / 2 queries                  | Profiling middleware                      |

---

# Summary

## Security Fixes

* SQL Injection eliminated using parameterized queries.
* Password storage secured with bcrypt hashing.

## Logic Fixes

* Coupon usage made atomic.
* Inventory updates moved into database transactions.

## Performance Fixes

* N+1 query pattern removed.
* Order history optimized with JOIN queries.

## Final Result

All five critical production issues were identified, documented, fixed, and verified successfully. The application now demonstrates improved security, data integrity, and performance.
