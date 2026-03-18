# Security Requirements — Patient Journal API

**Standard:** OWASP Application Security Verification Standard (ASVS) v5.0.0
**Target Level:** L2 (with selected L3 controls given healthcare/PII sensitivity)
**Context:** Machine-to-machine REST API handling protected health information (PHI) and PII, containerized, internal-only, HTTPS-enforced.

---

## 1. Input Validation and Business Logic (ASVS V2)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-INP-01 | Input validation rules must be documented for all expected data structures (patient records, encounter payloads, medical history). | L1 (V2.1.1) |
| SR-INP-02 | Input validation must be enforced at the trusted service/backend layer, never relying on client-side validation. | L1 (V2.2.1) |
| SR-INP-03 | All inputs must be validated against an allowlist or predefined schema rules that match business requirements. | L1 (V2.2.1) |
| SR-INP-04 | Combinations of related data fields (e.g., patient DOB + record date) must be validated for logical consistency. | L2 (V2.1.2) |
| SR-INP-05 | Business logic flows (e.g., ingest history before submitting encounter) must execute in the correct sequential order. | L1 (V2.3.1) |
| SR-INP-06 | Transactions must succeed entirely or be fully rolled back; partial writes of patient records are not permitted. | L2 (V2.3.3) |
| SR-INP-07 | Anti-automation controls (rate limiting, throttling) must prevent excessive or abusive calls to any endpoint. | L2 (V2.4.1) |

---

## 2. API and Web Service Security (ASVS V4)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-API-01 | All HTTP responses must include a proper `Content-Type` header with charset to prevent MIME-type confusion. | L1 (V4.1.1) |
| SR-API-02 | Only explicitly supported HTTP methods (e.g., GET, POST, PUT) must be allowed per endpoint; all others must be rejected with `405 Method Not Allowed`. | L1/L2 (V4.1.4) |
| SR-API-03 | HTTP headers set by infrastructure intermediaries (proxies, gateways) must not be overrideable by the calling client. | L2 (V4.1.3) |
| SR-API-04 | HTTP message boundaries must be determined correctly to prevent request smuggling. | L2 (V4.2.1) |
| SR-API-05 | URIs and header fields must be validated to reject excessively long or malformed requests (protection against buffer overflow / DoS). | L2 (V4.2.5) |

---

## 3. Authentication (ASVS V6)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-AUTH-01 | Rate limiting and adaptive response controls must be documented and implemented against credential-stuffing and brute-force attacks on API authentication. | L1 (V6.1.1) |
| SR-AUTH-02 | Default or well-known service accounts (e.g., `admin`, `root`, `sa`) must be absent or disabled in all environments. | L1 (V6.3.2) |
| SR-AUTH-03 | Machine-to-machine authentication must use short-lived, cryptographically signed tokens; shared secrets or static API keys without expiry are prohibited. | L1 (V6.4.1) |
| SR-AUTH-04 | Multi-factor authentication must be required for any human administrative access to the API or its infrastructure. | L2 (V6.3.3) |
| SR-AUTH-05 | All authentication pathways must be documented; undocumented or bypass routes are prohibited. | L2 (V6.3.4) |
| SR-AUTH-06 | Password/credential hints and knowledge-based authentication ("secret questions") must not be used. | L1 (V6.4.2) |
| SR-AUTH-07 | Any authentication mechanism must be revocable upon suspected compromise (token revocation endpoint or allowlist invalidation). | L3 (V6.5.6) |

---

## 4. Session and Token Management (ASVS V7 & V9)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-TOK-01 | Session/token validation must be performed exclusively at the trusted backend service layer. | L1 (V7.2.1) |
| SR-TOK-02 | Reference tokens (API keys, session tokens) must be unique, generated via a CSPRNG, and carry at minimum 128 bits of entropy. | L1 (V7.2.3) |
| SR-TOK-03 | A new token must be issued upon each successful authentication or re-authentication. | L1 (V7.2.4) |
| SR-TOK-04 | Token termination must immediately invalidate the token; subsequent requests with that token must be rejected. | L1 (V7.4.1) |
| SR-TOK-05 | All active tokens for a service account must be invalidated if the account is disabled or deleted. | L1 (V7.4.2) |
| SR-TOK-06 | An absolute maximum token lifetime must be defined and enforced; tokens must not be perpetual. | L2 (V7.3.2) |
| SR-TOK-07 | Self-contained tokens (JWTs) must be validated via cryptographic digital signature or MAC. | L1 (V9.1.1) |
| SR-TOK-08 | Only allowlisted JWT signing algorithms are accepted; the `alg: none` algorithm must be explicitly rejected. | L1 (V9.1.2) |
| SR-TOK-09 | JWT key material must originate from trusted, pre-configured sources; `jku` and `x5u` header parameters must be validated against an allowlist. | L1 (V9.1.3) |
| SR-TOK-10 | JWT validity window (`nbf`, `exp` claims) must be validated on every request. | L1 (V9.2.1) |
| SR-TOK-11 | Each service must validate that a received token was issued specifically for it (`aud` claim). | L2 (V9.2.3) |

---

## 5. Authorization and Access Control (ASVS V8)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-AZ-01 | Authorization rules must be documented for every endpoint, covering both function-level and data-level access. | L1 (V8.1.1) |
| SR-AZ-02 | Function-level access (which service/role may call which endpoint) must be restricted to explicitly granted permissions. | L1 (V8.2.1) |
| SR-AZ-03 | Data-level access must be enforced to prevent Insecure Direct Object Reference (IDOR) and Broken Object Level Authorization (BOLA); a caller must only access records it is explicitly permitted to read or write. | L1 (V8.2.2) |
| SR-AZ-04 | Field-level access control must be implemented to prevent Broken Object Property Level Authorization (BOPLA); sensitive PHI fields must not be returned to callers without appropriate scope. | L2 (V8.2.3) |
| SR-AZ-05 | Authorization must be enforced at the trusted service layer; client-supplied role or permission claims must never be trusted without server-side verification. | L1 (V8.3.1) |
| SR-AZ-06 | Authorization changes (e.g., token revocation, role removal) must take effect immediately or within a documented, minimally acceptable window. | L3 (V8.3.2) |

---

## 6. Cryptography (ASVS V11)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-CRYPTO-01 | All cryptographic implementations must use industry-validated libraries; custom or in-house cryptographic algorithms are prohibited. | L2 (V11.2.1) |
| SR-CRYPTO-02 | Weak or broken block cipher modes (ECB) and weak padding schemes must not be used; AES-GCM or equivalent authenticated encryption is required. | L1 (V11.3.1 / V11.3.2) |
| SR-CRYPTO-03 | Encrypted data must have its integrity verified via authenticated encryption (e.g., AES-GCM) or an attached MAC. | L2 (V11.3.3) |
| SR-CRYPTO-04 | Broken or weak hash functions (MD5, SHA-1) must not be used for any security-relevant purpose. | L1 (V11.4.1) |
| SR-CRYPTO-05 | Passwords and secrets at rest must be stored using an approved, salted key derivation function (e.g., Argon2id, bcrypt, scrypt). | L2 (V11.4.2) |
| SR-CRYPTO-06 | All random values used for security purposes (tokens, nonces, salts) must be generated using a CSPRNG with at least 128 bits of entropy. | L2 (V11.5.1) |
| SR-CRYPTO-07 | A cryptographic key lifecycle policy (generation, rotation, expiry, revocation) must be defined, documented, and enforced per NIST SP 800-57 or equivalent. | L2 (V11.1.1) |
| SR-CRYPTO-08 | A cryptographic inventory must be maintained, listing all algorithms, keys, and certificates used in the system. | L2 (V11.1.2) |

---

## 7. Secure Communication / TLS (ASVS V12)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-TLS-01 | Only TLS 1.2 and TLS 1.3 must be enabled; older protocol versions (SSLv3, TLS 1.0, TLS 1.1) must be disabled. | L1 (V12.1.1) |
| SR-TLS-02 | Only recommended cipher suites with forward secrecy (ECDHE) must be permitted; export-grade or NULL cipher suites are prohibited. | L2/L3 (V12.1.2) |
| SR-TLS-03 | TLS must be required for all external HTTP-based communication; plaintext HTTP must not be accepted. | L1 (V12.2.1) |
| SR-TLS-04 | External-facing endpoints must use publicly trusted TLS certificates from a recognized CA. | L1 (V12.2.2) |
| SR-TLS-05 | All internal service-to-service communication must also use TLS (no plaintext internal channels). | L2 (V12.3.3) |
| SR-TLS-06 | TLS clients within the system must validate peer certificates before communicating; certificate errors must fail closed, not be silently ignored. | L2 (V12.3.2) |
| SR-TLS-07 | Internal TLS connections must use certificates from a trusted internal CA or explicitly configured trusted certificate store (not arbitrary self-signed certs). | L2 (V12.3.4) |
| SR-TLS-08 | Mutual TLS (mTLS) must be evaluated and implemented for intra-service authentication between backend components. | L3 (V12.3.5) |

---

## 8. Secrets Management and Configuration (ASVS V13)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-SEC-01 | All secrets (API keys, DB credentials, TLS private keys, signing keys) must be stored in a dedicated secrets/key management solution (e.g., HashiCorp Vault, AWS Secrets Manager); hardcoded secrets in source code or container images are prohibited. | L2 (V13.3.1) |
| SR-SEC-02 | Access to the secrets manager must follow the principle of least privilege; each service must only have access to the secrets it strictly requires (RBAC enforced). | L2 (V13.3.2) |
| SR-SEC-03 | Secrets must be configured with an expiry and automatic rotation schedule; the rotation schedule must be documented. | L3 (V13.3.4) |
| SR-SEC-04 | Backend service communications must be authenticated using service accounts or short-lived tokens; default or shared credentials are prohibited. | L2 (V13.2.1 / V13.2.3) |
| SR-SEC-05 | Backend services must operate under least-privilege principles; service accounts must not hold permissions beyond what their function requires. | L2 (V13.2.2) |
| SR-SEC-06 | An allowlist of permitted external services and resources must be defined and enforced (SSRF prevention). | L2 (V13.2.4 / V13.2.5) |
| SR-SEC-07 | All application communication dependencies (external services, databases, message queues) must be documented. | L2 (V13.1.1) |
| SR-SEC-08 | Source control metadata directories (`.git`, `.svn`) must not be accessible in any deployed container or environment. | L1 (V13.4.1) |
| SR-SEC-09 | Debug modes, verbose error output, and stack traces must be disabled in production. | L2 (V13.4.2) |
| SR-SEC-10 | Internal API documentation endpoints (Swagger UI, OpenAPI explorer) and monitoring/health endpoints must not be publicly exposed. | L2 (V13.4.5) |

---

## 9. Data Protection (ASVS V14)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-DATA-01 | All sensitive data categories (PHI, PII, credentials) must be identified and classified into protection levels with documented handling requirements. | L2 (V14.1.1 / V14.1.2) |
| SR-DATA-02 | Sensitive data (patient identifiers, medical history, encounter details) must only be transmitted in HTTP request/response bodies or headers, never in URL query strings. | L1 (V14.2.1) |
| SR-DATA-03 | Responses containing sensitive PHI/PII must include appropriate cache-control headers (`Cache-Control: no-store`, `Pragma: no-cache`) to prevent caching by proxies and load balancers. | L2 (V14.2.2 / V14.3.2) |
| SR-DATA-04 | Sensitive data must not be transmitted to untrusted third parties; data flows to external services must be documented and approved. | L2 (V14.2.3) |
| SR-DATA-05 | Encryption and integrity controls must be applied to data at rest according to its classified protection level. | L2 (V14.2.4) |
| SR-DATA-06 | API responses must return only the minimum required subset of fields; entire internal data models must not be exposed (defense against data over-exposure). | L1 (V15.3.1) |
| SR-DATA-07 | Sensitive data must be subject to a documented retention policy with automatic deletion or anonymization after the retention period expires. | L3 (V14.2.7) |
| SR-DATA-08 | Logs must not contain unmasked sensitive data (PHI, PII, credentials); masking or hashing must be applied per the data protection classification. | L2 (V16.2.5) |

---

## 10. Security Logging and Error Handling (ASVS V16)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-LOG-01 | A logging inventory must be maintained, documenting what is logged, when, where, who has access, and the retention period. | L2 (V16.1.1) |
| SR-LOG-02 | All log entries must include sufficient metadata: timestamp (UTC), source IP/service, authenticated identity, operation performed, and outcome. | L2 (V16.2.1) |
| SR-LOG-03 | Log timestamps must use a synchronized time source; all timestamps must be in UTC or include an explicit UTC offset. | L2 (V16.2.2) |
| SR-LOG-04 | Logs must be written in a structured, machine-readable format (e.g., JSON) to allow correlation and ingestion by a SIEM or log processor. | L2 (V16.2.4) |
| SR-LOG-05 | All authentication events must be logged, including successful authentications, failed attempts, and token invalidations. | L2 (V16.3.1) |
| SR-LOG-06 | All failed authorization attempts must be logged, including the identity, endpoint, and reason for denial. | L2 (V16.3.2) |
| SR-LOG-07 | Security-relevant events (e.g., access to records outside a caller's normal pattern, attempted access to restricted endpoints) must be logged and surfaced for alerting. | L2 (V16.3.3) |

---

## 11. Secure Coding and Dependency Management (ASVS V15)

| ID | Requirement | Level |
|----|-------------|-------|
| SR-DEP-01 | Risk-based remediation timeframes must be defined and documented for vulnerable third-party components. | L1 (V15.1.1) |
| SR-DEP-02 | A Software Bill of Materials (SBOM) must be generated and maintained for all third-party libraries and base container images. | L2 (V15.1.2) |
| SR-DEP-03 | All components must be within their documented update/remediation timeframe; components with known critical vulnerabilities must not be deployed to production. | L1 (V15.2.1) |
| SR-DEP-04 | Production container images must not include development tools, test code, or unnecessary OS packages. | L2 (V15.2.3) |
| SR-DEP-05 | Third-party components must be sourced from trusted, verified repositories to mitigate dependency confusion and supply-chain attacks. | L3 (V15.2.4) |
| SR-DEP-06 | Mass assignment vulnerabilities must be mitigated by explicitly allowlisting accepted fields in all request deserialization paths; model binding must not automatically map all request properties to internal objects. | L2 (V15.3.3) |

---

## Summary Matrix

| Category | Requirements | Min Level |
|----------|-------------|-----------|
| Input Validation | SR-INP-01 to SR-INP-07 | L1 |
| API Security | SR-API-01 to SR-API-05 | L1 |
| Authentication | SR-AUTH-01 to SR-AUTH-07 | L1 |
| Token Management | SR-TOK-01 to SR-TOK-11 | L1 |
| Authorization | SR-AZ-01 to SR-AZ-06 | L1 |
| Cryptography | SR-CRYPTO-01 to SR-CRYPTO-08 | L1 |
| TLS / Secure Comms | SR-TLS-01 to SR-TLS-08 | L1 |
| Secrets Management | SR-SEC-01 to SR-SEC-10 | L1 |
| Data Protection | SR-DATA-01 to SR-DATA-08 | L1 |
| Security Logging | SR-LOG-01 to SR-LOG-07 | L2 |
| Dependencies / SBOM | SR-DEP-01 to SR-DEP-06 | L1 |

> **Total: 71 security requirements** derived from OWASP ASVS v5.0.0, scoped to a healthcare REST API handling PHI/PII.
