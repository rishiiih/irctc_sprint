# FAILURE-CASCADE.md

# SwiftEats World Cup Final Failure Cascade Analysis

## Scenario

- Total users receiving notification: 180,000,000
- Estimated CTR: 8%
- Active users: 14,400,000
- Users opening app in first minute: ~10,000,000
- API calls per user:
  - GET /restaurants
  - GET /restaurant/:id
  - POST /orders

---

## 1. Traffic Simulation Math

Total Requests:

10,000,000 users × 3 API calls

= 30,000,000 requests

Peak Window = 60 seconds

Peak RPS:

30,000,000 / 60

= 500,000 Requests/Second

### Result

Expected Traffic = **500,000 RPS**

Current Capacity = **~12,000 RPS**

Demand exceeds capacity by:

500,000 / 12,000

≈ **41.6×**

---

## 2. Component Capacity Numbers

### PostgreSQL

Configured:

max_connections = 100

Average query time = 20ms

Payment hold time = 800ms

Assume:

- 70% requests are normal queries
- 30% requests involve payment

Formula:

Connections Held =
(0.7 × RPS × 0.02)
+
(0.3 × RPS × 0.8)

Pool Exhaustion:

100 =
(0.014RPS)
+
(0.24RPS)

100 = 0.254RPS

RPS ≈ 394

### Finding

Database exhausts at roughly **394 RPS**

System receives **500,000 RPS**

Collapse is immediate.

---

### Node.js Capacity

Single Process

- 1 CPU
- 4GB RAM

Observed practical limit:

12,000–15,000 RPS

At higher traffic:

- Event loop backlog grows
- Response time spikes
- Heap fills
- OOM crash

---

### Payment Service

Synchronous payment request:

200–2000ms

Average:

800ms

DB connection remains occupied during payment.

Result:

Pool exhaustion occurs far earlier than expected.

---

## 3. Failure Cascade

---

### Failure 1 — PostgreSQL Pool Exhaustion

Severity: CRITICAL

Trigger:

~394 RPS

Observed Behaviour:

- New connections rejected
- Queries timeout
- API returns 500

User Impact:

Cannot browse menus or place orders.

Causes:

Node request queue buildup.

---

### Failure 2 — Node.js Event Loop Saturation

Severity: CRITICAL

Trigger:

~12,000–15,000 RPS

Observed Behaviour:

- Response times rise from 50ms → 5s+
- Requests queue indefinitely

User Impact:

App appears frozen.

Causes:

Memory growth and eventual crash.

---

### Failure 3 — Synchronous Payment Amplification

Severity: HIGH

Trigger:

Large ordering surge

Observed Behaviour:

- Connections held for 800ms+
- Payment gateway latency propagates to DB

User Impact:

Orders stuck in processing.

Causes:

Pool exhaustion accelerates.

---

### Failure 4 — Promo Code Race Condition

Severity: HIGH

Trigger:

Millions redeem simultaneously

Observed Behaviour:

SELECT promo
UPDATE promo

Executed separately.

User Impact:

Promo oversold.

Business Impact:

Revenue loss due to excessive discounts.

---

### Failure 5 — Static Asset Saturation

Severity: HIGH

Trigger:

10M users requesting images

Calculation:

10M users × 20 images × 200KB

≈ 40TB transfer

Observed Behaviour:

Network bandwidth exhausted.

User Impact:

Images fail to load.

Causes:

API becomes unreachable.

---

### Failure 6 — Monitoring Blindness

Severity: MEDIUM

Observed Behaviour:

No tracing.

No correlation IDs.

No clear root cause visibility.

Impact:

MTTD increases to 30–45 minutes.

---

## 4. Incident Timeline

| Time | Event |
|--------|--------|
| T+0s | Push notification sent |
| T+3s | PostgreSQL pool exhausted |
| T+5s | Response times increase sharply |
| T+8s | Connection failures begin |
| T+10s | Payment timeouts occur |
| T+12s | Promo budget oversold |
| T+15s | NIC saturation from image traffic |
| T+18s | Node.js crashes with OOM |
| T+20s | Service fully unavailable |
| T+45m | Root cause identified |
| T+2h | Service restored |

---

## Key Findings

1. PostgreSQL fails first at ~394 RPS.
2. Incoming traffic reaches 500,000 RPS.
3. Demand exceeds Node capacity by 41×.
4. Payment calls amplify database failures.
5. Lack of CDN causes 40TB image traffic to hit origin servers.