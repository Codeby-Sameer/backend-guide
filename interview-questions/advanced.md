# Backend Interview Questions: Advanced Tier

A collection of senior and staff-level backend engineering questions focusing on distributed consensus, kernel-level operations, concurrency anomalies, and deep architectural trade-offs.

---

### Q1. What is the Write Skew anomaly, and why does Snapshot Isolation (Repeatable Read) fail to prevent it?
**Answer:** Write Skew occurs when two concurrent transactions read overlapping or disjoint data, evaluate a shared business invariant, and execute non-conflicting write mutations on *different* rows that collectively violate the invariant.
- **Why Repeatable Read Fails:** Snapshot Isolation / `REPEATABLE READ` only detects conflicts when two transactions attempt to update or delete the **exact same row**. Because the transactions modify different rows, both commit without error, resulting in corrupted business state.
- **Example:** The "On-Call Doctor" problem: At least 1 doctor must be on call. Doctors Alice and Bob both read that 2 doctors are on call ($2 \ge 1$). Alice takes herself off call (updates Alice's row); Bob simultaneously takes himself off call (updates Bob's row). Both transactions commit, leaving zero doctors on call.
- **Remediation:** Explicit locking (`SELECT ... FOR UPDATE`), PostgreSQL `SERIALIZABLE` isolation (SSI), or unique constraints.  
**Why this matters:** Tests deep understanding of database isolation limits.  
**Possible follow-up:** How does PostgreSQL SSI (Serializable Snapshot Isolation) detect write skew without pessimistic locks?

---

### Q2. How does PostgreSQL Serializable Snapshot Isolation (SSI) detect serialization anomalies without blocking readers?
**Answer:** PostgreSQL SSI tracks **Read-Write Dependencies (rw-antidependencies)** in volatile memory using **SIREAD locks** (predicate locks) rather than blocking transactions with physical locks:
1. When a transaction reads a row or index range, an ephemeral SIREAD lock is registered in shared memory.
2. If another transaction writes to a row covered by a SIREAD lock, an "inbound rw-antidependency" edge is recorded in the transaction dependency graph.
3. The engine monitors for **cycles of dependencies** (indicating non-serializable interleaving: $T_1 \xrightarrow{rw} T_2 \xrightarrow{rw} T_1$).
4. When a cycle is detected, the engine immediately aborts one transaction with `ERROR: could not serialize access due to read/write dependencies among transactions (SQLSTATE 40001)`.  
**Why this matters:** Explains the state of the art in optimistic lock-free serializable database engines.  
**Possible follow-up:** What is the memory overhead of maintaining SIREAD locks?

---

### Q3. Why are Distributed Locks with TTLs (like Redis Redlock) unsafe for strong consistency without Fencing Tokens?
**Answer:** In distributed systems, a process holding a lock can experience an unpredictable **Stop-the-World Garbage Collection (GC) pause**, disk I/O stall, or network partition:
1. Client 1 acquires Lock $L$ from Redis with a 10-second TTL.
2. Client 1 enters a 12-second GC pause.
3. Lock $L$ expires in Redis.
4. Client 2 acquires Lock $L$ and begins modifying shared storage.
5. Client 1 wakes up, believes it still holds the lock, and writes to shared storage, corrupting data.
**The Solution (Fencing Tokens):** The lock server issues a monotonically increasing token (e.g., Token 34, Token 35). The storage service checks the token on every write; if a write arrives with a token smaller than the highest seen token, it is rejected.  
**Why this matters:** Famous distributed systems critique by Martin Kleppmann on distributed lock safety.  
**Possible follow-up:** How does etcd provide safe distributed leasing using raft-based revisions?

---

### Q4. What is the Transactional Outbox Pattern and how does it solve the "Dual-Write" distributed data problem?
**Answer:**
- **The Problem:** A backend service must update a database and publish an event to Kafka. If the DB commit succeeds but the network to Kafka drops, the event is lost. If Kafka publishes first but the DB transaction rolls back, downstream services act on phantom data.
- **The Outbox Solution:**
  1. In the *same local ACID transaction*, write the business mutation to the main table AND write the event payload to an `outbox` table.
  2. A separate background process (e.g., **Debezium Change Data Capture (CDC)** via Postgres WAL or an outbox polling worker) reads the `outbox` table and reliably publishes messages to Kafka.
  3. Guarantees that database updates and event publishing never diverge.  
**Why this matters:** The industry standard pattern for reliable event-driven microservices.  
**Possible follow-up:** How does Debezium read Postgres WAL logs via Logical Replication?

---

### Q5. How does Zero-Copy Data Transfer work in the Linux kernel, and why does Apache Kafka rely on it?
**Answer:** In traditional data transfer from disk to network, data is copied 4 times across user and kernel space:
1. Disk $\to$ OS Page Cache (Kernel space)
2. Page Cache $\to$ Application Buffer (User space via `read()`)
3. Application Buffer $\to$ Socket Buffer (Kernel space via `write()`)
4. Socket Buffer $\to$ Network Interface Card (NIC) Buffer.
**Zero-Copy (`sendfile()` system call):** Data is transferred **directly from OS Page Cache to the NIC buffer** within the kernel space, completely bypassing user space and CPU copy instructions. This allows Kafka to saturate 10Gbps network interfaces with near-zero CPU consumption.  
**Why this matters:** Demonstrates mastery of OS kernel I/O mechanics and high-performance systems design.  
**Possible follow-up:** What are the constraints of using `sendfile` when TLS encryption is required?

---

### Q6. How does Consistent Hashing with Virtual Nodes prevent data skew during cluster scaling?
**Answer:**
- Standard modulo hashing ($\text{hash}(\text{key}) \pmod N$) remaps almost 100% of keys when node count $N$ changes.
- **Consistent Hashing** maps nodes and keys onto a circular $360^\circ$ hash ring. Adding a node only remaps $K/N$ keys from its direct neighbor.
- **The Virtual Nodes Solution:** If physical nodes are placed on the ring randomly, uneven distribution causes hotspots (data skew). By assigning each physical machine 100–250 **Virtual Nodes** scattered across the ring, the hash space is uniformly distributed, and adding a new server absorbs proportional load from *all* existing servers uniformly.  
**Why this matters:** Core architecture of Amazon DynamoDB, Apache Cassandra, and Discord voice routing.  
**Possible follow-up:** How do you rebalance virtual nodes when adding a new physical host?

---

### Q7. What are the security risks of 0-RTT (Early Data) in TLS 1.3, and how do you mitigate them?
**Answer:**
- **The Vulnerability:** TLS 1.3 allows returning clients to send encrypted application data in the very first `ClientHello` packet (0-RTT session resumption). Because this data is sent before a new ephemeral key exchange completes, an attacker who intercepts the packet can **replay it to the server (Replay Attack)**.
- **Mitigation:**
  1. Only allow 0-RTT for strictly **safe and idempotent HTTP methods (`GET`, `HEAD`)**.
  2. Reject 0-RTT on state-mutating requests (`POST /v1/charges`).
  3. Implement anti-replay caches on load balancers (tracking ticket age and single-use nonces).  
**Why this matters:** Critical network protocol security engineering.  
**Possible follow-up:** How do web browsers signal that a request is early data?

---

### Q8. What is the Metric Cardinality Explosion problem in Prometheus, and how do you prevent it?
**Answer:** In Prometheus, every unique combination of metric name and label key-value pairs generates a distinct in-memory time series.
- If a developer adds a label `user_id` or `uuid` to a metric, 10 million users create 10 million distinct time series, consuming gigabytes of RAM and crashing the TSDB engine via Out-Of-Memory (OOM).
- **Prevention:**
  1. Enforce strict linting rules: Labels must only contain **low-cardinality bounded enums** (e.g., `status_code`, `http_method`, `region`).
  2. Put high-cardinality values (`user_id`, `request_id`, `ip_address`) exclusively in **Structured Logs** and **Distributed Trace Spans**.  
**Why this matters:** One of the most common catastrophic infrastructure outages in Kubernetes environments.  
**Possible follow-up:** How do you configure Prometheus `relabel_configs` to drop unsafe labels?

---

### Q9. How do you implement end-to-end W3C Distributed Trace Context propagation across heterogeneous microservices?
**Answer:** 
1. The edge API Gateway intercepts the request, generates a 128-bit `trace_id` and 64-bit `span_id`, and sets the W3C header:
   `traceparent: 00-{trace_id}-{span_id}-01`
2. Every microservice uses OpenTelemetry instrumentation to extract `traceparent` from incoming HTTP/gRPC/Kafka headers.
3. The runtime binds the trace context to thread-local or asynchronous context storage (e.g., Go `context.Context`, Node.js `AsyncLocalStorage`, Java `MDC`).
4. When making outbound calls, OpenTelemetry injects the updated header with a new child `span_id`.
5. Background exporters asynchronously stream spans via OTLP/gRPC to collectors (Jaeger, Tempo) without blocking application threads.  
**Why this matters:** Essential for enterprise-wide distributed observability.  
**Possible follow-up:** What is the difference between Head-based and Tail-based sampling in OpenTelemetry?

---

### Q10. What limitations arise when using PgBouncer in Transaction Pooling mode?
**Answer:** In Transaction Pooling mode, a physical PostgreSQL connection is released back to the pool immediately upon `COMMIT` or `ROLLBACK`. Consequently:
1. **Session-Level State is Lost / Polluted:** Commands like `SET TIME ZONE`, `SET search_path`, and `LISTEN/NOTIFY` do not persist across transactions or can bleed into subsequent clients.
2. **Session-Scoped Advisory Locks Fail:** `pg_advisory_lock()` remains tied to the physical connection, meaning another random transaction might inherit your lock! (Fix: Use transaction-scoped `pg_advisory_xact_lock()`).
3. **Prepared Statements Require Special Configuration:** Legacy drivers prepare statements at the session level; modern drivers use named prepared statement workarounds or transaction-scoped preparation.  
**Why this matters:** Essential operational knowledge for scaling PostgreSQL under high load.  
**Possible follow-up:** How does AWS RDS Proxy handle session state pinning?
