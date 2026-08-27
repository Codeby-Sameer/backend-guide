# Backend Interview Questions: Production Scenarios & Incident Debugging

A collection of realistic, high-stakes production incident scenarios designed to test troubleshooting methodologies, root-cause analysis, and system resilience engineering.

---

### Scenario 1: Flash Sale Inventory Overselling Bug
**The Situation:** During a limited flash sale where 50 units of a high-demand item were available, the order processing system sold 180 units. Customer support is flooded with complaints.
**Triage & Root Cause Analysis:**
- Inspect application code and database queries.
- Found the classic **Check-Then-Act race condition**:
  `SELECT stock FROM items WHERE id = ?` $\to$ in Python: `if stock > 0:` $\to$ `UPDATE items SET stock = stock - 1`.
- Under 10,000 concurrent requests, multiple threads read `stock = 1` before any thread updated the database.
**Immediate Remediation & Permanent Architecture Fix:**
1. **Immediate Mitigation:** Halt checkout on the affected SKU; trigger refunds with promo vouchers.
2. **Permanent Fix:** Enforce an atomic SQL decrement with an invariant constraint:
   ```sql
   UPDATE items SET stock = stock - 1 WHERE id = :id AND stock >= 1;
   ```
   Check `rows_affected == 1`. If 0, immediately return "Sold Out".
3. **Defense-in-Depth:** Add a database check constraint: `ALTER TABLE items ADD CONSTRAINT chk_stock_non_negative CHECK (stock >= 0);`.

---

### Scenario 2: 100% Database CPU Saturation During Traffic Surge
**The Situation:** During peak traffic, PostgreSQL CPU spikes to 100%, query response times jump from 15ms to 8,000ms, and HTTP 504 gateway timeouts flood client apps.
**Triage Steps:**
1. Run `pg_stat_activity` to find long-running queries:
   ```sql
   SELECT pid, now() - query_start AS duration, query, state 
   FROM pg_stat_activity 
   WHERE state = 'active' ORDER BY duration DESC;
   ```
2. Identify the culprit: A new query `SELECT * FROM orders WHERE DATE(created_at) = CURRENT_DATE` is performing a Sequential Scan over 40 million rows.
3. Run `EXPLAIN (ANALYZE, BUFFERS)` to confirm the `DATE()` wrapper prevents the B-Tree index on `created_at` from being used.
**Resolution:**
- Rewrite the query to use an index range:
  `WHERE created_at >= CURRENT_DATE AND created_at < CURRENT_DATE + INTERVAL '1 day'`.
- Deploy an immediate hotfix. CPU instantly drops from 100% to 14%.

---

### Scenario 3: Database Connection Pool Exhaustion Caused by Third-Party API
**The Situation:** Backend API pods begin throwing `HikariPool - Connection is not available, request timed out after 30000ms`. All database operations fail.
**Triage Steps:**
1. Check Prometheus metrics: `active_db_connections` is pegged at max pool size (`20`), while `db_cpu` is only 5% (Database is idle!).
2. Profile application thread dumps (`jstack` / Go pprof).
3. Discover that the user checkout handler was executing an external payment API call **inside an active database transaction**:
   ```python
   with db.transaction():
       db.execute("UPDATE orders SET status='PROCESSING' WHERE id=1")
       stripe.charges.create(...) # Hangs for 25 seconds due to Stripe network degradation!
   ```
**Resolution:**
- Move all external network calls **outside** of the database transaction.
- Wrap external HTTP calls in aggressive client timeouts (e.g., 2.5 seconds) and circuit breakers.

---

### Scenario 4: Kafka Consumer Lag Ballooning Out of Control
**The Situation:** In an order fulfillment pipeline, Kafka consumer lag increases by 100,000 messages every 15 minutes. Downstream email notifications and inventory syncs are delayed by hours.
**Triage Steps:**
1. Check consumer lag per partition using `kafka-consumer-groups.sh --describe`.
2. Observe that the topic has **4 partitions**, but the team scaled the Kubernetes deployment to **12 consumer pods**.
3. **Root Cause:** 8 consumer pods are completely idle because a partition can only be consumed by 1 pod in the same group! The 4 active pods are bottlenecked by single-threaded synchronous processing.
**Resolution:**
1. Increase the topic partition count from 4 to 16.
2. The 12 consumer pods automatically rebalance (via Cooperative Sticky Assignor), parallelizing throughput $4\times$.
3. Inside each consumer pod, introduce a worker thread pool to process independent message keys asynchronously while maintaining offset commits.

---

### Scenario 5: Cache Avalanche Crash at Midnight
**The Situation:** Exactly at 00:00 UTC, the primary database crashes under an unprecedented read tsunami.
**Triage Steps:**
1. Check Redis telemetry: Cache hit ratio drops from 98% to 10% within 10 seconds.
2. Review backend caching code: All product catalog items were cached with a fixed TTL of `86400` seconds (24 hours) during a midnight batch job yesterday.
3. At 00:00 UTC today, all 1,000,000 keys expired simultaneously, sending 50,000 req/s directly to PostgreSQL.
**Resolution:**
- Implement **TTL Jitter**: `TTL = 86400 + random.randint(0, 3600)` (spreads expiration over a 1-hour window).
- Add a distributed mutex / single-flight loader so that on cache miss, only 1 worker queries the DB for a given product ID while concurrent threads wait for the cache to populate.

---

### Scenario 6: Kubernetes Container CrashLoopBackOff (Exit Code 137)
**The Situation:** An image processing microservice crashes repeatedly under load with status `OOMKilled (Exit Code 137)`.
**Triage Steps:**
1. Run `kubectl describe pod <pod_name>`:
   ```text
   Last State: Terminated
   Reason: OOMKilled
   Exit Code: 137
   ```
2. Identify that the container memory limit was set to `512Mi` in the Kubernetes pod spec.
3. Analyze memory profile: When processing large 4K image uploads, the image buffer was decompressed into raw uncompressed bitmaps in RAM (each consuming 80MB), exceeding the cgroup limit and triggering the Linux kernel OOM Killer.
**Resolution:**
- Stream image files directly to disk / temporary storage using chunked streams instead of buffering full files into RAM.
- Increase memory request/limit to `1Gi` and set JVM/Go heap thresholds properly.

---

### Scenario 7: Duplicate Billing Caused by Client Network Timeouts
**The Situation:** A customer was billed 4 times for a single flight booking after experiencing a poor mobile connection.
**Triage Steps:**
1. Review access logs: 4 identical `POST /v1/bookings` requests were received with 5-second intervals from the same user.
2. The first request succeeded on the server, but the cellular network dropped the response packet. The mobile app timed out and retried 3 times.
**Resolution:**
- Enforce the **Idempotency-Key Pattern**: The mobile app generates a UUID before the first attempt and sends `Idempotency-Key: <UUID>`.
- The backend acquires an atomic lock in Redis/DB for the UUID. Subsequent retries return the cached original response without re-charging the credit card.

---

### Scenario 8: Fleet-Wide Internal Service Outage Due to Expired CA Certificate
**The Situation:** At 03:00 UTC, all internal gRPC inter-service communication fails across 40 microservices with `remote error: tls: bad certificate`.
**Triage Steps:**
1. Run `openssl s_client -connect user-service:8443 -showcerts` from an affected pod.
2. Discover that the internal Root CA certificate expired.
**Immediate & Permanent Fix:**
- **Emergency Fix:** Generate an updated CA bundle, deploy via Kubernetes ConfigMap, and restart pods.
- **Permanent Fix:** Implement automated certificate rotation (cert-manager with HashiCorp Vault) issuing short-lived 24-hour certificates with automated Prometheus alerts on certificates expiring within 30 days.

---

### Scenario 9: Broken Object Level Authorization (BOLA) Security Leak
**The Situation:** A security researcher reports that changing the URL from `/api/v1/invoices/inv_100` to `/api/v1/invoices/inv_101` allows viewing any company's private invoice data.
**Triage Steps:**
1. Audit the invoice controller code:
   ```python
   # VULNERABLE:
   def get_invoice(invoice_id):
       return db.query("SELECT * FROM invoices WHERE id = :id", id=invoice_id)
   ```
2. The endpoint authenticated the user (AuthN), but failed to authorize ownership (AuthZ).
**Resolution:**
- Always scope queries by the authenticated user's organization:
  ```python
  def get_invoice(invoice_id):
      return db.query(
          "SELECT * FROM invoices WHERE id = :id AND org_id = :current_user_org",
          id=invoice_id, current_user_org=g.current_user.org_id
      )
  ```
- Switch from sequential integer IDs to opaque UUIDv7 strings (`inv_018f...`).

---

### Scenario 10: Uncontrolled Retry Storm (Thundering Herd) Crashing Downstream Service
**The Situation:** When Service B experienced a 10-second network blip, Service A sent immediate retries for all 5,000 failing requests, overwhelming Service B and preventing it from recovering.
**Resolution:**
1. Enforce **Exponential Backoff with Full Jitter** on all client and microservice retries:
   $$T_{\text{sleep}} = \text{random}(0, \min(\text{max\_backoff}, \text{base} \times 2^{\text{attempt}}))$$
2. Implement **Circuit Breakers** (Netflix Hystrix / Resilience4j): After 50% failure rate over 10 seconds, trip the circuit to "OPEN" for 30 seconds, failing fast without sending network traffic to Service B.
