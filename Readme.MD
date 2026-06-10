# ARCHITECTURE.md

## Current Architecture

Internet
    │
    ▼
┌─────────────────────────────┐
│ Single Node.js Server       │
│ - All business logic        │
│ - No cache                  │
│ - SPOF (single failure)     │
│ - N+1 queries (Part A Bug 5)│
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ PostgreSQL                  │
│ - Reads + Writes together   │
│ - No read replicas          │
│ - Missing indexes           │
└─────────────────────────────┘

### Weaknesses Found in Part A

- Bug 1: SQL Injection
- Bug 2: Plaintext Passwords
- Bug 3: Double Discount
- Bug 4: Stock Decrement Failure
- Bug 5: N+1 Query (14s response time)

---

## Proposed Architecture (1 Lakh Users)

Users
  │
  ▼
CDN
  │
  ▼
Load Balancer
  │
 ├─────────────┬─────────────┐
 ▼             ▼             ▼
Node 1       Node 2       Node 3
  │             │             │
  └───────Redis Cluster───────┘
                │
                ▼
       PostgreSQL Primary
                │
        ┌───────┴───────┐
        ▼               ▼
 Read Replica 1   Read Replica 2

### Justifications

#### Redis
Part A showed GET /products hitting DB every time.
Redis reduces response time from ~300ms to <10ms.

#### Read Replicas
Part A Bug 5 showed read-heavy traffic.
Replicas separate reads from writes.

#### Load Balancer
Prevents single-server crashes.

#### CDN
Offloads image traffic from Node servers.