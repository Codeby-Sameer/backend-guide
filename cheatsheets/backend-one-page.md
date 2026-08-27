# Backend Master One-Page Cheatsheet

A concise, high-density reference sheet summarizing key backend formulas, status codes, algorithms, and architectural rules.

---

## 1. System Design Quick Formulas

| Concept | Formula / Rule |
| :--- | :--- |
| **QPS / Throughput** | $\text{QPS} = \frac{\text{Daily Active Users (DAU)} \times \text{Avg Requests/Day}}{86,400\text{ seconds}}$ |
| **Peak QPS** | $\text{Peak QPS} = \text{Average QPS} \times 2 \text{ (or } 3\text{)}$ |
| **Storage Growth** | $\text{Storage/Year} = \text{Daily Writes} \times \text{Average Payload Size} \times 365$ |
| **Optimal DB Connection Pool** | $\text{Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindles}$ (HikariCP) |
| **Sliding Window Counter Estimate**| $\text{Count} = (\text{Prev Count} \times (1 - \text{Overlap})) + \text{Current Count}$ |
| **Tail Latency Compounding** | $P(\text{Composite Spike}) = 1 - (1 - P(\text{Single Spike}))^N$ |

---

## 2. HTTP Status Codes Cheat Sheet

```
200 OK              - Successful request
201 Created         - Resource created (include Location header)
202 Accepted        - Async job accepted for background processing
204 No Content      - Action succeeded, no body returned (DELETE/PUT)
301 Moved Perm      - Permanent redirect (cached by browser)
304 Not Modified    - Resource unchanged (ETag / If-None-Match cache hit)
400 Bad Request     - Malformed JSON / invalid syntax
401 Unauthorized    - Unauthenticated (missing/invalid credentials)
403 Forbidden       - Authenticated but unauthorized (insufficient role)
404 Not Found       - Resource URI does not exist
409 Conflict        - State conflict (duplicate idempotency key / version mismatch)
422 Unprocessable   - Valid JSON syntax but fails business schema validation
429 Too Many Req    - Rate limit exceeded (include Retry-After header)
500 Internal Error  - Unhandled backend exception / bug
502 Bad Gateway     - Upstream server returned invalid/dropped response
503 Unavailable     - Service overloaded or down for maintenance
504 Gateway Timeout - Upstream server took too long to respond
```

---

## 3. Database Indexing & ACID Rules

```
Rule 1: Leftmost-Prefix Principle
Index on (A, B, C) -> Supports WHERE A; WHERE A AND B; WHERE A AND B AND C.
Does NOT support WHERE B; WHERE C; WHERE B AND C.

Rule 2: Equality First, Range Second
Place exact equality columns (=) before range columns (<, >, BETWEEN).

Rule 3: ACID Implementation
Atomicity   -> Undo Log / MVCC
Consistency -> Invariants, Schema Constraints, Foreign Keys
Isolation   -> Locks & MVCC Snapshots
Durability  -> Write-Ahead Log (WAL) with fsync()

Rule 4: Isolation Level Hierarchy
Read Uncommitted < Read Committed < Repeatable Read < Serializable
PostgreSQL default: Read Committed | MySQL InnoDB default: Repeatable Read
```

---

## 4. Cache & Concurrency Patterns

```
Cache-Aside Pattern:
- Read: Check Redis -> Hit: Return -> Miss: Read DB, Write Redis with TTL.
- Write: Write to DB -> DEL Redis Key.

Cache Pitfalls & Mitigations:
- Cache Stampede   -> Mutex / Single-Flight / XFetch early expiration
- Cache Penetration -> Bloom Filter / Cache Null with short TTL
- Cache Avalanche   -> TTL Jitter (+ random 0..300s)

Concurrency Hierarchy:
1. Atomic SQL: UPDATE items SET stock = stock - 1 WHERE id = 1 AND stock >= 1
2. Pessimistic Lock: SELECT ... FOR UPDATE (Sort IDs ascending to prevent deadlocks)
3. Optimistic Lock: UPDATE ... WHERE version = ?
4. Distributed Lock: Redis SET NX PX with atomic Lua release script
```

---

## 5. Message Queuing & Kafka Rules

```
- Kafka unit of ordering: Partition (Total order ONLY within a single partition).
- Kafka unit of parallelism: Number of partitions = Max consumers in a Consumer Group.
- Delivery Guarantee: At-Least-Once delivery + Idempotent Consumer processing.
- Dual-Write Solution: Transactional Outbox Pattern + Debezium CDC.
```

---

## 6. Observability Pillars (RED & USE)

```
RED Method (APIs):
- Rate     : http_requests_total (Counter)
- Errors   : http_requests_5xx (Counter)
- Duration : http_request_duration_seconds (Histogram -> p50, p95, p99)

USE Method (Infrastructure):
- Utilization : % time resource was busy (CPU, Disk)
- Saturation  : Queue depth (Load Average, Thread Pool Queue)
- Errors      : Device / Packet errors
```
