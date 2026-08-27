# Backend Interview Questions: Intermediate Tier

A rigorous collection of intermediate backend engineering questions focusing on internal engine mechanics, performance bottlenecks, concurrency anomalies, and trade-off analysis.

---

### Q1. Why does an index speed up database reads but slow down database writes?
**Answer:**
- **Reads ($O(\log N)$):** An index builds a balanced B+ Tree sorted by the indexed column. Queries traverse 3–4 node levels to locate a row rather than scanning millions of un-indexed table pages ($O(N)$).
- **Writes Overhead:** Every `INSERT`, `DELETE`, and `UPDATE` on indexed columns requires the database engine to synchronously update the main table heap **AND** every single secondary B+ Tree index. Inserting out-of-order keys causes expensive **B-Tree Page Splits**, disk I/O, and Write-Ahead Log (WAL) bloat.  
**Example:** Adding 10 indexes to an `orders` table speeds up reporting queries but cuts insert throughput from 5,000 writes/sec to 800 writes/sec.  
**Why this matters:** Balancing read optimization against write throughput.  
**Possible follow-up:** What is a B-Tree page split and how does it cause disk fragmentation?

---

### Q2. How does the Leftmost-Prefix Principle govern composite index usage?
**Answer:** A composite index on `(A, B, C)` sorts data hierarchically by `A`, then ties by `B`, then ties by `C`.
- Queries filtering by `(A)`, `(A, B)`, or `(A, B, C)` utilize fast B-Tree index seeks because the leading edge matches the tree sorting.
- Queries filtering solely by `(B)` or `(B, C)` cannot seek the index and must perform a full index scan or table scan because values of `B` are scattered across different values of `A`.  
**Example:** A telephone directory sorted by `(LastName, FirstName)` allows looking up "Smith" or "Smith, Bob" instantly, but is useless for finding all people named "Bob".  
**Why this matters:** Core rule for designing compound database indexes.  
**Possible follow-up:** What happens if the query uses `WHERE A = 5 AND B > 10 AND C = 20`?

---

### Q3. Explain Multi-Version Concurrency Control (MVCC) and why it is superior to traditional read/write locks.
**Answer:** In traditional locking systems, readers acquire Shared (S) locks and block writers, while writers acquire Exclusive (X) locks and block readers, causing severe lock contention.  
**MVCC** solves this by maintaining multiple historical versions of each row. When a transaction updates a row, the database creates a new version tagged with the transaction ID. Read queries access an immutable snapshot corresponding to their transaction start time.
*Result:* **Readers never block writers, and writers never block readers.**  
**Example:** PostgreSQL uses `xmin` and `xmax` tuple headers to determine row version visibility for each transaction snapshot.  
**Why this matters:** Foundation of modern high-concurrency relational databases (PostgreSQL, MySQL InnoDB).  
**Possible follow-up:** What is the downside of MVCC (table bloat, VACUUM)?

---

### Q4. What is the difference between Pessimistic and Optimistic Locking, and when should you choose each?
**Answer:**
- **Pessimistic Locking (`SELECT FOR UPDATE`):** Explicitly locks the database row upfront, forcing concurrent transactions to wait. Best for **high-contention** workloads (e.g., flash sales, inventory booking, ticket sales) where collisions are frequent.
- **Optimistic Locking (`WHERE version = ?`):** Holds zero locks during read. Increments a `version` column upon update and checks that the version hasn't changed. Best for **low-contention** workloads and long user think times (e.g., editing a profile or wiki article).  
**Example:** Booking the last seat on a flight $\to$ Pessimistic Lock. Editing a blog post description $\to$ Optimistic Lock.  
**Why this matters:** Core concurrency pattern selection.  
**Possible follow-up:** How does optimistic locking behave when 1,000 users update the same row simultaneously?

---

### Q5. What is the N+1 Query Problem and how do you resolve it?
**Answer:** The N+1 problem occurs when an application executes 1 query to fetch $N$ parent records, and subsequently runs $N$ separate queries inside a loop to fetch related child records. This creates $N+1$ database round-trips, causing severe network latency and connection pool exhaustion.  
**Solutions:**
1. **SQL JOIN (Eager Loading):** `SELECT * FROM authors LEFT JOIN books ON ...` (1 query).
2. **Batched Subquery (`IN (...)`):** Fetch parents, then fetch all children via `WHERE author_id IN (1, 2, ... N)` (2 queries total).
3. **GraphQL DataLoader:** Batches IDs across event loop ticks.  
**Example:** Fetching 100 users and their company details triggers 101 queries without eager loading, taking 500ms; with batched loading it takes 2 queries in 8ms.  
**Why this matters:** #1 cause of performance degradation in ORM-based backends.  
**Possible follow-up:** When is a `JOIN` worse than two separate batched queries?

---

### Q6. Why is a database connection pool necessary and how should it be sized?
**Answer:** Establishing a raw database connection is expensive: it requires TCP 3-way handshake, TLS negotiation, authentication, and server-side process/memory allocation (~10MB per Postgres worker). A connection pool maintains pre-established, persistent connections that application threads borrow and return.  
**Sizing:** Contrary to intuition, large pools (e.g., 1,000 connections) crash databases through CPU context switching and disk queue contention. Sizing follows HikariCP's formula:
$$\text{Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindle Count}$$
A 16-core database performs optimally with a pool of ~33 connections.  
**Why this matters:** System capacity planning and resource optimization.  
**Possible follow-up:** What is the difference between client-side connection pooling and PgBouncer?

---

### Q7. Explain the Cache-Aside pattern and why you should Delete rather than Update cache keys on DB write.
**Answer:**
- **Read Flow:** App checks Redis $\to$ if Hit, return data $\to$ if Miss, read DB, write to Redis with TTL, return data.
- **Write Flow:** App updates the Database, then **deletes (invalidates)** the Redis key.
- **Why Delete instead of Update?** Updating the cache on write creates race conditions: if Thread 1 writes DB (Val A) and Thread 2 writes DB (Val B), but Thread 2 updates Redis before Thread 1 due to network jitter, Redis ends up with stale Val A permanently. Invalidation avoids the race condition by forcing the next read to fetch fresh DB data.  
**Why this matters:** Fundamental cache consistency design.  
**Possible follow-up:** What is the difference between Cache-Aside and Write-Through caching?

---

### Q8. What is the difference between Apache Kafka and traditional message brokers like RabbitMQ?
**Answer:**
- **RabbitMQ (Traditional Queue):** Operates on a **Push model**. Messages are transient jobs pushed to consumers and deleted immediately once acknowledged (ACK). Best for discrete background tasks and complex exchange routing.
- **Apache Kafka (Distributed Commit Log):** Operates on a **Pull model**. Messages are appended to immutable, partitioned commit logs on disk and retained based on a time/size TTL. Consumers track their own offsets and can replay historical events. High throughput ($>1\text{M}$ msgs/sec).  
**Why this matters:** Choosing the right messaging backbone for microservices.  
**Possible follow-up:** How does Kafka guarantee message ordering?

---

### Q9. What is the Token Bucket rate limiting algorithm and why is it preferred over Fixed Window Counter?
**Answer:**
- **Fixed Window Counter:** Resets request count at fixed time boundaries (e.g., top of the minute), which allows a "boundary burst" of $2\times$ the limit across the boundary edge (e.g., 100 reqs at 00:59 + 100 reqs at 01:01 = 200 reqs in 2 seconds).
- **Token Bucket:** A bucket holds up to $B$ tokens and refills continuously at rate $r$ tokens/sec. Each request consumes 1 token. It permits short, natural traffic bursts while mathematically bounding the long-term average rate to $r$.  
**Why this matters:** Industry standard algorithm for production APIs (AWS, Stripe).  
**Possible follow-up:** How do you implement a token bucket in Redis without running a continuous background refill loop?

---

### Q10. Why is Average Latency a misleading metric, and why are p95/p99 percentiles preferred?
**Answer:** Average (arithmetic mean) latency hides severe outlier degradation and bimodal distributions. If 99 requests take 10ms and 1 request hangs for 10,000ms, the average is 110ms—masking the fact that 1% of users suffered a 10-second freeze.  
**p99 (99th percentile tail latency)** measures the worst 1 in 100 requests, accurately surfacing lock contention, database query spikes, and garbage collection pauses.  
**Why this matters:** Core Site Reliability Engineering (SRE) measurement principle.  
**Possible follow-up:** What is the difference between an SLI, an SLO, and an SLA?
