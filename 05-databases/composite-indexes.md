# Composite Indexes & Column Ordering

## 1. One-minute explanation

A **Composite Index** (also known as a Compound, Multi-Column, or Concatenated Index) is a single B+ Tree index built over two or more columns in a database table. The order of columns in the index definition is critical due to the **Leftmost-Prefix Principle**: the index is sorted primarily by the first column, secondarily by the second column, and so on. An index on `(customer_id, created_at)` can efficiently accelerate queries filtering on `customer_id` alone, or `customer_id AND created_at`, but is **completely useless** for queries filtering solely on `created_at`. Creating composite indexes with the optimal column order enables fast equality seeks, range scans, and eliminates expensive memory-intensive sorting operations (`ORDER BY`).

---

## 2. What is it?

A composite index stores keys in lexicographical order, similar to a physical telephone directory sorted by `(Last_Name, First_Name)`.

```
Telephone Directory Analogy:
(Smith, Alice)
(Smith, Bob)
(Smith, Charlie)
(Taylor, David)
(Williams, Zoe)
```
- Can you quickly find all people named **"Smith"**? **Yes** (Leading column matched).
- Can you quickly find **"Smith, Bob"**? **Yes** (Both columns matched).
- Can you quickly find all people with first name **"Bob"**? **No!** You must scan the entire directory from start to finish because "Bob" is scattered under Smith, Taylor, Williams, etc.

---

## 3. Why do we need it?

Single-column indexes are often inadequate for real-world backend queries that filter across multiple dimensions simultaneously.

```sql
SELECT * FROM orders 
WHERE tenant_id = 'org_123' 
  AND status = 'SHIPPED' 
  AND created_at >= '2026-01-01'
ORDER BY created_at DESC 
LIMIT 20;
```

If you create three separate single-column indexes on `tenant_id`, `status`, and `created_at`:
1. The database query engine must choose one index, perform a scan, and filter the remaining conditions in memory against heap rows.
2. Or it must perform an expensive **Bitmap Index Scan** (merging bitmasks from multiple indexes).
3. A single well-crafted composite index on `(tenant_id, status, created_at DESC)` satisfies the entire `WHERE` clause and `ORDER BY` in a single index seek with zero post-sort overhead.

---

## 4. How does it work internally?

### 1. Leftmost-Prefix Principle
For an index defined on columns `(A, B, C)`:
- `WHERE A = 1` ──► **Uses Index (A)**
- `WHERE A = 1 AND B = 2` ──► **Uses Index (A, B)**
- `WHERE A = 1 AND B = 2 AND C = 3` ──► **Uses Index (A, B, C)**
- `WHERE A = 1 AND C = 3` ──► **Uses Index (A only)**; evaluates C as a filter.
- `WHERE B = 2 AND C = 3` ──► **Cannot Seek Index** (Leading column A is missing; forces table scan or full index scan).
- `WHERE C = 3` ──► **Cannot Seek Index**.

### 2. The Equality First, Range Second Rule
In a B+ Tree, index seek evaluation stops at the **first range predicate** (`<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`).

```
Rule: [ Equality Columns ] ──► [ Range Column ] ──► [ Remaining Columns (Filter Only) ]
```

For index `(status, created_at, amount)`:
- Query: `WHERE status = 'PAID' AND created_at > '2026-01-01' AND amount > 500`
  1. `status = 'PAID'` is an exact seek.
  2. `created_at > '2026-01-01'` is a range scan along the leaf chain.
  3. `amount > 500` **cannot** be used for tree seeking! It can only be evaluated as an index filter because the leaf entries are no longer sorted by `amount` across different timestamps.

---

## 5. Architecture / Flow

```mermaid
graph TD
    subgraph Composite Index on (customer_id, created_at)
        Entry1["Key: (101, 2026-01-01 10:00) ──► Row 1"]
        Entry2["Key: (101, 2026-01-05 14:00) ──► Row 2"]
        Entry3["Key: (101, 2026-01-10 09:00) ──► Row 3"]
        Entry4["Key: (102, 2026-01-02 11:00) ──► Row 4"]
        Entry5["Key: (102, 2026-01-08 16:00) ──► Row 5"]
        Entry6["Key: (103, 2026-01-03 12:00) ──► Row 6"]

        Entry1 --> Entry2 --> Entry3 --> Entry4 --> Entry5 --> Entry6
    end

    subgraph Query Usability
        Q1["WHERE customer_id = 101 AND created_at > '2026-01-02'"] -->|Fast Index Seek & Range Scan| Matched1["Seeks directly to (101, 2026-01-05), scans to (101, 2026-01-10)"]
        Q2["WHERE customer_id = 101"] -->|Fast Index Seek| Matched2["Seeks all 101 records efficiently"]
        Q3["WHERE created_at = '2026-01-05'"] -->|Full Scan Required| Matched3["Cannot seek! customer_id 101 is scattered"]
    end
```

---

## 6. Simple Example: Demonstrating Column Order Impact

```sql
-- Create table
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    org_id INT NOT NULL,
    action_type VARCHAR(64) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Case A: Index on (org_id, created_at)
CREATE INDEX idx_audit_org_date ON audit_logs (org_id, created_at);

-- Query 1: Filter by org and sort by date (Ultra Fast - No Sort Node)
EXPLAIN ANALYZE 
SELECT * FROM audit_logs 
WHERE org_id = 50 
ORDER BY created_at DESC 
LIMIT 10;
-- Execution Plan: Index Scan Backward using idx_audit_org_date (Zero sorting in memory!)

-- Query 2: Filter solely by date (Cannot use index seek)
EXPLAIN ANALYZE 
SELECT * FROM audit_logs 
WHERE created_at > '2026-01-01';
-- Execution Plan: Seq Scan on audit_logs (Full Table Scan)
```

---

## 7. Production Example: Optimizing E-Commerce User Orders Query

```sql
-- Production query from dashboard API
SELECT order_id, total_price, status, created_at 
FROM orders 
WHERE user_id = 94812 
  AND status IN ('COMPLETED', 'PROCESSING')
ORDER BY created_at DESC 
LIMIT 20;
```

### Optimal Composite Index Design
```sql
CREATE INDEX idx_orders_user_status_created 
ON orders (user_id, status, created_at DESC) 
INCLUDE (total_price, order_id);
```

**Why this index is optimal:**
1. `user_id` (Equality): Narrows search to a single customer.
2. `status` (IN list / Equality): Narrows search to specific statuses.
3. `created_at DESC` (Order): Matches `ORDER BY created_at DESC`, completely avoiding PostgreSQL `Sort` or MySQL `Using filesort`.
4. `INCLUDE (total_price, order_id)`: Provides a **Covering Index**, avoiding heap lookups.

---

## 8. When should we use it?

- Queries with multiple `WHERE` conditions connected by `AND`.
- Queries combining `WHERE` equality filters with `ORDER BY` sorting.
- Multi-tenant architectures where every query filters by `tenant_id` or `org_id` as the leading column.
- Enforcing composite uniqueness constraints (e.g., `UNIQUE(company_id, employee_email)`).

---

## 9. When should we avoid it?

- **Separate, Independent Filter Queries:** If Query 1 filters solely by `A` and Query 2 filters solely by `B`, an index on `(A, B)` will never help Query 2.
- **Excessively Wide Indexes:** Indexes with 8+ columns consume massive disk space and degrade `INSERT` performance.

---

## 10. Tradeoffs: Index Consolidation

| Pattern | Write Overhead | Storage | Query Flexibility |
| :--- | :--- | :--- | :--- |
| **Two Single Indexes: (A), (B)** | High (2 trees updated per write) | High | Helps `WHERE A` and `WHERE B`, but suboptimal for `WHERE A AND B`. |
| **One Composite Index: (A, B)** | Low (1 tree updated per write) | Low | Optimizes `WHERE A AND B` and `WHERE A`, but **useless** for `WHERE B`. |
| **All Permutations: (A, B), (B, A)** | Very High (2 wide trees) | Very High | Optimizes all combinations, but severe write amplification. |

---

## 11. Common Mistakes

1. **Incorrect Column Order (Range Before Equality):** Defining `(created_at, customer_id)` instead of `(customer_id, created_at)`.
2. **Redundant Index Duplication:** Creating an index on `(A)` when an index on `(A, B, C)` already exists. The single-column index on `(A)` is redundant and should be dropped.
3. **Mismatched Sort Order:** Index defined on `(A ASC, B ASC)` querying `ORDER BY A ASC, B DESC` (cannot leverage index order for forward/backward scan).

---

## 12. Related Concepts

- [Database Indexes Fundamentals](file:///home/sameer/backendguide/05-databases/indexes.md)
- [Locks & Deadlocks](file:///home/sameer/backendguide/05-databases/locks-deadlocks.md)
- [N+1 Query Problem](file:///home/sameer/backendguide/05-databases/n-plus-one.md)

---

## 13. Interview Questions

### Q1. Why is an index on `(customer_id, created_at)` fundamentally different from an index on `(created_at, customer_id)`?
**Answer:** In a B+ Tree, keys are sorted lexicographically by the first column, and ties in the first column are sorted by the second.
- `(customer_id, created_at)` sorts all entries by `customer_id` first. It allows the database to instantly seek customer 100 in $O(\log N)$ and read their chronological orders.
- `(created_at, customer_id)` sorts all entries across the entire database by timestamp first. Finding customer 100 requires scanning the entire index because customer 100's orders are scattered across every date.  
**Example:** Querying "Show me Alice's orders from yesterday" takes 1ms on `(customer_id, created_at)` and 4 seconds on `(created_at, customer_id)`.  
**Why this matters:** Core rule of database index design.  
**Possible follow-up:** When would `(created_at, customer_id)` be the right index?

### Q2. What is the Leftmost-Prefix Rule, and which of the following queries can use an index on `(A, B, C)`?
**Queries:**
1. `WHERE A = 1 AND B = 2` ──► **YES (Uses A, B seek)**
2. `WHERE B = 2 AND C = 3` ──► **NO (Cannot seek; leading column A missing)**
3. `WHERE A = 1 AND C = 3` ──► **PARTIAL (Seeks A; filters C)**
4. `WHERE C = 3 AND B = 2 AND A = 1` ──► **YES (SQL optimizer reorders conditions to A, B, C)**
5. `WHERE A > 10 AND B = 2` ──► **PARTIAL (Range scan on A; B cannot be used for seek)**  
**Why this matters:** Standard interview query optimization test.  
**Possible follow-up:** How does PostgreSQL Index Skip Scan / Loose Index Scan work?

### Q3. If you have an existing index on `(user_id, status)`, do you also need a separate index on `(user_id)`?
**Answer:** **No.** The composite index `(user_id, status)` already provides an efficient index seek for any query filtering solely by `user_id` because `user_id` is the leading leftmost prefix. Keeping a separate index on `(user_id)` wastes RAM and slows down write operations without providing query performance benefit.  
**Why this matters:** Identifying and dropping redundant indexes during database maintenance.  
**Possible follow-up:** Is the same true for a query filtering solely on `(status)`?

### Q4. How does a composite index eliminate `Using filesort` / `Sort` nodes in execution plans?
**Answer:** A B+ Tree maintains its leaf nodes in strictly pre-sorted order. When a query contains `WHERE user_id = ? ORDER BY created_at DESC`, an index on `(user_id, created_at DESC)` allows the database engine to seek to `user_id` and read the leaf nodes sequentially in reverse direction. Because rows arrive pre-sorted from the storage engine, the database completely bypasses in-memory quicksort or disk-based external merge sort operations.  
**Why this matters:** Sorting millions of rows in memory is a primary cause of CPU and memory saturation.  
**Possible follow-up:** What happens if the query sorts by `ORDER BY user_id DESC, created_at ASC`?

### Q5. What is the "Equality First, Range Second" heuristic in composite index design?
**Answer:** When designing composite indexes for multi-predicate queries:
1. Place columns used in **Exact Equality (`=`)** first.
2. Place columns used in **Range Filters (`<`, `>`, `BETWEEN`, `IN`)** or **`ORDER BY`** last.
Because a B+ tree can only traverse a single range path, any columns placed after a range column cannot participate in index tree seeks.  
**Example:** For `WHERE tenant_id = 5 AND age > 21 AND country = 'US'`, the index should be `(tenant_id, country, age)`, NOT `(tenant_id, age, country)`.  
**Why this matters:** Golden rule for composite index construction.  
**Possible follow-up:** How do `IN` lists behave compared to range operators?

---

## 14. Advanced Interview Questions

### Q6. How do modern databases perform "Index Skip Scans" (Loose Index Scans)?
**Answer:** If an index exists on `(gender, registration_date)` and a query asks `WHERE registration_date = '2026-08-01'` (omitting `gender`), standard leftmost prefix rules dictate a full scan. An **Index Skip Scan** optimizes this by recognizing that `gender` has very low cardinality (e.g., 'M', 'F'). The engine performs two distinct seeks: `(gender='M', registration_date='2026-08-01')` and `(gender='F', registration_date='2026-08-01')`, skipping all intermediate index pages.

---

## 15. Production Scenarios

### Scenario: High Latency on Multi-Tenant Activity Feed
**Problem:** In a multi-tenant SaaS application, loading the tenant activity feed `WHERE tenant_id = 100 ORDER BY created_at DESC LIMIT 50` takes 4.5 seconds.
**Analysis:** The table had two single-column indexes: `idx_tenant (tenant_id)` and `idx_created (created_at)`. The database picked `idx_tenant`, fetched 2,000,000 heap rows, and performed an external disk sort on `created_at`.
**Fix:** Added composite index `(tenant_id, created_at DESC)`. Query time dropped from 4,500ms to 0.8ms.

---

## 16. Debugging Scenarios

### Scenario: Diagnosing Sort Memory Spills (`Sort Method: external merge Disk`)
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM events 
WHERE user_id = 10 
ORDER BY timestamp DESC 
LIMIT 100;
```
If the plan shows:
`Sort (cost=... actual time=...) Sort Method: external merge Disk: 18432kB`
Create composite index `(user_id, timestamp DESC)` to eliminate the sort completely.

---

## 17. Common Misconceptions

- *Misconception:* "SQL WHERE clause order must match the composite index column order."
  - *Reality:* The SQL query optimizer reorders `AND` predicates automatically. `WHERE B = 2 AND A = 1` uses index `(A, B)` just as effectively as `WHERE A = 1 AND B = 2`.
- *Misconception:* "More columns in a composite index is always better."
  - *Reality:* Each additional column increases index tree size, consumes more buffer cache, and slows down data ingestion.

---

## 18. Quick Revision

- Leftmost-Prefix Rule: Leading columns must match for index seek.
- Place **Equality columns first**, **Range/Sort columns second**.
- An index on `(A, B)` makes a single-column index on `(A)` redundant.
- Eliminates expensive `ORDER BY` sorting operations.

---

## 19. Interview-Ready Answer

> "A composite index covers multiple columns ordered hierarchically according to the Leftmost-Prefix Principle. The index is sorted by the first column, with secondary columns acting as tie-breakers. To maximize performance, we place exact equality columns first, followed by range or sort columns. A properly ordered composite index enables logarithmic index seeks across multiple filters while completely eliminating CPU-intensive in-memory sort operations."
