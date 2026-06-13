# QuickBite Performance Optimization Audit

## Overview

This project focuses on profiling, analyzing, and optimizing the QuickBite backend API.

The objective was to identify performance bottlenecks, measure baseline system behavior, analyze database query execution, eliminate N+1 query patterns, add missing indexes, and validate improvements through benchmarking.

---

## Work Completed

### 1. Baseline Performance Profiling

Performed load testing using Artillery to establish current system performance.

Metrics collected:

- P50 latency
- P95 latency
- Error rate
- Throughput

Endpoints tested:

- GET /api/restaurants
- GET /api/restaurants/:id/menu
- GET /api/orders/history
- POST /api/orders

### Key Findings

- High response times on restaurant and order endpoints.
- Significant degradation under concurrent traffic.
- Increased error rates during peak load.

---

## 2. Query Count Instrumentation

Added query-count logging middleware to measure database activity per request.

### Findings

| Endpoint | Queries |
|-----------|----------|
| GET /api/restaurants | 1 |
| GET /api/restaurants/:id/menu | 23 |
| GET /api/orders/history | 101 |

### Issues Identified

- Multiple database queries executed inside loops.
- Severe N+1 query problem in order history.
- Excessive database round trips.

---

## 3. Database Profiling with EXPLAIN ANALYZE

Analyzed slow queries using PostgreSQL EXPLAIN ANALYZE.

### Queries Investigated

1. Order History Query
2. Order Items Lookup
3. Restaurant Menu Query

### Findings

- Sequential scans on large tables.
- Missing indexes.
- Poor query execution plans.
- Large number of rows scanned unnecessarily.

---

## 4. N+1 Query Optimization

### Problem

Application fetched related records individually inside loops.

Example:

```js
for (const order of orders) {
  await db.query(...)
}