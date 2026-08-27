# Backend Interview Questions: Rapid-Fire Revision

A fast-paced, high-yield cheat sheet of 40 essential backend questions and punchy answers for pre-interview revision.

---

### Networking & APIs
1. **HTTP vs HTTPS default ports?**  
   Port 80 (HTTP) vs Port 443 (HTTPS).
2. **What are the three pillars of HTTPS security?**  
   Confidentiality (encryption), Integrity (tamper detection), Authentication (certificates).
3. **What is the difference between HTTP/1.1 and HTTP/2?**  
   HTTP/1.1 is text-based and serial; HTTP/2 is binary and multiplexed over a single TCP connection.
4. **What solves TCP Head-of-Line blocking in HTTP/3?**  
   QUIC running over UDP.
5. **What is the difference between PUT and PATCH?**  
   `PUT` is full replacement (idempotent); `PATCH` is partial update (non-idempotent by default).
6. **What is the mathematical definition of Idempotency?**  
   $f(f(x)) = f(x)$ — executing multiple times yields identical side-effects as once.
7. **What is HSTS?**  
   `Strict-Transport-Security` header forcing browsers to use HTTPS exclusively, preventing SSL stripping.
8. **What is SNI?**  
   Server Name Indication; sends target hostname in `ClientHello` so a server can present the correct SSL certificate.
9. **Difference between 401 and 403?**  
   `401 Unauthorized` = Unauthenticated (missing/invalid token); `403 Forbidden` = Unauthorized (lacks permissions).
10. **Difference between 502 and 504?**  
    `502 Bad Gateway` = Upstream returned invalid/refused response; `504 Gateway Timeout` = Upstream took too long to reply.

---

### Security & Authentication
11. **Are standard JWTs encrypted?**  
    No. They are Base64URL-encoded and digitally signed (JWS), not encrypted (JWE).
12. **Why use RS256 over HS256 in microservices?**  
    RS256 is asymmetric (Auth service signs with Private Key; services verify with Public Key without risk of token forgery).
13. **What does the `HttpOnly` cookie flag prevent?**  
    JavaScript access to cookies, mitigating token theft via Cross-Site Scripting (XSS).
14. **What does `SameSite=Lax` prevent?**  
    Cross-Site Request Forgery (CSRF) on cross-origin POST requests.
15. **What is Refresh Token Rotation?**  
    Invalidating and replacing the refresh token upon every use to detect and contain token theft.
16. **Why store passwords with bcrypt/Argon2 instead of SHA256?**  
    Passwords have low entropy; bcrypt/Argon2 are intentionally slow and memory-hard to resist GPU brute-forcing.

---

### Databases & Transactions
17. **What data structure is standard for relational database indexes?**  
    B+ Tree (high fan-out, shallow depth, doubly linked leaves for range scans).
18. **Difference between Clustered and Secondary index in MySQL InnoDB?**  
    Clustered index leaf contains actual table row data (Primary Key); secondary index leaf contains the Primary Key value.
19. **What is the Leftmost-Prefix rule?**  
    A composite index on `(A, B, C)` can only be searched if queries filter by leading columns starting with `A`.
20. **What is a Covering Index?**  
    An index containing all columns requested by a query, enabling an Index-Only scan without reading table heap pages.
21. **What is Write-Ahead Logging (WAL)?**  
    Appending sequential mutation records to disk before modifying in-memory data pages to ensure Durability.
22. **What does ACID stand for?**  
    Atomicity (all-or-nothing), Consistency (rules hold), Isolation (concurrency control), Durability (crash permanence).
23. **What is a Dirty Read?**  
    Reading uncommitted changes made by another transaction that later rolls back.
24. **What is a Non-Repeatable Read?**  
    Re-reading a row and finding its values modified by another committed transaction.
25. **What is a Phantom Read?**  
    Re-running a range query and discovering new rows inserted by another committed transaction.
26. **What is Write Skew?**  
    Concurrent transactions reading overlapping data and writing non-conflicting changes that collectively violate an invariant.
27. **What is the default isolation level in PostgreSQL?**  
    Read Committed.
28. **What is the default isolation level in MySQL InnoDB?**  
    Repeatable Read.
29. **How do you prevent deadlocks during transfers between accounts?**  
    Sort IDs and lock rows in deterministic ascending numerical order.
30. **What is the N+1 Query Problem?**  
    1 parent query + $N$ lazy child queries in a loop. Fixed via Eager Loading (`JOIN` or batched `IN`).

---

### Caching & Messaging
31. **What is the Cache-Aside pattern?**  
    App reads cache; on miss reads DB & writes cache. On DB update, app deletes (invalidates) cache key.
32. **What is a Cache Stampede?**  
    A hot key expires, causing thousands of concurrent requests to miss and hammer the database simultaneously.
33. **What is Cache Penetration?**  
    Querying non-existent keys that miss cache and hit DB on every request. Fixed with Bloom Filters or caching nulls.
34. **What is Cache Avalanche?**  
    Mass expiration of keys at the exact same second. Fixed with randomized TTL Jitter.
35. **How does Kafka achieve strict message ordering?**  
    Ordering is strictly guaranteed *only within a single partition* using a consistent message key.
36. **Maximum parallel consumers in a Kafka consumer group?**  
    Equal to the number of partitions in the subscribed topic.

---

### Scalability & Infrastructure
37. **Difference between Layer 4 and Layer 7 load balancing?**  
    Layer 4 routes raw TCP/UDP packets without payload inspection; Layer 7 terminates TLS and routes based on HTTP paths/headers.
38. **Why are Sticky Sessions an anti-pattern?**  
    Causes load hotspotting, breaks auto-scaling, and causes user data loss during node failover.
39. **How do Docker containers achieve process isolation?**  
    Linux Namespaces (what processes can see) and cgroups (what resources processes can use).
40. **What is the RED method in observability?**  
    Rate (req/s), Errors (5xx/s), Duration (latency distribution / percentiles).
