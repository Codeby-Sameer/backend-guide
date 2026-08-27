# Race Conditions & Distributed Concurrency

## 1. One-minute explanation

A **Race Condition** occurs in concurrent and distributed systems when two or more threads or processes access shared mutable state simultaneously, and the final outcome depends on the non-deterministic timing, scheduling, or order of execution. The most notorious patterns are **Read-Modify-Write** (lost updates) and **Check-Then-Act** (time-of-check to time-of-use / TOCTOU bugs). In backend systems, race conditions lead to catastrophic bugs like inventory overselling, double-spending in digital wallets, and unauthorized duplicate transactions. We eliminate race conditions using **Atomic SQL Updates**, **Pessimistic Row Locking** (`SELECT FOR UPDATE`), **Optimistic Concurrency Control** (version tokens), or **Distributed Mutexes** (Redis / etcd).

---

## 2. What is it?

### The Anatomy of a Race Condition

#### Pattern 1: Read-Modify-Write (Lost Update)
```
Initial State: counter = 10

Thread 1: Reads counter (10)
Thread 2: Reads counter (10)
Thread 1: Computes 10 + 1 = 11, writes counter = 11
Thread 2: Computes 10 + 1 = 11, writes counter = 11 (Overwrites Thread 1!)

Expected: 12. Actual: 11 (One increment is lost).
```

#### Pattern 2: Check-Then-Act (Time-Of-Check to Time-Of-Use)
```
Initial State: stock = 1 (Only 1 item left in warehouse)

Thread 1: Checks: is stock (1) >= 1? -> TRUE
Thread 2: Checks: is stock (1) >= 1? -> TRUE
Thread 1: Deducts 1 -> stock becomes 0 -> Fulfills Order 101
Thread 2: Deducts 1 -> stock becomes -1 -> Fulfills Order 102 (OVERSELLING BUG!)
```

---

## 3. Why do we need it? The Distributed Scale Problem

In monolithic single-process applications, language-level primitives (Java `synchronized`, Go `sync.Mutex`, C++ `std::mutex`) can protect shared memory. However, in modern cloud architectures:
- Applications run across **50 horizontally scaled container pods**.
- In-memory language mutexes only lock within a single container process.
- Concurrency must be controlled at the **shared persistence tier** (Relational Database, Distributed Lock Manager).

---

## 4. How does it work internally? The 4 Defense Levels

```
Hierarchy of Solutions (From Simplest to Most Complex):
1. Atomic SQL Statement (Preferred): Single atomic update in database engine
2. Pessimistic Row Lock (SELECT FOR UPDATE): Serializes transactions on specific row
3. Optimistic Concurrency Control (Version check): Detects collision on commit
4. Distributed Lock (Redis SET NX / Redlock): Synchronizes across multiple systems
```

### 1. Atomic SQL Update (The Gold Standard)
Whenever possible, collapse the Read-Modify-Write cycle into a single atomic database statement that uses the database's internal row locks:

```sql
-- Atomic Decrement with Invariant Guard
UPDATE inventory 
SET stock = stock - 1 
WHERE product_id = 99 AND stock >= 1;
```
If `rows_affected == 1`, the purchase succeeded. If `rows_affected == 0`, the product is out of stock. Zero explicit transactions or distributed locks required!

### 2. Pessimistic Locking (`SELECT FOR UPDATE`)
Acquires an exclusive lock on the target row, forcing concurrent transactions to queue:

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 99 FOR UPDATE;
-- Application verifies business logic: if stock >= requested_quantity
UPDATE inventory SET stock = stock - requested_quantity WHERE product_id = 99;
COMMIT;
```

### 3. Optimistic Concurrency Control (OCC)
Uses a monotonically increasing `version` integer:

```sql
UPDATE accounts 
SET balance = 150.00, version = version + 1 
WHERE id = 42 AND version = 3;
```
If another transaction updated the row first, `version` is already 4, `rows_affected` returns 0, and the application catches the conflict and retries.

### 4. Distributed Locks (Redis / Redlock)
When a critical section spans multiple distinct systems (e.g., Calling Stripe API + Updating Warehouse DB + Sending Email):

```python
# Redis SET with NX and EX (Atomic)
SET lock:inventory:99 "worker_uuid_123" NX EX 10
```

---

## 5. Architecture / Flow

```mermaid
sequenceDiagram
    autonumber
    participant U1 as User 1 (API Pod A)
    participant DB as Inventory Database (stock = 1)
    participant U2 as User 2 (API Pod B)

    Note over DB: Initial State: Item 99 has stock = 1

    U1->>DB: BEGIN; SELECT stock FROM items WHERE id=99 FOR UPDATE;
    Note over DB: Lock Granted to User 1 (X-Lock on row 99)
    
    U2->>DB: BEGIN; SELECT stock FROM items WHERE id=99 FOR UPDATE;
    Note over DB: User 2 is BLOCKED! Waits for User 1 to finish...

    Note over U1: Validates stock (1 >= 1) -> OK!
    U1->>DB: UPDATE items SET stock = 0 WHERE id=99;
    U1->>DB: COMMIT; (Releases Lock)
    
    Note over DB: User 2 Unblocked! Evaluates query...
    DB-->>U2: Returns stock = 0
    Note over U2: Validates stock (0 >= 1) -> FALSE (Out of stock!)
    U2->>DB: ROLLBACK;
    U2-->>U2: Returns "Sold Out" error to User 2
```

---

## 6. Simple Example: Python Vulnerable vs Fixed Code

### Vulnerable Code (Check-Then-Act Race)
```python
# ANTI-PATTERN: DO NOT USE IN PRODUCTION
def purchase_ticket(user_id: int, event_id: int):
    event = db.query("SELECT available_tickets FROM events WHERE id = :id", id=event_id)
    
    if event.available_tickets > 0:
        # RACE WINDOW: Another thread can execute this exact block simultaneously!
        time.sleep(0.05) # Simulating processing delay
        new_tickets = event.available_tickets - 1
        db.execute("UPDATE events SET available_tickets = :t WHERE id = :id", t=new_tickets, id=event_id)
        return True
    return False
```

### Fixed Code (Atomic SQL Guard)
```python
def purchase_ticket_atomic(user_id: int, event_id: int):
    # Single atomic query with condition
    rows_updated = db.execute(
        """
        UPDATE events 
        SET available_tickets = available_tickets - 1 
        WHERE id = :id AND available_tickets > 0
        """,
        id=event_id
    )
    
    if rows_updated > 0:
        db.execute("INSERT INTO tickets (user_id, event_id) VALUES (:u, :e)", u=user_id, e=event_id)
        return True
    return False # Out of tickets!
```

---

## 7. Production Example: Digital Wallet Transfer (Pessimistic Locking + Deterministic Order)

```python
def transfer_wallet_balance(sender_id: str, receiver_id: str, amount: float):
    # 1. Enforce strict numerical lock ordering to prevent deadlocks
    first_id, second_id = sorted([sender_id, receiver_id])
    
    with db.transaction():
        # 2. Acquire Exclusive locks in deterministic order
        sender = db.execute("SELECT balance FROM wallets WHERE id = :id FOR UPDATE", id=first_id)
        receiver = db.execute("SELECT balance FROM wallets WHERE id = :id FOR UPDATE", id=second_id)
        
        # 3. Check business invariant
        if sender.balance < amount:
            raise InsufficientFundsException("Sender does not have required balance")
            
        # 4. Apply mutations safely
        db.execute("UPDATE wallets SET balance = balance - :a WHERE id = :id", a=amount, id=sender_id)
        db.execute("UPDATE wallets SET balance = balance + :a WHERE id = :id", a=amount, id=receiver_id)
```

---

## 8. When should we use which approach?

- **Atomic SQL Updates:** Use whenever the mutation is simple arithmetic or status change on a single table.
- **Pessimistic Locking (`FOR UPDATE`):** Use when business validation logic requires inspecting multiple rows/tables before mutating under high contention.
- **Optimistic Locking (`version`):** Use when read-to-write ratios are high, and write collisions are infrequent.
- **Distributed Locks (Redis/etcd):** Use when synchronizing actions across non-database external services (e.g., preventing duplicate batch file generation on S3).

---

## 9. When should we avoid distributed locks?

- Never use distributed locks when a simple SQL atomic update or database unique constraint achieves the exact same guarantee with lower latency and zero distributed state.

---

## 10. Tradeoffs

| Concurrency Mechanism | Performance | Safety / Reliability | Implementation Complexity |
| :--- | :--- | :--- | :--- |
| **Atomic SQL** | **Maximum** ($<1$ms) | **100% ACID** | Very Low |
| **Pessimistic Lock** | Moderate (Holds row locks) | **100% ACID** | Low |
| **Optimistic Lock** | High under low load; poor under high load | **100% ACID** (Requires retry loop) | Moderate |
| **Distributed Lock** | Moderate (Network hops to Redis) | Susceptible to TTL expiration bugs | High |

---

## 11. Common Mistakes

1. **The "Lock-Expired-Mid-Work" Bug (Distributed Locks):** Setting a Redis lock TTL to 5 seconds. The worker gets stuck in a 10-second GC pause or slow HTTP call. The lock expires. Another worker acquires the lock, causing concurrent execution. (Fix: Use a lock heartbeat / auto-extender thread, or fencing tokens).
2. **Missing Fencing Tokens:** Failing to pass a monotonically increasing fencing token to the storage layer, allowing a stale "zombie" worker to overwrite fresh data.
3. **Relying on In-Memory Thread Locks in Auto-Scaled Backends:** Using `threading.Lock()` or `sync.Mutex` in a service deployed to 10 Kubernetes pods.

---

## 12. Related Concepts

- [Locks & Deadlocks](file:///home/sameer/backendguide/05-databases/locks-deadlocks.md)
- [Isolation Levels & Write Skew](file:///home/sameer/backendguide/05-databases/isolation-levels.md)
- [Idempotency & Safe Retries](file:///home/sameer/backendguide/03-reliability/idempotency.md)

---

## 13. Interview Questions

### Q1. What is the difference between a "Read-Modify-Write" race condition and a "Check-Then-Act" race condition?
**Answer:**
- **Read-Modify-Write (Lost Update):** Two threads read state $X$, compute $X' = f(X)$ in memory, and write $X'$ back. The second thread's write overwrites and obliterates the first thread's mutation. (e.g., Two concurrent likes on an Instagram post incrementing 100 to 101 instead of 102).
- **Check-Then-Act (TOCTOU):** A thread checks a condition (e.g., `if balance >= 100`), finds it true, but before it acts, another thread mutates the state (e.g., withdraws the balance). The first thread acts on stale assumptions, violating invariants (e.g., balance drops to negative).  
**Why this matters:** Core mental model for concurrency analysis.  
**Possible follow-up:** How does an atomic SQL update prevent both patterns?

### Q2. Why is `UPDATE inventory SET stock = stock - 1 WHERE id = 1 AND stock >= 1` superior to using a distributed lock?
**Answer:**
1. **Single Point of Truth & ACID:** The database is already the canonical source of truth; its internal engine enforces row-level latching and WAL durability natively.
2. **Zero Network Overhead:** Eliminates 2–4 extra network round-trips to an external Redis cluster to acquire and release the lock.
3. **Immune to TTL Expiry:** Database locks are held for the exact duration of the statement; there is no arbitrary timeout that can expire prematurely during unexpected GC or network pauses.  
**Why this matters:** Tests pragmatic senior engineering judgment vs over-engineering.  
**Possible follow-up:** When is an atomic update insufficient?

### Q3. What is a "Fencing Token" in distributed systems and why is it necessary?
**Answer:** When using distributed locks with TTLs (e.g., Redis or ZooKeeper), a client may experience a long Garbage Collection (GC) pause or network partition after acquiring Lock $N$. If the TTL expires while the client is paused, a second client acquires Lock $N+1$. When the first client wakes up, it falsely believes it still holds the lock and writes to storage.  
**A Fencing Token** is a monotonically increasing integer issued with every lock grant. The storage service (database) records the highest token seen and rejects any write presenting an older token ($N < N+1$), effectively blocking the zombie client.  
**Why this matters:** Famous distributed systems problem highlighted by Martin Kleppmann in the Redlock analysis.  
**Possible follow-up:** How does etcd implement fencing via revision numbers?

### Q4. How do you implement a distributed lock safely using Redis?
**Answer:**
1. Acquire using atomic `SET key uuid NX PX 10000` (sets key only if not exists, with 10s auto-expiry).
2. Store a unique random UUID as the value.
3. Execute the critical section.
4. Release using an **atomic Lua script** that verifies the key's value matches the client's UUID before calling `DEL`. This prevents a slow client from deleting a lock acquired by a newer client after its own TTL expired.  
**Why this matters:** Standard interview question for distributed caching and concurrency.  
**Possible follow-up:** What are the limitations of the Redlock multi-node algorithm?

### Q5. How does Optimistic Concurrency Control (OCC) handle high write contention?
**Answer:** Under low contention, OCC is extremely fast because it acquires zero locks. However, under **high write contention** (e.g., 10,000 users buying 5 concert tickets), almost all transactions fail the `WHERE version = ?` check and are forced to retry in tight loops. This causes severe CPU waste, database connection churn, and thread starvation. Under high contention, **Pessimistic Locking** (`SELECT FOR UPDATE`) or an **Asynchronous Request Queue** is vastly superior.  
**Why this matters:** Critical tradeoff analysis in high-throughput API architecture.  
**Possible follow-up:** How do you implement exponential backoff with jitter on OCC retries?

---

## 14. Advanced Interview Questions

### Q6. How do database engines implement Compare-And-Swap (CAS) at the CPU hardware level?
**Answer:** Modern CPUs provide atomic hardware instructions such as `CMPXCHG` (x86) or `LL/SC` (Load-Link / Store-Conditional on ARM). When a database engine or runtime executes an atomic operation (e.g., Go `sync/atomic`), the CPU locks the memory cache line (via cache coherence protocols like MESI) to ensure no other CPU core can read or modify that memory address during the read-modify-write cycle.

---

## 15. Production Scenarios

### Scenario: Double-Redemption of Promo Coupons During Marketing Campaign
**Problem:** Users discovered that by clicking "Apply Promo Code" rapidly in two browser tabs simultaneously, they received two $50 gift card credits for a single one-time code.
**Analysis:** The API used `if not promo.is_used: promo.is_used = True`. Concurrent threads evaluated the check simultaneously.
**Fix:** Added a database unique constraint `UNIQUE (user_id, promo_code_id)` on the `applied_promotions` table. The second concurrent insert triggers a database unique constraint violation and fails cleanly.

---

## 16. Debugging Scenarios

### Scenario: Detecting Concurrency Bugs via Stress Testing
Use load testing tools (e.g., `k6` or `Locust`) with 100 virtual users executing simultaneous identical checkout actions:
```javascript
// k6 script: simulate concurrent purchases of the same product
export default function () {
  http.post('http://localhost:8080/v1/orders', JSON.stringify({ product_id: 99 }));
}
```
Verify after the test that `SELECT stock FROM inventory WHERE product_id = 99` is strictly $\ge 0$ and `total_orders == initial_stock`.

---

## 17. Common Misconceptions

- *Misconception:* "Wrapping code in a database transaction automatically prevents race conditions."
  - *Reality:* Standard `READ COMMITTED` transactions do not prevent lost updates or check-then-act bugs unless you use explicit row locks (`FOR UPDATE`) or atomic SQL statements.
- *Misconception:* "A fast in-memory Redis check eliminates all race conditions."
  - *Reality:* If you execute `GET` followed by `SET`, the network round-trip window between the two commands is open to race conditions. Operations must be atomic (e.g., `INCR`, `SET NX`, or Lua scripts).

---

## 18. Quick Revision

- Race conditions occur when concurrent execution produces non-deterministic shared state.
- Always prefer **Atomic SQL Statements** (`SET stock = stock - 1 WHERE stock >= 1`).
- Use **Pessimistic Locking** (`SELECT FOR UPDATE`) for high contention.
- Use **Optimistic Locking** (`WHERE version = ?`) for low contention.
- Distributed locks require unique tokens, Lua release scripts, and fencing tokens.

---

## 19. Interview-Ready Answer

> "A race condition occurs when concurrent processes access shared mutable state without proper synchronization, causing lost updates or check-then-act bugs. In horizontally scaled backend systems, in-memory mutexes are insufficient because state is shared across multiple server instances. We eliminate race conditions primarily at the database layer using atomic SQL update statements with invariant guards, pessimistic row-level locking with SELECT FOR UPDATE for multi-step transactions, or optimistic concurrency control using version tokens. For coordinating across external third-party services, we utilize distributed locks in Redis backed by atomic Lua scripts and fencing tokens."
