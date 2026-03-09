# Security Requirements — Patient Journal System (Phase 1)

## Overview

This document defines security requirements for the Patient Journal System API — a
machine-to-machine (M2M) integration API built with Python 3.12 / FastAPI. The system
handles highly sensitive data: Personal Identifiable Information (PII) and Protected
Health Information (PHI), including patient demographics, medical histories, and
clinical encounters.

Requirements are derived from the **OWASP Application Security Verification Standard
(ASVS) v5.0.0** and are scoped to the threat model and data sensitivity of this system.
Each requirement references its ASVS identifier for traceability.

---

## 1. Input Validation and Injection Prevention

*ASVS Chapter: V1 — Encoding and Sanitization | V2 — Validation and Business Logic*

### 1.1 Input Decoding and Encoding Architecture

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-INJ-01 | The API must decode or unescape input into a canonical form only once, before any validation or sanitization step. Double-decoding scenarios must be explicitly prevented. | V1.1.1 | L2 |
| SR-INJ-02 | Output encoding must be applied as a final step before data is passed to any interpreter (e.g., database, OS, template engine). | V1.1.2 | L2 |

### 1.2 Injection Prevention

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-INJ-03 | All HTTP responses must include a `Content-Type` header that correctly reflects the response media type and character encoding (e.g., `application/json; charset=utf-8`). | V1.2.1 | L1 |
| SR-INJ-04 | All database queries (SQL or NoSQL) must use parameterized queries, ORMs, or equivalent mechanisms. String concatenation to build queries is prohibited. | V1.2.4 | L1 |
| SR-INJ-05 | Operating system calls (if any) must use parameterized OS queries or equivalent safe APIs. Direct shell invocation with user-controlled data is prohibited. | V1.2.5 | L1 |
| SR-INJ-06 | Regular expressions must escape special characters to prevent ReDoS (Regular Expression Denial of Service) vulnerabilities. Expressions must be audited for catastrophic backtracking. | V1.2.9 | L2 |
| SR-INJ-07 | If URLs are dynamically constructed from patient or encounter data, untrusted values must be URL-encoded. Only safe URL protocols (`https:`) are permitted in dynamic links. | V1.2.2 | L1 |

### 1.3 Safe Deserialization

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-INJ-08 | XML parsers (if used) must be configured to disable external entity processing (XXE). Unsafe features such as `FEATURE_EXTERNAL_GENERAL_ENTITIES` must be explicitly disabled. | V1.5.1 | L1 |
| SR-INJ-09 | Deserialization of untrusted data (JSON payloads from the legacy system) must enforce safe input handling. Where possible, use schema-validated models (e.g., Pydantic) rather than permissive deserialization. | V1.5.2 | L2 |

### 1.4 Server-Side Request Forgery (SSRF)

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-INJ-10 | Any functionality that causes the API server to make outbound HTTP requests must validate and restrict target URLs against an allowlist of trusted hosts to prevent SSRF. | V1.3.6 | L2 |

### 1.5 Input Validation

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-VAL-01 | All API endpoint input fields (patient identifiers, dates, clinical codes, text fields) must be validated against documented business rules for format, length, and acceptable values. | V2.2.1 | L1 |
| SR-VAL-02 | Input validation must be enforced at the service layer (FastAPI route handlers / Pydantic models). Client-supplied values must never be trusted without validation. | V2.2.2 | L1 |
| SR-VAL-03 | Combinations of related fields (e.g., encounter date must not precede patient registration date) must be validated for logical consistency. | V2.2.3 | L2 |
| SR-VAL-04 | The application must document input validation rules for all endpoint request fields, including data types, allowed values, and size limits. | V2.1.1 | L1 |

---

## 2. Authentication

*ASVS Chapter: V6 — Authentication | V9 — Self-contained Tokens*

Since this is an M2M integration API, authentication is service-to-service. Human
password-based authentication requirements (V6.2.x) are not applicable.

### 2.1 General Authentication Controls

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-AUTH-01 | Authentication controls (rate limiting, lockout, brute-force prevention) for the API must be documented. | V6.1.1 | L1 |
| SR-AUTH-02 | Default service accounts, credentials, or API keys (e.g., `admin`/`admin`) must not be present in any deployed environment. | V6.3.2 | L1 |
| SR-AUTH-03 | Brute-force and credential stuffing protections must be implemented at the API gateway or application layer (e.g., rate limiting on authentication endpoints). | V6.3.1 | L1 |
| SR-AUTH-04 | All system-generated client credentials (API keys, client secrets) must be cryptographically random and have a defined expiry. Initial credentials must be invalidated after first use where applicable. | V6.4.1 | L1 |
| SR-AUTH-05 | The application must support strong M2M authentication. The OAuth 2.0 client credentials grant or mutual TLS (mTLS) is the recommended mechanism for service-to-service authentication. | V6.3.3 | L2 |

### 2.2 Self-contained Tokens (JWT / OAuth Access Tokens)

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-TKN-01 | Self-contained tokens (e.g., JWTs) must be validated using their digital signature or MAC before any claims are trusted. | V9.1.1 | L1 |
| SR-TKN-02 | Only cryptographic algorithms on an explicit allowlist may be used to create and verify tokens. The `alg: none` algorithm must be rejected. | V9.1.2 | L1 |
| SR-TKN-03 | Key material used to validate tokens must come from trusted, pre-configured sources (e.g., JWKS endpoint of a trusted IdP). Dynamic key injection from request data is prohibited. | V9.1.3 | L1 |
| SR-TKN-04 | Token expiry (`exp` claim) must be validated. Expired tokens must be rejected. | V9.2.1 | L1 |
| SR-TKN-05 | The API must validate that a received token is of the expected type and intended for use with this service (audience claim `aud`). | V9.2.2 | L2 |
| SR-TKN-06 | The API must only accept tokens whose `aud` (audience) claim matches the service's identifier. Tokens issued to other services must be rejected. | V9.2.3 | L2 |
| SR-TKN-07 | If the same signing key is used to issue tokens for multiple audiences, the API must verify the `aud` claim to prevent token misuse across services. | V9.2.4 | L2 |

---

## 3. Session Management

*ASVS Chapter: V7 — Session Management*

For an M2M API, "sessions" correspond to access token lifetimes and revocation.

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-SESS-01 | Token verification must be performed by a trusted backend service; client-supplied session state must not be trusted. | V7.2.1 | L1 |
| SR-SESS-02 | Access tokens must be dynamically generated, not static or predictable. | V7.2.2 | L1 |
| SR-SESS-03 | Reference tokens (opaque tokens), if used instead of JWTs, must be generated with at least 128 bits of entropy using a CSPRNG. | V7.2.3 | L1 |
| SR-SESS-04 | Token inactivity timeout and absolute maximum token lifetime must be defined and documented per the application's risk profile. | V7.1.1 | L2 |
| SR-SESS-05 | Inactivity-based token expiry must be enforced, requiring re-authentication per the documented risk level. | V7.3.1 | L2 |
| SR-SESS-06 | An absolute maximum token lifetime must be enforced regardless of activity. | V7.3.2 | L2 |
| SR-SESS-07 | On token revocation (e.g., client credential rotation or integration shutdown), all tokens issued under the revoked credential must be invalidated immediately. | V7.4.2 | L1 |

---

## 4. Authorization

*ASVS Chapter: V8 — Authorization*

The API processes patient records which are subject to strict access control. Only
authorized systems (the legacy system integration) should access patient data.

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-AUTHZ-01 | Authorization rules must be documented, specifying which services (clients) may invoke each endpoint and what patient data they are permitted to read or write. | V8.1.1 | L1 |
| SR-AUTHZ-02 | Field-level access restrictions must be documented (e.g., certain sensitive fields may require elevated permissions). | V8.1.2 | L2 |
| SR-AUTHZ-03 | Function-level authorization must be enforced on every API endpoint. No endpoint may be accessible without explicit authorization verification. | V8.2.1 | L1 |
| SR-AUTHZ-04 | Data-level (patient record) access must be restricted so that a client may only access records it is explicitly authorized to view or modify. Horizontal privilege escalation (accessing another patient's record by guessing an ID) must be prevented. | V8.2.2 | L1 |
| SR-AUTHZ-05 | Field-level access controls must be enforced server-side. Sensitive fields (e.g., diagnosis codes, social security numbers) must not be returned to clients lacking appropriate permissions. | V8.2.3 | L2 |
| SR-AUTHZ-06 | Authorization must be enforced at the service layer; it must never rely solely on client-supplied claims without server-side verification. | V8.3.1 | L1 |

---

## 5. API and Web Service Security

*ASVS Chapter: V4 — API and Web Service*

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-API-01 | Every HTTP response with a message body must include a `Content-Type` header matching the actual content format and character encoding. | V4.1.1 | L1 |
| SR-API-02 | HTTP methods not explicitly required by the API (e.g., `TRACE`, `DELETE` on read-only endpoints) must be blocked. The API must return `405 Method Not Allowed` for unsupported methods. | V4.1.4 | L3 |
| SR-API-03 | HTTP header fields set by intermediary layers (load balancers, proxies) such as `X-Forwarded-For` or `X-Real-IP` must not be overridable by the API client. The API must not use client-controlled header values for trust decisions. | V4.1.3 | L2 |
| SR-API-04 | All application components (load balancer, API server) must use consistent parsing of HTTP requests to prevent HTTP request smuggling attacks. | V4.2.1 | L2 |
| SR-API-05 | Anti-automation controls (e.g., rate limiting per client, per endpoint) must be in place to prevent abuse of bulk data import endpoints (`POST /patients`, `POST /medical-history`). | V2.4.1 | L2 |

---

## 6. Cryptography

*ASVS Chapter: V11 — Cryptography*

### 6.1 Cryptographic Inventory

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-CRYPTO-01 | A documented cryptographic inventory must be maintained, covering all cryptographic mechanisms used by the system (at-rest encryption, TLS configuration, token signing, password hashing). | V11.1.2 | L2 |
| SR-CRYPTO-02 | A key management policy must be documented covering key generation, storage, rotation, and revocation procedures. | V11.1.1 | L2 |

### 6.2 Algorithms and Implementation

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-CRYPTO-03 | Only industry-validated cryptographic libraries must be used. Custom or unvetted cryptographic implementations are prohibited. | V11.2.1 | L2 |
| SR-CRYPTO-04 | Insecure cipher modes (e.g., ECB) and weak padding schemes (e.g., PKCS#1 v1.5 for encryption) must not be used. | V11.3.1 | L1 |
| SR-CRYPTO-05 | Symmetric encryption must use approved authenticated encryption modes (e.g., AES-256-GCM). | V11.3.2 | L1 |
| SR-CRYPTO-06 | Encrypted sensitive data (PHI/PII at rest) must be protected against unauthorized modification using authenticated encryption or a separate integrity mechanism. | V11.3.3 | L2 |
| SR-CRYPTO-07 | Only approved hash functions (SHA-256 or stronger) must be used for integrity verification and digital signatures. MD5 and SHA-1 are prohibited. | V11.4.1 | L1 |
| SR-CRYPTO-08 | If passwords or credentials are stored, they must use a memory-hard key derivation function (e.g., Argon2id, bcrypt, or scrypt). | V11.4.2 | L2 |
| SR-CRYPTO-09 | All security-sensitive random values (tokens, credentials, nonces) must be generated using a Cryptographically Secure Pseudo-Random Number Generator (CSPRNG). | V11.5.1 | L2 |
| SR-CRYPTO-10 | The system must be designed with cryptographic agility — the ability to replace cryptographic algorithms without major refactoring. Algorithm selection must be configurable. | V11.2.2 | L2 |

---

## 7. Secure Communication

*ASVS Chapter: V12 — Secure Communication*

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-TLS-01 | All API endpoints must be accessible only over TLS. Plaintext HTTP connections must not be accepted. | V12.2.1 | L1 |
| SR-TLS-02 | Only TLS 1.2 and TLS 1.3 must be enabled. Older protocol versions (TLS 1.0, TLS 1.1, SSL) must be disabled. | V12.1.1 | L1 |
| SR-TLS-03 | Only recommended TLS cipher suites must be enabled. Weak or deprecated cipher suites must be disabled. Cipher suite configuration must follow current BSI/NIST/Mozilla guidance. | V12.1.2 | L2 |
| SR-TLS-04 | All inbound and outbound connections between internal system components must use TLS. Unencrypted internal communication is not permitted for any channel that carries PHI/PII. | V12.3.1 | L2 |
| SR-TLS-05 | When the API client (legacy system) or internal services act as TLS clients, they must validate the server certificate chain before transmitting data. | V12.3.2 | L2 |
| SR-TLS-06 | Where mutual TLS (mTLS) is used for M2M authentication, the API must validate the client certificate against a trusted CA before processing the request. | V12.1.3 | L2 |
| SR-TLS-07 | Certificates used for external-facing TLS endpoints must be issued by a publicly trusted Certificate Authority. | V12.2.2 | L1 |

---

## 8. Configuration and Secrets Management

*ASVS Chapter: V13 — Configuration*

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-CFG-01 | All secrets (database passwords, API keys, JWT signing keys, encryption keys) must be stored in a secrets management solution (e.g., HashiCorp Vault, AWS Secrets Manager, environment variable injection via a secrets store). Secrets must never be hard-coded or committed to source control. | V13.3.1 | L2 |
| SR-CFG-02 | Access to secrets must follow the principle of least privilege. Only the components that require a secret may access it. | V13.3.2 | L2 |
| SR-CFG-03 | Secrets must have defined expiry and rotation policies. | V13.3.4 | L3 |
| SR-CFG-04 | The application must not be deployed with source control metadata (e.g., `.git` directory). | V13.4.1 | L1 |
| SR-CFG-05 | Debug mode and verbose error messages must be disabled in all production environments. Stack traces must not be returned to API clients. | V13.4.2 | L2 |
| SR-CFG-06 | The `HTTP TRACE` method must be disabled in production. | V13.4.4 | L2 |
| SR-CFG-07 | Internal API documentation endpoints (e.g., `/docs`, `/redoc`, `/openapi.json`) must not be accessible in production unless explicitly protected by authentication and authorization. | V13.4.5 | L2 |
| SR-CFG-08 | All external services and communication channels used by the application must be documented, including an allowlist of permitted outbound connections. | V13.1.1 | L2 |
| SR-CFG-09 | An allowlist of external resources and systems with which the application may communicate must be enforced at the network or application layer. | V13.2.4 | L2 |
| SR-CFG-10 | Backend service-to-service credentials must be rotated regularly and must not use the same credential as any user-facing account. | V13.2.3 | L2 |

---

## 9. Data Protection

*ASVS Chapter: V14 — Data Protection*

### 9.1 Data Classification

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-DATA-01 | All data processed by the API must be classified according to sensitivity. Patient demographics and medical history constitute PHI and must be treated as the highest sensitivity class. A documented protection policy must exist for each class. | V14.1.1 | L2 |
| SR-DATA-02 | Protection requirements for each data class must be documented, covering encryption at rest, encryption in transit, access controls, retention periods, and disposal procedures. | V14.1.2 | L2 |

### 9.2 Data Handling

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-DATA-03 | Sensitive data (PHI, PII) must only be transmitted in the HTTP message body or request headers. It must never appear in URL query parameters or path segments where it would be logged by intermediaries. | V14.2.1 | L1 |
| SR-DATA-04 | The API must prevent PHI/PII from being cached in any server-side cache layer (e.g., reverse proxy cache). Appropriate `Cache-Control: no-store` headers must be set on responses containing sensitive data. | V14.2.2 | L2 |
| SR-DATA-05 | Sensitive patient data must not be transmitted to third parties (analytics services, external trackers) without explicit consent and legal basis. | V14.2.3 | L2 |
| SR-DATA-06 | Data minimization must be applied: API responses must return only the fields required by the requesting service for its documented purpose. Over-fetching of patient data is prohibited. | V14.2.6 | L3 |
| SR-DATA-07 | Sensitive data retention and disposal rules must be enforced. Outdated patient records must be managed per the documented retention policy (e.g., anonymization or deletion after legal retention period). | V14.2.7 | L3 |
| SR-DATA-08 | Sensitive information (PII/PHI) must be removed from metadata of any files uploaded or exported by the system unless storage of such metadata is explicitly required and documented. | V14.2.8 | L3 |

---

## 10. Security Logging and Audit Trail

*ASVS Chapter: V16 — Security Logging and Error Handling*

Healthcare systems require comprehensive audit trails for regulatory compliance (e.g.,
GDPR, HIPAA-equivalent requirements).

### 10.1 Logging Requirements

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-LOG-01 | A logging inventory must exist, documenting what events are logged at each layer of the system, and the format and destination of logs. | V16.1.1 | L2 |
| SR-LOG-02 | Each log entry must include: timestamp (with timezone), source system identifier, endpoint/operation, result (success/failure), and the patient or resource identifier affected (without logging the full PHI payload). | V16.2.1 | L2 |
| SR-LOG-03 | All logging components must synchronize time sources (NTP). Timestamps in security-relevant logs must be consistent and non-ambiguous. | V16.2.2 | L2 |
| SR-LOG-04 | Logs must not contain full PHI values (e.g., medical history text, diagnoses). Only identifiers, operation types, and outcomes should be logged. | V16.2.5 | L2 |
| SR-LOG-05 | All authentication events (successful token validation, failed authentication, token expiry) must be logged. | V16.3.1 | L2 |
| SR-LOG-06 | All failed authorization attempts (e.g., a client attempting to access a patient record it is not authorized for) must be logged, including the resource requested and the client identity. | V16.3.2 | L2 |
| SR-LOG-07 | Security events defined in the application's security documentation (e.g., bulk data access, repeated failures, suspicious patterns) must be logged. | V16.3.3 | L2 |
| SR-LOG-08 | Unexpected errors, TLS failures, and security control failures must be logged for monitoring and incident response. | V16.3.4 | L2 |

### 10.2 Log Integrity and Protection

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-LOG-09 | Log data must be encoded to prevent log injection attacks (e.g., log forging via crafted newline characters in patient name fields). | V16.4.1 | L2 |
| SR-LOG-10 | Logs must be write-protected. Application processes must have write-only access to log destinations; they must not be able to modify or delete existing log entries. | V16.4.2 | L2 |
| SR-LOG-11 | Logs must be transmitted to and stored in a logically separate system from the application to prevent tampering in the event of a compromise. | V16.4.3 | L2 |

### 10.3 Error Handling

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-ERR-01 | When an unexpected or security-sensitive error occurs, the API must return a generic error message to the client. Internal error details (stack traces, database errors, configuration details) must never be exposed in responses. | V16.5.1 | L2 |
| SR-ERR-02 | The application must continue to operate securely when external dependencies (database, legacy system) are unavailable. Failures must result in safe defaults (deny access), not open access. | V16.5.2 | L2 |
| SR-ERR-03 | The application must fail gracefully and securely on unhandled exceptions, preventing partial writes or inconsistent state that could corrupt patient data. | V16.5.3 | L2 |

---

## 11. Business Logic Security

*ASVS Chapter: V2 — Validation and Business Logic*

| ID | Requirement | ASVS Ref | Level |
|----|-------------|----------|-------|
| SR-BIZ-01 | Business logic flows (e.g., patient registration, medical history ingestion) must only be processed in the correct sequence. Out-of-order operations (e.g., submitting an encounter for a non-existent patient) must be rejected. | V2.3.1 | L1 |
| SR-BIZ-02 | Business logic limits (e.g., maximum records per batch, rate of encounter submission) must be documented and enforced to prevent abuse. | V2.3.2 | L2 |
| SR-BIZ-03 | Database or data store transactions must be used to ensure that multi-step operations (e.g., writing a patient record and its initial history) are atomic. Partial writes must not result in inconsistent state. | V2.3.3 | L2 |

---

## Appendix: ASVS Level Reference

| Level | Description |
|-------|-------------|
| L1 | Minimum baseline — applicable to all applications. Provides basic defense against common attacks. |
| L2 | Standard security for applications handling sensitive data. Recommended for systems processing PHI/PII. |
| L3 | Advanced security for high-value, high-risk applications (e.g., systems with regulatory obligations such as healthcare or finance). |

**Recommended target level for this system: L2 (with selected L3 requirements for data protection and cryptography, given the PHI/PII nature of the data).**

---

*Generated: 2026-03-09*
*ASVS Version: 5.0.0 — https://owasp.org/www-project-application-security-verification-standard/*
