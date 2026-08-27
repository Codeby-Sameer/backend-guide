# REST APIs & API Design

## 1. One-minute explanation

REST (Representational State Transfer) is an architectural style for designing networked applications centered around **resources** identified by URIs and manipulated using standard **HTTP verbs** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`). REST is fundamentally **stateless**, meaning each request from client to server must contain all the information necessary to understand and process the request. High-performance REST API design demands strict adherence to HTTP semantics (idempotency, safety, standard status codes), scalable pagination (cursor-based vs offset-based), structured error responses (RFC 7807), and predictable versioning strategies.

---

## 2. What is it?

Defined by Roy Fielding in 2000, REST leverages the existing architecture and semantics of the World Wide Web.

### The 6 Architectural Constraints of REST
1. **Client-Server Architecture:** Separation of user interface concerns from data storage and business logic.
2. **Statelessness:** No client context is stored on the server between requests. Session state is held entirely on the client.
3. **Cacheability:** Responses must implicitly or explicitly define themselves as cacheable or non-cacheable (`Cache-Control`, `ETag`).
4. **Uniform Interface:** Identification of resources (URIs), manipulation of resources through representations (JSON/XML), self-descriptive messages, and hypermedia (HATEOAS).
5. **Layered System:** The client cannot tell whether it is connected directly to the end server or an intermediary (CDN, proxy, load balancer).
6. **Code on Demand (Optional):** Servers can temporarily extend client functionality by transferring executable code (e.g., JavaScript).

---

## 3. Why do we need it?

Before REST, distributed APIs commonly used heavyweight, tightly coupled protocols like SOAP (XML-RPC over WSDL) or proprietary binary protocols. REST provided:
- **Loose Coupling:** Clients and servers can evolve independently without breaking contracts.
- **Interoperability:** Standard HTTP and JSON work universally across languages, platforms, and devices.
- **Native Web Caching:** Exploits HTTP caching proxies, CDNs, and browser caches natively.
- **Predictability:** Standardized verbs and status codes create an intuitive developer experience.

---

## 4. How does it work internally?

### 1. HTTP Verbs: Safety and Idempotency

| Method | Description | Safe? (No Side Effects) | Idempotent? ($f(f(x)) = f(x)$) |
| :--- | :--- | :--- | :--- |
| `GET` | Retrieve representation of a resource | **Yes** | **Yes** |
| `HEAD` | Retrieve headers only (no response body) | **Yes** | **Yes** |
| `OPTIONS` | Return supported HTTP methods/CORS | **Yes** | **Yes** |
| `POST` | Create a new resource or execute an action | **No** | **No** |
| `PUT` | Complete replacement of a target resource | **No** | **Yes** |
| `PATCH` | Partial update / modification of a resource | **No** | **Conditionally** |
| `DELETE` | Remove a resource | **No** | **Yes** |

### 2. PUT vs PATCH
- **`PUT /users/42`**: Replaces the entire resource. Any fields omitted in the payload are cleared or reset to defaults.
- **`PATCH /users/42`**: Modifies only the specified fields. Unmentioned fields remain untouched.

### 3. HTTP Status Codes Cheat Sheet

```
+--------------------------------------------------------------------------------+
| 2xx Success       | 200 OK, 201 Created, 202 Accepted, 204 No Content          |
+--------------------------------------------------------------------------------+
| 3xx Redirection   | 301 Moved Permanently, 304 Not Modified, 307/308 Redirect  |
+--------------------------------------------------------------------------------+
| 4xx Client Errors | 400 Bad Request, 401 Unauthorized, 403 Forbidden,          |
|                   | 404 Not Found, 409 Conflict, 422 Unprocessable, 429 Too Many|
+--------------------------------------------------------------------------------+
| 5xx Server Errors | 500 Internal Error, 502 Bad Gateway, 503 Unavailable,      |
|                   | 504 Gateway Timeout                                        |
+--------------------------------------------------------------------------------+
```

### 4. Pagination Strategies: Offset vs Cursor

#### Offset-Based (`LIMIT 20 OFFSET 1000000`)
- **How it works:** `SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;`
- **Fatal Flaws:** 
  1. **Performance ($O(N)$):** Database must scan and discard 1,000,000 index rows before returning 20.
  2. **Page Drift / Inconsistency:** If a new row is inserted while the user paginates from page 1 to page 2, the user sees duplicate items.

#### Keyset / Cursor-Based (`WHERE id > cursor LIMIT 20`)
- **How it works:** `SELECT * FROM orders WHERE id > 'ord_abc123' ORDER BY id ASC LIMIT 20;`
- **Advantages:**
  1. **Performance ($O(\log N)$):** Direct index seek using the B-Tree index. Constant latency regardless of pagination depth.
  2. **Consistency:** Immune to new inserts or deletes shifting pages.

---

## 5. Architecture / Flow

```mermaid
graph TD
    subgraph Client Layer
        Web[Web SPA]
        Mobile[Mobile iOS/Android]
    end

    subgraph API Gateway / Ingress
        GW[API Gateway]
        RateLimit[Rate Limiter (429)]
        Auth[Auth Validator (401/403)]
    end

    subgraph REST Resource Handlers
        UsersRouter["/v1/users<br/>GET: List | POST: Create"]
        UserDetailRouter["/v1/users/{id}<br/>GET: Read | PUT: Replace | PATCH: Modify | DELETE: Remove"]
        UserOrdersRouter["/v1/users/{id}/orders?cursor=xyz&limit=25<br/>GET: List Orders (Cursor Paginated)"]
    end

    subgraph Data Store
        DB[(Database with Indexes)]
    end

    Web --> GW
    Mobile --> GW
    GW --> RateLimit
    RateLimit --> Auth
    Auth --> UsersRouter
    Auth --> UserDetailRouter
    Auth --> UserOrdersRouter
    UsersRouter --> DB
    UserDetailRouter --> DB
    UserOrdersRouter --> DB
```

---

## 6. Simple Example: Clean RESTful Endpoint Design

```http
# Create a new user
POST /v1/users HTTP/1.1
Content-Type: application/json

{
  "name": "Sarah Connor",
  "email": "sarah@example.com"
}

# Response: 201 Created + Location Header
HTTP/1.1 201 Created
Location: /v1/users/usr_9876
Content-Type: application/json

{
  "id": "usr_9876",
  "name": "Sarah Connor",
  "email": "sarah@example.com",
  "created_at": "2026-08-27T10:00:00Z"
}
```

---

## 7. Production Example: Standardized Error Handling (RFC 7807) & Cursor Pagination

### Standardized Problem Details (RFC 7807)
```json
{
  "type": "https://api.example.com/errors/insufficient-inventory",
  "title": "Insufficient Inventory",
  "status": 409,
  "detail": "Product sku_4492 has only 2 items available, but 5 were requested.",
  "instance": "/v1/orders/ord_5521",
  "code": "OUT_OF_STOCK",
  "invalid_params": [
    {
      "name": "quantity",
      "reason": "Requested quantity exceeds available balance"
    }
  ]
}
```

### Production Cursor-Based Pagination Response
```json
{
  "object": "list",
  "data": [
    { "id": "ord_100", "total": 45.00, "status": "completed" },
    { "id": "ord_101", "total": 120.50, "status": "shipped" }
  ],
  "has_more": true,
  "next_cursor": "ord_101"
}
```

---

## 8. When should we use it?

- **Public APIs & Developer Platforms:** REST over JSON is the universal industry standard for third-party developer ecosystems (e.g., Stripe, GitHub, Twilio).
- **CRUD-Heavy Applications:** Systems whose domain models map naturally to resources.
- **Web Applications Benefiting from HTTP Caching:** Leveraging CDNs (Cloudflare, Fastly) for GET requests.

---

## 9. When should we avoid it?

- **High-Throughput Inter-Service Microservices:** gRPC / Protobuf is significantly faster, strongly typed, and uses binary serialization over HTTP/2.
- **Complex Graphs with Over/Under-Fetching:** GraphQL is better suited when mobile clients need to request arbitrary relational shapes in a single query.
- **Real-Time Streaming / Bi-directional Events:** WebSockets, Server-Sent Events (SSE), or gRPC streams are superior to REST polling.

---

## 10. Tradeoffs: REST vs GraphQL vs gRPC

| Metric | REST | GraphQL | gRPC |
| :--- | :--- | :--- | :--- |
| **Data Format** | JSON (Plaintext) | JSON (Plaintext) | Protocol Buffers (Binary) |
| **Transport** | HTTP/1.1, HTTP/2 | Typically HTTP/1.1 or HTTP/2 | HTTP/2 (Mandatory) |
| **Over/Under-fetching**| Common (Fixed schemas) | Solved (Client requests exact fields) | Minimal (Typed binary payload) |
| **Caching** | Excellent (Native HTTP/CDN) | Difficult (POST requests bypass CDN cache) | Custom application-level caching |
| **Streaming** | Limited | Subscriptions (WebSocket/SSE) | Native Bi-directional Streaming |
| **Tooling & Ubiquity** | Universal (Every language/curl)| Excellent developer ecosystems | Requires schema compilation (protoc) |

---

## 11. Common Mistakes

1. **Using Verbs in Endpoint URIs:** E.g., `POST /api/createUser` or `GET /api/getOrders`. Correct: `POST /v1/users`, `GET /v1/orders`.
2. **Incorrect Status Codes:** Returning `200 OK` with an error payload `{ "success": false, "error": "Unauthorized" }` breaks HTTP caching, proxies, and client error interceptors.
3. **Using Offset Pagination for Large Datasets:** Causing database query timeouts when offset reaches hundreds of thousands.
4. **Breaking API Contracts Without Versioning:** Renaming or removing fields in a live API without backwards compatibility.

---

## 12. Related Concepts

- [HTTP vs HTTPS Fundamentals](../01-networking/http-vs-https.md)
- [Idempotency & Safe Retries](../03-reliability/idempotency.md)
- [Authentication vs Authorization (401 vs 403)](../04-security/authentication-vs-authorization.md)
- [Rate Limiting (429 Handling)](../09-scalability/rate-limiting.md)

---

## 13. Interview Questions

### Q1. What is the difference between `PUT` and `PATCH` in REST APIs?
**Answer:** `PUT` is a complete replacement of the target resource representation. The client must supply all attributes; any omitted fields are overwritten or reset to null. `PATCH` is a partial update; the client transmits only the fields to be changed, and the server applies changes incrementally. Furthermore, `PUT` is strictly idempotent ($f(f(x)) = f(x)$), whereas `PATCH` is not guaranteed to be idempotent (e.g., a patch operation appending an item to a list).  
**Example:** `PUT /users/1` with `{"name": "Bob"}` deletes Bob's previous email. `PATCH /users/1` with `{"name": "Bob"}` leaves Bob's email unchanged.  
**Why this matters:** Data integrity and API contract semantics.  
**Possible follow-up:** How can you make a `PATCH` request idempotent?

### Q2. Why is Offset-based pagination an anti-pattern for large database tables?
**Answer:** In SQL, `OFFSET N` forces the database engine to traverse and evaluate $N$ rows in the index/table and discard them before returning the requested page. For `OFFSET 1000000 LIMIT 20`, the DB processes 1,000,020 rows, leading to $O(N)$ CPU and I/O degradation. Furthermore, if records are inserted or deleted while a client paginates, records shift across pages, causing duplicate reads or skipped records. Cursor/Keyset pagination (`WHERE id > last_seen_id ORDER BY id ASC LIMIT 20`) uses direct B-tree index seeks in $O(\log N)$ time and remains consistent during concurrent writes.  
**Example:** Querying page 50,000 on Postgres with offset takes 3.5s; with cursor it takes 1.2ms.  
**Why this matters:** Scalability of high-volume data APIs.  
**Possible follow-up:** When is Offset pagination still acceptable?

### Q3. What is the difference between 401 Unauthorized and 403 Forbidden?
**Answer:**
- **`401 Unauthorized`** means **Unauthenticated**: The client has not provided valid credentials (missing or expired token). The client *can* authenticate and retry.
- **`403 Forbidden`** means **Unauthorized**: The server knows who the client is, but the client does not possess the requisite permissions/roles to perform the action. Re-authenticating with the same credentials will not change the outcome.  
**Example:** No token provided -> 401. Regular user trying to call `DELETE /admin/database` -> 403.  
**Why this matters:** Core security design and client error handling logic.  
**Possible follow-up:** Should you ever return 404 instead of 403 for security?

### Q4. How do you design API Versioning in production? Compare URI Path vs Header vs Query Param.
**Answer:**
1. **URI Path (`/v1/orders` vs `/v2/orders`):** Most explicit, easiest to route at the API gateway/reverse proxy layer, easily testable in browsers. (Recommended industry standard).
2. **Custom Header (`Accept: application/vnd.company.v2+json` or `X-API-Version: 2026-08-01`):** Keeps URIs clean and truly RESTful (content negotiation), but harder to test via curl/browser and breaks standard CDN URL caching unless configured with `Vary` headers.
3. **Query Parameter (`/orders?version=2`):** Simple to implement, but litters query parameter namespaces.  
**Why this matters:** Backwards compatibility without breaking millions of live client integrations.  
**Possible follow-up:** How does Stripe implement backward-compatible date-based API versioning?

### Q5. What is Richardson Maturity Model?
**Answer:** A model classifying API maturity:
- **Level 0:** The Swamp of POX (Plain Old XML) — single URI, single HTTP POST method (e.g., SOAP/XML-RPC).
- **Level 1:** Resources — multiple URIs representing domain entities (`/users`, `/orders`), but still using single HTTP method.
- **Level 2:** HTTP Verbs & Status Codes — proper use of GET, POST, PUT, DELETE, and HTTP status codes (200, 201, 404, etc.). Most production REST APIs operate here.
- **Level 3:** Hypermedia Controls (HATEOAS) — responses include hypermedia links informing the client what next actions are possible.  
**Why this matters:** Demonstrates academic and practical understanding of REST architecture.  
**Possible follow-up:** Why is Level 3 (HATEOAS) rarely adopted in public microservices?

---

## 14. Advanced Interview Questions

### Q6. How do you handle non-CRUD actions in RESTful API design?
**Answer:** Real-world operations like "Checkout a Cart", "Cancel an Order", or "Resend Email" do not always map cleanly to CRUD. Strategies:
1. **Treat the Action as a Sub-resource / State Transition:** `POST /orders/123/cancellations` or `POST /checkout-sessions`.
2. **Use PATCH for Status Updates:** `PATCH /orders/123` with payload `{"status": "CANCELLED"}`.
3. **Dedicated Action Endpoints:** `POST /orders/123/cancel` (acceptable pragmatic REST when operations are complex business workflows).

---

## 15. Production Scenarios

### Scenario: API Client Overwhelms Backend With Polling For Async Job Status
**Problem:** Clients submit long-running report generation requests and poll `GET /reports/{id}` every 500ms, causing database CPU spikes.
**Solution:**
1. Return `202 Accepted` with a `Location: /reports/123/status` header and a `Retry-After: 30` header indicating expected wait time.
2. Replace polling with **Webhooks** or **Server-Sent Events (SSE)** / WebSockets so the server pushes the completed notification when the background job finishes.

---

## 16. Debugging Scenarios

### Scenario: Cloudflare CDN Caching Sensitive User Data
**Incident:** User A logs in and sees User B's profile data on `GET /api/v1/me`.
**Root Cause:** The endpoint returned `Cache-Control: public, max-age=300` or lacked `Cache-Control: no-store, private`, causing the edge CDN cache to store User B's response and serve it to subsequent requests.
**Fix:** Set `Cache-Control: private, no-cache, no-store, must-revalidate` and `Vary: Authorization, Cookie` on all authenticated endpoints.

---

## 17. Common Misconceptions

- *Misconception:* "Every JSON API over HTTP is a true REST API."
  - *Reality:* Most APIs are pragmatic HTTP APIs (Level 2). True REST requires hypermedia links (HATEOAS) and strict statelessness.
- *Misconception:* "DELETE operations must always return 204 No Content."
  - *Reality:* If a DELETE operation returns a payload (e.g., the deleted entity or an undo token), `200 OK` is appropriate. If the deletion is scheduled asynchronously, `202 Accepted` is returned.

---

## 18. Quick Revision

- REST resources are nouns (`/users`), manipulated via HTTP verbs (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`).
- `PUT` = complete replacement (idempotent); `PATCH` = partial update.
- Use **Cursor-based pagination** for high-volume datasets.
- Follow **RFC 7807** Problem Details for structured error responses.
- Version APIs via URI paths (`/v1/...`) for clarity and proxy routability.

---

## 19. Interview-Ready Answer

> "A well-architected REST API treats domain entities as resources identified by plural nouns, manipulated through standard HTTP verbs with proper safety and idempotency guarantees. It utilizes accurate HTTP status codes, implements cursor-based pagination to ensure constant O(log N) lookup times and consistent page states, formats errors using RFC 7807 Problem Details, and maintains strict statelessness so any backend node can process any incoming request."
