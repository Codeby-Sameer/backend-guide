# Backend Interview Questions: Beginner Tier

A curated collection of foundational backend engineering interview questions designed to test core mental models across networking, APIs, databases, caching, and infrastructure.

---

### Q1. What happens from a networking and backend perspective when you type a URL into a browser and press Enter?
**Answer:** 
1. **DNS Lookup:** Browser checks local DNS cache, OS hosts file, and queries recursive DNS servers to resolve the domain (e.g., `api.example.com`) to an IP address.
2. **TCP 3-Way Handshake:** Client initiates a TCP connection to the server IP on port 80 (HTTP) or 443 (HTTPS) via `SYN` $\to$ `SYN-ACK` $\to$ `ACK`.
3. **TLS Handshake (HTTPS):** Client and server negotiate TLS version, cipher suites, verify the server's X.509 certificate, and derive symmetric session keys.
4. **HTTP Request:** Browser sends an HTTP request (`GET / HTTP/1.1`) with headers (`Host`, `User-Agent`, `Accept`).
5. **Server Processing:** The reverse proxy/load balancer terminates TLS, routes the request to an application server, which executes business logic, queries databases/caches, and renders a response.
6. **HTTP Response & Rendering:** Server returns HTTP response (`200 OK` + HTML/JSON payload). The browser renders the DOM and downloads secondary assets.  
**Example:** Visiting `https://github.com` executes DNS $\to$ TCP (port 443) $\to$ TLS 1.3 $\to$ HTTP/2 GET.  
**Why this matters:** Universal diagnostic question testing end-to-end understanding of internet infrastructure.  
**Possible follow-up:** How does DNS caching at different layers affect failover time?

---

### Q2. What is the difference between HTTP GET and HTTP POST?
**Answer:** 
- **`GET`:** Used to retrieve data from a server. It is **Safe** (does not mutate server state) and **Idempotent** (calling it 10 times produces the same system state). Parameters are passed in the URL query string. It is heavily cached by browsers, CDNs, and proxies.
- **`POST`:** Used to submit data to the server to create a resource or trigger a state change. It is **Not Safe** and **Not Idempotent** (calling it 10 times creates 10 resources). Data is transmitted inside the HTTP request body. It is non-cacheable by default.  
**Example:** `GET /users/42` views Alice's profile. `POST /users` creates a new user profile.  
**Why this matters:** Fundamental RESTful API protocol contract.  
**Possible follow-up:** Is a GET request allowed to have an HTTP request body?

---

### Q3. What is the difference between a Relational (SQL) and a Non-Relational (NoSQL) database?
**Answer:**
- **SQL (Relational):** Data is organized into structured tables with predefined schemas and strict relationships enforced by foreign keys. Follows ACID transactions. Excels at complex queries, joins, and financial/transactional integrity (e.g., PostgreSQL, MySQL).
- **NoSQL (Non-Relational):** Data is organized into flexible models (Document, Key-Value, Columnar, or Graph) with dynamic or no schemas. Typically prioritizes horizontal scalability and eventual consistency (e.g., MongoDB, Redis, Cassandra, DynamoDB).  
**Example:** PostgreSQL is preferred for banking transactions; MongoDB is preferred for catalogs with rapidly evolving JSON schemas.  
**Why this matters:** Choosing the correct persistence model during system design.  
**Possible follow-up:** When should you choose PostgreSQL over MongoDB for JSON data?

---

### Q4. What is a Database Primary Key vs a Foreign Key?
**Answer:**
- **Primary Key (PK):** A column (or set of columns) that uniquely identifies each row in a database table. It cannot contain `NULL` values and is automatically indexed by the database engine (often as a clustered index).
- **Foreign Key (FK):** A column in one table that references the Primary Key of another table, establishing a referential relationship and enforcing database integrity constraints (preventing orphan records).  
**Example:** In an `orders` table, `order_id` is the Primary Key, and `user_id` is a Foreign Key referencing `users(id)`.  
**Why this matters:** Core relational database schema normalization.  
**Possible follow-up:** What is the performance impact of missing indexes on Foreign Key columns?

---

### Q5. What is the difference between Authentication and Authorization?
**Answer:**
- **Authentication (AuthN):** Verifying the identity of the user ("Who are you?"). Verified using passwords, biometric passkeys, or digital certificates. Returns `401 Unauthorized` if invalid.
- **Authorization (AuthZ):** Determining what permissions and actions an authenticated user is permitted to perform ("What are you allowed to do?"). Evaluated using RBAC, ABAC, or scopes. Returns `403 Forbidden` if denied.  
**Example:** Logging in with your Google account is Authentication. Being prevented from viewing the Google Cloud Admin Billing console is Authorization.  
**Why this matters:** Foundational application security boundary.  
**Possible follow-up:** What is the difference between 401 Unauthorized and 403 Forbidden status codes?

---

### Q6. What is a Session Cookie and what do `HttpOnly` and `Secure` flags do?
**Answer:** A session cookie is a small piece of data stored in the browser by the server to maintain state across stateless HTTP requests.
- **`HttpOnly`:** Prevents client-side JavaScript (`document.cookie`) from reading the cookie, protecting against Cross-Site Scripting (XSS) token theft.
- **`Secure`:** Instructs the browser to transmit the cookie exclusively over encrypted HTTPS (TLS) connections, preventing plaintext sniffing.
- **`SameSite=Lax/Strict`:** Restricts cross-origin cookie transmission, protecting against CSRF attacks.  
**Example:** `Set-Cookie: session_id=xyz123; HttpOnly; Secure; SameSite=Lax; Max-Age=86400`  
**Why this matters:** Baseline security configuration for any cookie-based web application.  
**Possible follow-up:** How does `SameSite=Strict` differ from `SameSite=Lax`?

---

### Q7. What is Caching and why do we use it in backend systems?
**Answer:** Caching is the technique of storing copies of data in high-speed volatile memory (RAM, such as Redis or Memcached) so subsequent requests are served in microseconds without querying slower disks or recomputing complex algorithms. It reduces database CPU load, eliminates network latency, and prevents system crashes during high traffic spikes.  
**Example:** Caching product catalog JSON in Redis for 1 hour reduces database queries by 95%.  
**Why this matters:** Essential performance optimization pattern.  
**Possible follow-up:** What happens when data in the database changes while the cache still holds the old data?

---

### Q8. What is the purpose of a Load Balancer?
**Answer:** A load balancer sits between incoming client traffic and a pool of backend application servers. It distributes requests evenly across servers (using algorithms like Round Robin or Least Connections) to prevent any single server from becoming overwhelmed, monitors server health, and automatically routes around failed nodes to ensure high availability.  
**Example:** An AWS Application Load Balancer distributes 10,000 requests/second across 5 EC2 instances.  
**Why this matters:** Enables horizontal scaling and fault tolerance.  
**Possible follow-up:** What happens if the load balancer itself crashes?

---

### Q9. What is a Docker Container and how does it differ from a standard application process?
**Answer:** A Docker container is an isolated Linux process running with its own dedicated filesystem, network interface, and process tree, packaged with all required runtime libraries and system binaries. Unlike a standard process, a container's visibility and resource access are strictly isolated by Linux kernel **Namespaces** and **cgroups**.  
**Example:** Running `docker run -p 8080:8080 my-api` launches an isolated container running Node.js without needing Node installed on the host machine.  
**Why this matters:** Eliminates environmental drift across dev, staging, and production.  
**Possible follow-up:** Do containers have their own operating system kernel?

---

### Q10. What are HTTP Status Code classes (2xx, 3xx, 4xx, 5xx)?
**Answer:**
- **2xx (Success):** Request was received, understood, and accepted (`200 OK`, `201 Created`, `204 No Content`).
- **3xx (Redirection):** Client must take additional action to complete the request (`301 Moved Permanently`, `304 Not Modified`).
- **4xx (Client Error):** The client made an invalid request (`400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`).
- **5xx (Server Error):** The server failed to fulfill an apparently valid request due to internal failure (`500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`).  
**Why this matters:** Crucial for API predictability, client error handling, and proxy caching.  
**Possible follow-up:** What is the difference between a 502 Bad Gateway and a 504 Gateway Timeout?
