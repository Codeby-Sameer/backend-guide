# Distributed Systems & Scaling Cheatsheet

A concise reference sheet for distributed systems theorems, messaging semantics, caching patterns, and resiliency patterns.

---

## 1. Core Distributed Theorems

### CAP Theorem
In any asynchronous network subject to **Network Partitions ($P$)**, a system can guarantee at most two of the three:
- **CP (Consistency + Partition Tolerance):** Rejects writes or errors if consensus cannot be reached (e.g., CockroachDB, etcd, ZooKeeper, HBase).
- **AP (Availability + Partition Tolerance):** Accepts writes on all nodes, returning potentially stale data (e.g., Cassandra, DynamoDB, CouchDB).
- *(CA is impossible over public networks because physical partitions are inevitable).*

### PACELC Theorem
$$\text{If Partition (P)} \to \text{Choose between (A)vailability and (C)onsistency}$$
$$\text{Else (E)} \to \text{Choose between (L)atency and (C)onsistency}$$

---

## 2. Distributed Resilience Patterns

```
┌───────────────────────────┬───────────────────────────────────────────────────────────────────────────┐
│ Resilience Pattern        │ Core Mechanism & Purpose                                                  │
├───────────────────────────┼───────────────────────────────────────────────────────────────────────────┤
│ **Circuit Breaker**       │ Trips to 'OPEN' state after 50% failures, failing fast to allow upstream  │
│                           │ services to recover without receiving traffic.                            │
├───────────────────────────┼───────────────────────────────────────────────────────────────────────────┤
│ **Exponential Backoff**   │ Increases delay between retries exponentially ($T = 2^{\text{attempt}}$)  │
│                           │ with randomized **Jitter** to prevent Thundering Herd / Retry Storms.     │
├───────────────────────────┼───────────────────────────────────────────────────────────────────────────┤
│ **Bulkhead Pattern**      │ Isolates resources (thread pools, connection pools) so a failure in one   │
│                           │ downstream service does not exhaust resources for other services.         │
├───────────────────────────┼───────────────────────────────────────────────────────────────────────────┤
│ **Transactional Outbox**  │ Solves the Dual-Write problem by writing state and events to a single DB  │
│                           │ in 1 ACID transaction, using CDC (Debezium) to publish to Kafka.          │
├───────────────────────────┼───────────────────────────────────────────────────────────────────────────┤
│ **Saga Pattern**          │ Replaces blocking 2PC with choreographed or orchestrated local            │
│                           │ transactions combined with explicit **Compensating Transactions**.       │
└───────────────────────────┴───────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Distributed Messaging & Kafka Matrix

| Concept | Guarantee / Mechanism |
| :--- | :--- |
| **Ordering** | Guaranteed **strictly within a partition**; non-deterministic across partitions. |
| **Partition Key** | $\text{MurmurHash2}(\text{key}) \pmod{\text{num\_partitions}}$ |
| **At-Least-Once** | Process message $\to$ Commit offset. Requires **Idempotent Consumers**. |
| **At-Most-Once** | Commit offset $\to$ Process message. Risk of lost messages on crash. |
| **Rebalance Protocol**| **Cooperative Sticky Assignor** (avoids stop-the-world eager rebalance pauses).|
