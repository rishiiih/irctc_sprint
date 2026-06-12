# CONCURRENCY.md

# ShowTime Ticketing System - Concurrency Strategy

## Objective

Prevent double-booking of seats during high-demand events while supporting:

- 500,000 concurrent users
- Response time < 500ms
- AWS budget ≤ $2,000/month

---

# Option A: PostgreSQL Row-Level Locking

## Approach

Use SELECT ... FOR UPDATE inside a transaction.

Example:

BEGIN;

SELECT id
FROM seats
WHERE id = $1
AND status = 'available'
FOR UPDATE;

UPDATE seats
SET status = 'held',
    held_until = NOW() + INTERVAL '10 minutes'
WHERE id = $1;

COMMIT;

### How It Prevents Double Booking

The first transaction acquires the row lock.

Any other transaction attempting to access the same seat must wait.

Only one transaction can update the seat.

Therefore two bookings cannot confirm the same seat.

---

## Capacity Analysis

Assumptions:

- PgBouncer pool size = 500 connections
- 80% normal requests
- 20% payment requests
- Normal query time = 20ms
- Payment processing = 800ms

Formula:

Connections Held

= (0.8 × RPS × 0.02)

+ (0.2 × RPS × 0.8)

= 0.016RPS + 0.16RPS

= 0.176RPS

Pool exhaustion:

500 = 0.176RPS

RPS = 500 / 0.176

RPS ≈ 2841

### Finding

Pool exhausts at roughly:

2,841 RPS

This is far below expected Coldplay launch traffic.

---

## Risks

### Deadlocks

Scenario:

User A:
- Seat A12
- Seat A13

User B:
- Seat A13
- Seat A12

Both lock seats in different order.

Result:

Deadlock.

### Mitigation

Always lock seats in ascending seat_id order.

Example:

ORDER BY seat_id ASC

before acquiring locks.

---

# Option B: Redis Distributed Lock

## Approach

Acquire seat lock before touching database.

Key format:

seat_lock:{event_id}:{seat_id}

Example:

seat_lock:101:A12

---

## Lock Acquisition

SET lock_key value NX EX 30

Parameters:

NX = only create if not existing

EX = expire automatically

TTL = 30 seconds

If lock acquisition fails:

Return:

SEAT_TEMPORARILY_UNAVAILABLE

---

## Release Logic

Use Lua script.

if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end

This prevents deleting another user's lock.

---

## What If Redis Fails?

If Redis becomes unavailable:

- New locks cannot be created.
- Booking flow is paused.
- Existing confirmed bookings remain safe.

Because final booking confirmation still occurs inside PostgreSQL transaction.

No double-booking occurs.

Availability decreases but correctness remains intact.

---

## TTL Selection

Chosen TTL:

30 seconds

### Too Short

5 seconds:

- Slow network may expire lock.
- Another user acquires same seat.

### Too Long

5 minutes:

- Abandoned sessions block inventory.

### Why 30 Seconds

Enough time to:

- Verify seat
- Create booking
- Persist hold

Without blocking inventory excessively.

---

# Chosen Strategy

## Hybrid Approach

We choose:

Redis SETNX + PostgreSQL Transaction

### Booking Flow

1. Acquire Redis seat lock.
2. Verify seat availability.
3. Update seat to HELD.
4. Create booking record.
5. Commit transaction.
6. Release Redis lock.

---

# Why Hybrid?

### Capacity

PostgreSQL-only locking fails around:

≈ 2,841 RPS

Coldplay sale traffic can exceed:

50,000+ RPS

Redis handles:

100,000+ lock operations/sec

without stressing database connections.

---

### Cost

A small Redis cluster fits inside the $2,000 budget.

Adding dozens of DB replicas to solve lock contention would exceed budget.

---

### Correctness

Redis protects hot-seat contention.

PostgreSQL remains source of truth.

Final booking confirmation still benefits from ACID guarantees.

---

# Limitations

This strategy cannot handle:

- Complete Redis cluster outage
- Region-wide network failure
- Simultaneous multi-region booking races

Additional mechanisms would be required:

- Multi-AZ Redis
- Cross-region consensus
- Event sourcing

---

# When Would We Switch?

We would switch to PostgreSQL-only locking if:

- Concurrent booking traffic < 5,000 users
- Infrastructure budget extremely limited
- Simplicity is preferred over throughput

We would keep Hybrid for:

- IPL Finals
- Coldplay sales
- Cricket World Cup
- Any event expecting massive contention

---

# Final Decision

Chosen Strategy:

Redis SETNX Distributed Lock + PostgreSQL Transactions

Reason:

Provides the best balance between:

- Throughput
- Correctness
- Cost

while ensuring zero double-bookings under extreme traffic spikes.