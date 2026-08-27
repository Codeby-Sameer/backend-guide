# Database & SQL Cheatsheet

A deep technical cheat sheet covering indexing, query optimization, PostgreSQL and MySQL engine differences, transactions, and locking.

---

## 1. Index Types & Engine Differences

| Index Type | PostgreSQL | MySQL (InnoDB) | Best Use Case |
| :--- | :--- | :--- | :--- |
| **B+ Tree** | Default index type (`CREATE INDEX`) | Default index type | General equality (`=`) and range scans (`<`, `>`, `BETWEEN`) |
| **Clustered Index** | Not native (Heap table model) | **Primary Key** is clustered index | Primary key lookups |
| **Covering Index** | Supported via `INCLUDE (col)` | Composite index covering columns | Index-Only Scans |
| **Partial Index** | `WHERE condition` | Generated columns only | Indexing active subset (e.g. `WHERE status = 'ACTIVE'`) |
| **GIN / GiST** | Generalized Inverted Index | Limited / FULLTEXT | JSONB fields, Arrays, Full-Text, PostGIS |
| **Hash Index** | Memory/WAL logged $O(1)$ | Memory engine only | Exact equality only (No range scans) |

---

## 2. EXPLAIN / Query Plan Nodes Cheat Sheet

```
+──────────────────────────┬───────────────────────────────────────────────────────────+
│ Plan Node                │ Performance Interpretation                                │
+──────────────────────────┼───────────────────────────────────────────────────────────+
│ **Seq Scan / Table Scan**│ Scans every disk page in table (Slow on large tables).     │
│ **Index Scan**           │ Traverses B-Tree, seeks matching rows, reads heap pages.  │
│ **Index Only Scan**      │ Satisfies query entirely from index leaf (Fastest!).      │
│ **Bitmap Index Scan**    │ Merges multiple index hits into bitmap before heap fetch. │
│ **Using filesort**       │ In-memory quicksort or disk merge sort (Missing index).   │
│ **Using temporary**      │ Created temporary table on disk/RAM to group/sort.        │
+──────────────────────────┴───────────────────────────────────────────────────────────+
```

---

## 3. SQL Query Optimization Rules

```sql
-- 1. Avoid functions on indexed columns in WHERE (Breaks Index Seek)
-- BAD:
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2026;
-- GOOD:
SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';

-- 2. Use Keyset/Cursor Pagination instead of OFFSET
-- BAD (O(N) Scans):
SELECT * FROM logs ORDER BY id LIMIT 20 OFFSET 1000000;
-- GOOD (O(log N) Seek):
SELECT * FROM logs WHERE id > 1000000 ORDER BY id ASC LIMIT 20;

-- 3. Composite Index Leftmost Prefix Rule
-- Index on (tenant_id, status, created_at DESC)
-- Perfect for:
SELECT * FROM orders 
WHERE tenant_id = 5 AND status = 'COMPLETED' 
ORDER BY created_at DESC LIMIT 20;
```

---

## 4. ANSI SQL Isolation Levels & Anomalies

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | **Allowed** | **Allowed** | **Allowed** | **Allowed** |
| **Read Committed** (Postgres default) | *Prevented* | **Allowed** | **Allowed** | **Allowed** |
| **Repeatable Read** (MySQL default) | *Prevented* | *Prevented* | *Prevented (InnoDB)* | **Allowed** |
| **Serializable** | *Prevented* | *Prevented* | *Prevented* | *Prevented* |

---

## 5. PostgreSQL Diagnostic Queries

```sql
-- Active long-running queries
SELECT pid, now() - query_start AS duration, query, state 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY duration DESC;

-- Table bloat & dead tuples
SELECT relname, n_live_tup, n_dead_tup, 
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0),2) as dead_pct
FROM pg_stat_user_tables 
ORDER BY n_dead_tup DESC;

-- Missing index detection (high seq scan vs idx scan ratio)
SELECT relname, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch
FROM pg_stat_user_tables 
WHERE seq_scan > 1000 
ORDER BY seq_tup_read DESC;
```
