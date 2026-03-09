# Threat Model — Patient Journal System (Phase 1)

**Method:** STRIDE
**System:** Patient Journal API — Machine-to-machine integration API (Python 3.12 / FastAPI)
**Date:** 2026-03-09

---

## 1. System Overview

The Patient Journal System Phase 1 is a REST API that receives patient data from a
legacy system via M2M integration. It processes and stores sensitive Protected Health
Information (PHI) and Personally Identifiable Information (PII).

### 1.1 Assets

| Asset | Sensitivity |
|-------|-------------|
| Patient demographic records (name, DOB, SSN, address) | High — PII |
| Medical history (diagnoses, treatments, medications) | High — PHI |
| Clinical encounters / visit notes | High — PHI |
| API credentials (client secrets, JWT signing keys) | Critical |
| Database contents | Critical |
| Audit / security logs | High |

### 1.2 Actors

| Actor | Trust Level | Description |
|-------|-------------|-------------|
| Legacy System | Semi-trusted | The authorized M2M client submitting patient data |
| API Server | Trusted (internal) | FastAPI application processing requests |
| Database | Trusted (internal) | Stores patient records and medical data |
| Log Aggregator | Trusted (internal) | Receives and stores security/audit logs |
| Attacker (external) | Untrusted | Any unauthorized party attempting to access the system |
| Compromised Insider | Untrusted | An internal actor with partial access attempting to escalate |

### 1.3 Trust Boundaries

```
┌──────────────────────────────────────────────────────────────┐
│  External Zone                                               │
│                                                              │
│   ┌───────────────┐                                          │
│   │  Legacy System│                                          │
│   │  (M2M Client) │                                          │
│   └───────┬───────┘                                          │
│           │ HTTPS / mTLS                                     │
└───────────┼──────────────────────────────────────────────────┘
            │ ◄── Trust Boundary 1 (Internet / API Gateway)
┌───────────┼──────────────────────────────────────────────────┐
│  API Zone │                                                  │
│           ▼                                                  │
│   ┌───────────────┐         ┌──────────────┐                 │
│   │  FastAPI App  │────────►│  Log System  │                 │
│   │  (API Server) │         └──────────────┘                 │
│   └───────┬───────┘                                          │
│           │ ◄── Trust Boundary 2 (Internal network)          │
└───────────┼──────────────────────────────────────────────────┘
            │
┌───────────┼──────────────────────────────────────────────────┐
│  Data Zone│                                                  │
│           ▼                                                  │
│   ┌───────────────┐                                          │
│   │   Database    │                                          │
│   │  (PHI/PII)    │                                          │
│   └───────────────┘                                          │
└──────────────────────────────────────────────────────────────┘
```

### 1.4 API Endpoints (Attack Surface)

| Endpoint | Operation | Data Handled |
|----------|-----------|--------------|
| `POST /patients` | Submit new patient record | PII (demographics) |
| `GET /patients/{id}` | Query patient profile | PII |
| `POST /patients/{id}/medical-history` | Ingest medical history | PHI |
| `POST /patients/{id}/encounters` | Submit encounter/visit | PHI |
| `GET /patients/{id}/medical-history` | Retrieve medical history | PHI |

---

## 2. STRIDE Threat Analysis

### 2.1 Spoofing

*An attacker impersonates a legitimate actor to gain unauthorized access.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| S-01 | Attacker forges requests as the legacy system by replaying or stealing its API credential | API Gateway / Auth Layer | Implement short-lived OAuth 2.0 access tokens (client credentials grant) or mTLS with client certificate validation. Rotate credentials regularly. | SR-AUTH-01, SR-AUTH-04, SR-AUTH-05, SR-TLS-06 |
| S-02 | Attacker presents a crafted or stolen JWT with modified claims (e.g., elevated `scope`) | Token Validation | Validate JWT signature against a trusted JWKS. Enforce `alg` allowlist; reject `alg: none`. Validate `aud`, `iss`, and `exp` claims. | SR-TKN-01, SR-TKN-02, SR-TKN-03, SR-TKN-04, SR-TKN-05 |
| S-03 | Attacker submits requests with a spoofed `X-Forwarded-For` or similar header to bypass IP-based controls | API Server | Do not use client-controlled headers for trust or IP-based access decisions. Strip or fix proxy headers at the gateway layer. | SR-API-03 |
| S-04 | Default or guessable service credentials exist in a deployment | API Server / DB | Remove all default accounts. Require strong, randomly-generated credentials for all service accounts. | SR-AUTH-02, SR-AUTH-04 |

---

### 2.2 Tampering

*An attacker modifies data in transit or at rest.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| T-01 | Patient records are modified in transit between the legacy system and the API (MITM) | Network / TLS | Enforce TLS 1.2+ on all endpoints. Disable HTTP. Use certificate pinning or mTLS for the M2M channel. | SR-TLS-01, SR-TLS-02, SR-TLS-06 |
| T-02 | An attacker with database access modifies patient PHI directly (bypassing the API) | Database | Encrypt sensitive fields at rest using authenticated encryption (AES-256-GCM). Integrity-protected storage detects unauthorized modifications. | SR-CRYPTO-05, SR-CRYPTO-06 |
| T-03 | SQL injection via patient data fields (name, diagnosis text, etc.) modifies database records | API / Database Layer | Use parameterized queries / ORM exclusively. Never construct queries from user-controlled input. | SR-INJ-04 |
| T-04 | An attacker modifies a patient ID in a request to overwrite another patient's record | API / Authorization Layer | Validate that the authenticated client is authorized to write to the specific patient record. Enforce object-level authorization checks. | SR-AUTHZ-04, SR-AUTHZ-06 |
| T-05 | Audit logs are modified or deleted to conceal unauthorized access or data changes | Log System | Write logs to an append-only, logically separate system. Application processes must not have delete/modify rights on log storage. | SR-LOG-10, SR-LOG-11 |
| T-06 | HTTP request smuggling allows an attacker to inject a malicious request into the pipeline | Load Balancer / API Server | Ensure consistent HTTP parsing across all components. Reject ambiguous `Content-Length`/`Transfer-Encoding` combinations. | SR-API-04 |

---

### 2.3 Repudiation

*An actor denies having performed an action; the system cannot prove otherwise.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| R-01 | The legacy system denies having submitted a fraudulent or erroneous patient record | Audit Logging | Log all write operations with the authenticated client identity, timestamp, endpoint, resource ID, and outcome. Logs must be tamper-evident. | SR-LOG-01, SR-LOG-02, SR-LOG-05 |
| R-02 | An operator denies having accessed or exported patient records | Audit Logging | Log all read operations on PHI endpoints (`GET /patients/{id}`, `GET /patients/{id}/medical-history`) including the requesting identity and resource accessed. | SR-LOG-02, SR-LOG-06 |
| R-03 | Inconsistent timestamps across system components make it impossible to reconstruct an incident timeline | Logging Infrastructure | Synchronize all system clocks via NTP. Use UTC timestamps in all logs. | SR-LOG-03 |
| R-04 | Log content is forged by injecting newline characters into patient name or notes fields | Log Processing | Encode or sanitize all logged values to prevent log injection. | SR-LOG-09 |

---

### 2.4 Information Disclosure

*Sensitive data is exposed to unauthorized parties.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| I-01 | PHI/PII transmitted over an unencrypted channel is intercepted | Network | Enforce TLS on all endpoints. Reject plain HTTP connections. | SR-TLS-01, SR-TLS-04 |
| I-02 | Database credentials or JWT signing keys are exposed via source code, config files, or environment dumps | Secrets Management | Store all secrets in a dedicated secrets manager. Never commit secrets to source control. | SR-CFG-01, SR-CFG-02 |
| I-03 | Stack traces, database errors, or internal configuration details are returned in API error responses | Error Handling | Return generic error messages to clients. Log detailed errors internally only. | SR-ERR-01, SR-CFG-05 |
| I-04 | An authorized client accesses another client's patient records (IDOR — Insecure Direct Object Reference) | Authorization Layer | Enforce data-level authorization: verify that the authenticated client is permitted to access the specific `{id}` requested. | SR-AUTHZ-04 |
| I-05 | PHI is cached by a reverse proxy or CDN and returned to a subsequent unauthorized request | Caching Layer | Set `Cache-Control: no-store, no-cache` on all responses containing PHI/PII. | SR-DATA-04 |
| I-06 | OpenAPI documentation (`/docs`, `/redoc`) exposes API structure and data schemas in production | API Server | Disable or protect documentation endpoints in production with authentication. | SR-CFG-07 |
| I-07 | PHI/PII appears in server access logs (e.g., patient ID in URL path, data in query strings) | Logging / API Design | Route sensitive identifiers only through request bodies or headers. Avoid placing PHI in URL paths or query parameters. | SR-DATA-03 |
| I-08 | Excessive data is returned in API responses beyond what the client requires | API Response Design | Apply data minimization: return only the fields required for the client's documented purpose. | SR-DATA-06 |
| I-09 | Encrypted data at rest is decrypted by an attacker who obtains the encryption key alongside the ciphertext | Key Management | Store encryption keys separately from the data they protect, in a dedicated key management service. | SR-CRYPTO-01, SR-CRYPTO-02 |

---

### 2.5 Denial of Service

*The service is made unavailable to legitimate users.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| D-01 | The legacy system (or a compromised process) floods bulk import endpoints (`POST /patients`, `POST /patients/{id}/medical-history`) with high-volume requests | API Gateway / Rate Limiting | Implement per-client rate limiting on all endpoints, with lower limits on write/bulk endpoints. | SR-API-05 |
| D-02 | An attacker sends extremely large payloads to exhaust server memory or processing | API Server | Enforce maximum request body size limits. Return `413 Payload Too Large` for oversized requests. | SR-VAL-01 |
| D-03 | ReDoS attack via a crafted patient name or text field that triggers catastrophic regex backtracking | Input Validation | Audit all regular expressions used in validation. Use possessive quantifiers or linear-time regex engines. | SR-INJ-06 |
| D-04 | Database connection pool exhaustion via slow or malformed queries | Database Layer | Use connection pool limits, query timeouts, and parameterized queries that prevent long-running injection payloads. | SR-INJ-04, SR-ERR-02 |
| D-05 | Application fails open on dependency unavailability (database down), potentially serving stale or incorrect data | Availability / Error Handling | Design the API to fail closed: if a required dependency is unavailable, return a `503 Service Unavailable` error, not partial or stale data. | SR-ERR-02 |

---

### 2.6 Elevation of Privilege

*An actor gains capabilities beyond what they are authorized for.*

| Threat ID | Threat | Component | Mitigations | Security Req. |
|-----------|--------|-----------|-------------|---------------|
| E-01 | The legacy system client exploits a missing authorization check to access admin or management endpoints | Authorization Layer | Enforce function-level authorization on every endpoint. No endpoint may be accessible without explicit authorization. Deny by default. | SR-AUTHZ-03, SR-AUTHZ-06 |
| E-02 | An attacker exploits a JWT with a manipulated `scope` or `role` claim to access restricted operations | Token Validation | Validate token claims server-side. Do not derive authorization from client-supplied claims without server-side verification. | SR-TKN-05, SR-AUTHZ-06 |
| E-03 | OS command injection via a file path or patient record field escalates to server-level execution | API / OS Layer | Never pass user-controlled data to OS commands. Use parameterized OS calls or avoid shell invocation entirely. | SR-INJ-05 |
| E-04 | Deserialization of a crafted payload triggers server-side code execution | Deserialization | Use schema-validated deserialization (Pydantic models). Avoid `pickle` or dynamic class instantiation from untrusted input. | SR-INJ-09 |
| E-05 | An attacker who compromises the API process gains access to secrets stored in environment variables or flat config files | Secrets Management | Use a secrets manager with runtime injection. Limit the blast radius: the API process should only access the secrets it needs. | SR-CFG-01, SR-CFG-02 |
| E-06 | SSRF allows the API server to be used as a proxy to reach internal services (database admin panel, metadata endpoints) | SSRF Protection | Validate and restrict all outbound requests against an allowlist of permitted hosts. Block requests to internal CIDR ranges and cloud metadata endpoints. | SR-INJ-10, SR-CFG-09 |

---

## 3. Risk Summary

The table below summarizes threat risk using a simple **Likelihood × Impact** rating
(H = High, M = Medium, L = Low).

| Threat ID | Category | Likelihood | Impact | Risk | Priority |
|-----------|----------|-----------|--------|------|----------|
| I-04 (IDOR) | Information Disclosure | H | H | **Critical** | P1 |
| S-01 (Credential theft) | Spoofing | H | H | **Critical** | P1 |
| T-03 (SQL Injection) | Tampering | H | H | **Critical** | P1 |
| I-03 (Error information leak) | Information Disclosure | H | M | **High** | P1 |
| S-02 (JWT forgery) | Spoofing | M | H | **High** | P1 |
| T-01 (MITM) | Tampering | M | H | **High** | P1 |
| I-02 (Secret exposure) | Information Disclosure | M | H | **High** | P1 |
| E-01 (Missing authz check) | Elevation of Privilege | M | H | **High** | P1 |
| D-01 (Bulk endpoint flood) | Denial of Service | M | M | **Medium** | P2 |
| T-05 (Log tampering) | Tampering | L | H | **Medium** | P2 |
| R-01 (Repudiation) | Repudiation | M | M | **Medium** | P2 |
| I-05 (PHI caching) | Information Disclosure | M | H | **High** | P2 |
| E-06 (SSRF) | Elevation of Privilege | L | H | **Medium** | P2 |
| I-07 (PHI in logs) | Information Disclosure | H | M | **High** | P2 |
| D-03 (ReDoS) | Denial of Service | L | M | **Low** | P3 |
| T-06 (Request smuggling) | Tampering | L | M | **Low** | P3 |

---

## 4. Security Requirements Cross-Walk

The table below maps each STRIDE category to the security requirements from
`SECURITY_REQUIREMENTS.md` that primarily mitigate threats in that category.

| STRIDE Category | Primary Mitigating Requirements |
|----------------|---------------------------------|
| **Spoofing** | SR-AUTH-01 through SR-AUTH-05, SR-TKN-01 through SR-TKN-07, SR-TLS-06, SR-SESS-01 through SR-SESS-07 |
| **Tampering** | SR-INJ-03 through SR-INJ-09, SR-CRYPTO-05, SR-CRYPTO-06, SR-TLS-01 through SR-TLS-07, SR-AUTHZ-04, SR-LOG-10, SR-API-04 |
| **Repudiation** | SR-LOG-01 through SR-LOG-11, SR-ERR-01 |
| **Information Disclosure** | SR-AUTHZ-04, SR-DATA-01 through SR-DATA-08, SR-CFG-01, SR-CFG-05, SR-CFG-07, SR-TLS-01, SR-ERR-01 |
| **Denial of Service** | SR-API-05, SR-VAL-01, SR-INJ-06, SR-ERR-02 |
| **Elevation of Privilege** | SR-AUTHZ-01 through SR-AUTHZ-06, SR-TKN-05, SR-INJ-05, SR-INJ-09, SR-INJ-10, SR-CFG-01, SR-CFG-02 |

---

## 5. Out of Scope (Phase 1)

The following threats are noted but deferred to later phases or addressed at the
infrastructure level outside this application's scope:

- **Physical security** of servers and data centres
- **Supply chain attacks** on Python dependencies (mitigated at CI/CD level — dependency scanning)
- **Insider threat** by database administrators with direct DB access (mitigated by DB-level encryption and access logging)
- **End-user facing threats** (XSS, CSRF) — not applicable in Phase 1 as there is no browser-facing interface

---

*STRIDE model references: Microsoft STRIDE Threat Modelling methodology*
*Cross-referenced with: SECURITY_REQUIREMENTS.md (OWASP ASVS v5.0.0)*
