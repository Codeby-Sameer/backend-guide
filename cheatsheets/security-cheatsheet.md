# Security & Authentication Cheatsheet

A practical security cheat sheet covering OWASP API security, JWT hardening, cookie flags, password hashing, and authorization models.

---

## 1. OWASP API Security Top 10 Summary

```
API1: Broken Object Level Authorization (BOLA/IDOR) -> Verify tenant/user owns the object.
API2: Broken Authentication                         -> Weak tokens, missing MFA, bad refresh rotation.
API3: Broken Object Property Level Authorization    -> Mass assignment (exposing sensitive fields).
API4: Unrestricted Resource Consumption             -> Missing rate limiting, unbounded page sizes.
API5: Broken Function Level Authorization (BFLA)    -> Non-admins accessing /admin endpoints.
API6: Unrestricted Access to Sensitive Business Flows -> Bot scalping, coupon abuse, spam.
API7: Server Side Request Forgery (SSRF)            -> Backend fetching arbitrary user-supplied URLs.
API8: Security Misconfiguration                     -> Default passwords, verbose stack traces in prod.
API9: Improper Inventory Management                 -> Zombie/shadow APIs without authentication.
API10: Unsafe Consumption of Third-Party APIs      -> Blindly trusting upstream webhook payloads.
```

---

## 2. JWT Security Checklist

- [x] **Algorithm Restriction:** Explicitly whitelist algorithms (`algorithms=['RS256']`). Reject `alg: "none"`.
- [x] **Asymmetric Signing (RS256/ES256):** Auth server signs with Private Key; microservices verify with Public Key.
- [x] **Short Lifespan:** Access token TTL = 5 to 15 minutes.
- [x] **Refresh Token Rotation (RTR):** Single-use refresh tokens; detect replay and revoke family.
- [x] **No Secrets in Payload:** Never store passwords, credit cards, or sensitive PII in standard JWTs.
- [x] **Key ID (`kid`) Validation:** Validate `kid` against trusted JWKS; sanitize against directory traversal.

---

## 3. Cookie Hardening Flags

```http
Set-Cookie: session_id=abc123xyz; 
            Path=/; 
            Domain=api.example.com; 
            Secure; 
            HttpOnly; 
            SameSite=Lax; 
            Max-Age=86400
```
- **`HttpOnly`:** Blocks JavaScript (`document.cookie`), mitigating XSS token theft.
- **`Secure`:** Transmits exclusively over encrypted HTTPS (Port 443).
- **`SameSite=Lax`:** Default for top-level navigations; blocks CSRF on cross-site POSTs.
- **`SameSite=Strict`:** Blocks cross-site cookie sharing completely on all incoming links.

---

## 4. Password vs API Key Storage

```
Passwords (Human-chosen, Low Entropy):
-> Mandatory: Slow, memory-hard hashing functions (bcrypt, Argon2id, PBKDF2).
-> NEVER use raw MD5, SHA-1, or plain SHA-256 for passwords!

API Keys (Machine-generated, High Entropy):
-> Mandatory: Fast cryptographic hash (SHA-256 / SHA-512) with constant-time comparison.
```
