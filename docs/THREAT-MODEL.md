# Threat Model — WithPB MVP

**Version:** 1.0
**Date:** 2026-06-12
**Methodology:** STRIDE
**Status:** Initial draft — pre-code, assumption-based

---

## 1. Scope and Assumptions

This threat model covers the expected attack surface of the WithPB MVP as a typical B2B SaaS web application. Because the product is in the idea/pre-build stage, this model is based on industry-standard SaaS architecture patterns. It should be revised as the architecture is finalized.

**Assumed architecture:**
- Web frontend (browser-based SPA or SSR)
- REST or GraphQL API backend
- Relational database (user data, business data)
- Authentication via email/password or OAuth (third-party IdP)
- Hosting on cloud infrastructure (e.g., AWS, GCP, or Vercel + managed DB)
- CI/CD pipeline with secrets in environment variables or a secrets manager

**In scope:**
- User-facing web application
- API layer
- Database layer
- Authentication and session management
- Third-party integrations (OAuth, email, payments)
- CI/CD pipeline and developer toolchain
- Admin functionality

**Out of scope:**
- Physical security
- Employee social engineering (addressed separately in security policy)
- DDoS at network layer (mitigated by cloud provider / CDN)

---

## 2. Trust Boundaries and Data Flows

### Trust Boundaries

| Boundary | Description |
|----------|-------------|
| **TB-1** | Internet ↔ CDN/Edge (public untrusted traffic enters) |
| **TB-2** | CDN/Edge ↔ API server (all requests treated as untrusted) |
| **TB-3** | API server ↔ Database (trusted, internal network) |
| **TB-4** | API server ↔ Third-party services (OAuth, email, payments) |
| **TB-5** | Developer machine ↔ CI/CD pipeline ↔ production environment |
| **TB-6** | Admin UI ↔ API (elevated privileges, separate auth flow) |

### Data Flows

| Flow ID | Source | Destination | Data | Trust Boundary Crossed |
|---------|--------|-------------|------|----------------------|
| DF-1 | Browser | API server | User credentials, session tokens | TB-1, TB-2 |
| DF-2 | API server | Database | User records, business data, secrets | TB-3 |
| DF-3 | Browser | OAuth IdP | Auth code, tokens | TB-1, TB-4 |
| DF-4 | API server | Email service | Transactional emails, PII | TB-4 |
| DF-5 | API server | Payment processor | Payment tokens, billing data | TB-4 |
| DF-6 | CI/CD | Production | Build artifacts, secrets | TB-5 |
| DF-7 | Admin | API server | Admin operations, user data | TB-6 |

---

## 3. STRIDE Threat Analysis

### Legend

| Rating | Description |
|--------|-------------|
| **CRITICAL** | Immediate data breach, full system compromise, or regulatory violation |
| **HIGH** | Significant data exposure, privilege escalation, or service disruption |
| **MEDIUM** | Limited data exposure, partial compromise, or degraded service |
| **LOW** | Minimal impact, requires chaining with other vulnerabilities |

---

### 3.1 Spoofing (Identity)

**S-01 — Credential stuffing against login endpoint**
- **Component:** Authentication (DF-1)
- **Description:** Attackers use leaked credential lists from other breaches to brute-force user accounts.
- **Risk:** HIGH
- **Mitigations:**
  - Implement account lockout after N failed attempts (e.g., 5 within 15 min)
  - Rate-limit login endpoint per IP and per account
  - Use CAPTCHA or risk-based challenges on suspicious patterns
  - Alert users of login from new devices/locations

**S-02 — Session token theft / session hijacking**
- **Component:** Session management (DF-1)
- **Description:** Attacker steals a valid session token via XSS, network interception, or log exposure, then impersonates the user.
- **Risk:** HIGH
- **Mitigations:**
  - Set `HttpOnly`, `Secure`, `SameSite=Strict` on session cookies
  - Use short-lived tokens with refresh token rotation
  - Log and alert on concurrent sessions from different IPs
  - Enforce HTTPS everywhere (HSTS header)

**S-03 — OAuth token replay / CSRF on OAuth callback**
- **Component:** OAuth flow (DF-3)
- **Description:** Attacker intercepts or reuses OAuth `code` to authenticate as the victim.
- **Risk:** HIGH
- **Mitigations:**
  - Validate `state` parameter on all OAuth callbacks
  - Use PKCE for all OAuth/OIDC flows
  - Validate `nonce` for OpenID Connect ID tokens
  - Enforce one-time use on authorization codes

**S-04 — JWT algorithm confusion / weak secret**
- **Component:** API authentication (DF-1, DF-2)
- **Description:** If using JWTs, attacker exploits `alg:none` or brute-forces a weak HMAC secret to forge tokens.
- **Risk:** CRITICAL
- **Mitigations:**
  - Enforce explicit algorithm whitelist (reject `none` and asymmetric-to-symmetric downgrade)
  - Use RS256 or ES256 with proper key rotation
  - Validate `iss`, `aud`, `exp`, `nbf` claims on every request

---

### 3.2 Tampering (Integrity)

**T-01 — SQL / NoSQL injection**
- **Component:** API ↔ Database (DF-2)
- **Description:** Malicious input in API parameters alters database queries, enabling data read/write/delete.
- **Risk:** CRITICAL
- **Mitigations:**
  - Use parameterized queries or ORM with safe defaults for ALL database access
  - Never interpolate user input into query strings
  - Apply principle of least privilege to DB service account

**T-02 — Mass assignment / object property pollution**
- **Component:** API layer (DF-1)
- **Description:** Attacker includes unauthorized fields in request bodies (e.g., `isAdmin: true`) that are applied without filtering.
- **Risk:** HIGH
- **Mitigations:**
  - Use explicit input schemas (Zod, Joi, Pydantic) with allowlists
  - Never pass raw request bodies directly to ORM `create`/`update` methods
  - Separate DTOs from internal models

**T-03 — Insecure direct object reference (IDOR)**
- **Component:** API resource endpoints (DF-1, DF-2)
- **Description:** Attacker modifies a resource ID in a request (e.g., `GET /api/invoices/456`) to access another user's data.
- **Risk:** HIGH
- **Mitigations:**
  - Enforce authorization checks on every resource access: does the requesting user own this resource?
  - Use UUIDs rather than sequential IDs for public-facing identifiers
  - Implement row-level security or tenant-scoped query filters

**T-04 — Dependency supply chain tampering**
- **Component:** CI/CD pipeline (TB-5)
- **Description:** Attacker publishes a malicious version of a package dependency; CI pulls it into the build.
- **Risk:** HIGH
- **Mitigations:**
  - Pin dependency versions in lockfile (`package-lock.json`, `yarn.lock`, `poetry.lock`)
  - Enable automated CVE scanning (GitHub Dependabot, Snyk, or equivalent)
  - Verify package integrity with checksums in CI
  - Use a private registry or allowlist for production dependencies

---

### 3.3 Repudiation (Non-repudiation)

**R-01 — No audit trail for sensitive operations**
- **Component:** API layer, Admin (DF-7)
- **Description:** Critical actions (account deletion, billing changes, admin operations) leave no durable audit log, making incident response and compliance difficult.
- **Risk:** MEDIUM
- **Mitigations:**
  - Implement an immutable audit log for: auth events (login, logout, failed attempts), data mutations, admin actions, billing events
  - Store logs out-of-band from the primary database (append-only, separate IAM permissions)
  - Include actor ID, resource ID, action, timestamp, and IP on every log entry

**R-02 — Logs contain sensitive data**
- **Component:** Application logging (DF-2)
- **Description:** Developers inadvertently log passwords, tokens, or PII; log aggregation systems then expose this data.
- **Risk:** MEDIUM
- **Mitigations:**
  - Implement log sanitization middleware that redacts tokens, passwords, and PII fields
  - Define and enforce a structured log schema
  - Apply access controls and encryption to log storage

---

### 3.4 Information Disclosure

**I-01 — Cross-site scripting (XSS)**
- **Component:** Web frontend (TB-1)
- **Description:** Stored or reflected XSS allows attacker to steal session tokens, exfiltrate user data, or perform actions on behalf of the victim.
- **Risk:** HIGH
- **Mitigations:**
  - Use a framework with auto-escaping by default (React, Vue, etc.) — never use `dangerouslySetInnerHTML` with user content
  - Set `Content-Security-Policy` header to restrict script sources
  - Sanitize any user-generated HTML with a safe allowlist library (DOMPurify)
  - Set `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY`

**I-02 — Verbose error messages / stack traces to clients**
- **Component:** API layer (DF-1)
- **Description:** Unhandled exceptions return stack traces or internal details to the browser, leaking file paths, database schema, or library versions.
- **Risk:** MEDIUM
- **Mitigations:**
  - Catch and normalize all unhandled exceptions at the API gateway level
  - Return generic error messages to clients; log full details server-side
  - Use different error verbosity for development vs. production environments

**I-03 — Sensitive data in URLs / query strings**
- **Component:** API, browser (DF-1)
- **Description:** Tokens, IDs, or user data in URLs get captured in logs, browser history, and Referer headers.
- **Risk:** MEDIUM
- **Mitigations:**
  - Never put secrets, tokens, or PII in URL query strings
  - Use POST bodies for sensitive parameters
  - Ensure auth tokens are transmitted only via `Authorization` header or `HttpOnly` cookies

**I-04 — Secrets in source code / CI logs**
- **Component:** Repository (TB-5)
- **Description:** Developer commits API keys, DB credentials, or private keys; they become visible to anyone with repo access or in CI logs.
- **Risk:** CRITICAL
- **Mitigations:**
  - Enable git pre-commit hooks to detect secrets (e.g., `gitleaks`, `truffleHog`)
  - Configure GitHub secret scanning with push protection
  - Use environment variables or a secrets manager (AWS Secrets Manager, Doppler) — never `.env` files in repo
  - Rotate any credential that was ever committed, even if immediately deleted

**I-05 — Excessive data in API responses**
- **Component:** API layer (DF-1)
- **Description:** API returns full database objects (including internal fields, hashed passwords, or other users' data) instead of a scoped response DTO.
- **Risk:** MEDIUM
- **Mitigations:**
  - Define explicit response serializers/DTOs for every endpoint
  - Never return ORM model objects directly
  - Strip internal fields (`passwordHash`, `internalFlags`, etc.) at the serialization layer

---

### 3.5 Denial of Service

**D-01 — API rate limiting absent**
- **Component:** API layer (TB-2)
- **Description:** No rate limiting allows an attacker to exhaust server resources, trigger excessive third-party API costs, or enumerate data at high speed.
- **Risk:** HIGH
- **Mitigations:**
  - Apply rate limits per IP and per authenticated user on all endpoints
  - Apply stricter limits on sensitive endpoints (auth, password reset, email verification)
  - Return `Retry-After` headers on 429 responses
  - Consider token-bucket or sliding-window algorithm

**D-02 — Unvalidated file uploads**
- **Component:** API layer (DF-1)
- **Description:** Attacker uploads extremely large files, malformed archives, or polyglot files that consume disk space, memory, or CPU.
- **Risk:** MEDIUM
- **Mitigations:**
  - Enforce file size limits (e.g., 10 MB) at the application layer before processing
  - Validate MIME type and extension against an allowlist
  - Process uploaded files asynchronously in an isolated worker (not inline)
  - Store uploads in object storage (S3), not the application filesystem

**D-03 — Long-running / unbounded queries**
- **Component:** API ↔ Database (DF-2)
- **Description:** Attacker triggers queries with no result limit (e.g., `GET /search?q=*`) that scan the entire database.
- **Risk:** MEDIUM
- **Mitigations:**
  - Enforce pagination with a maximum page size on all list endpoints
  - Set database query timeouts at the connection pool level
  - Add appropriate indexes to support common query patterns

---

### 3.6 Elevation of Privilege

**E-01 — Broken access control / missing authorization**
- **Component:** API layer (DF-1, DF-7)
- **Description:** Authenticated user accesses admin endpoints or another tenant's data because the API checks authentication but not authorization.
- **Risk:** CRITICAL
- **Mitigations:**
  - Implement RBAC (Role-Based Access Control) with explicit permission checks on every route
  - Default to deny — require explicit grants, not just "user is authenticated"
  - Write authorization unit tests for each role on each sensitive endpoint
  - Separate admin routes behind a dedicated auth middleware

**E-02 — Insecure password reset flow**
- **Component:** Auth (DF-1, DF-4)
- **Description:** Password reset tokens are predictable, long-lived, or reusable, enabling account takeover.
- **Risk:** HIGH
- **Mitigations:**
  - Use cryptographically random tokens (32+ bytes from CSPRNG)
  - Expire tokens after 15–60 minutes
  - Invalidate token immediately after use (single-use)
  - Invalidate all active sessions on password reset

**E-03 — Server-side request forgery (SSRF)**
- **Component:** API layer (TB-2, TB-4)
- **Description:** Any feature that fetches a user-supplied URL (webhooks, link previews, OAuth metadata) can be abused to reach internal services (DB, metadata service, CI).
- **Risk:** HIGH
- **Mitigations:**
  - Validate and allowlist permitted URL schemes and hostnames
  - Block requests to RFC-1918 ranges (10.x, 172.16.x, 192.168.x) and loopback
  - Block requests to cloud metadata endpoints (169.254.169.254)
  - Perform URL resolution server-side, not via client-controlled redirects

**E-04 — Overprivileged CI/CD credentials**
- **Component:** CI/CD pipeline (TB-5)
- **Description:** CI secrets have production write access; a compromised pipeline deploys malicious code or exfiltrates secrets.
- **Risk:** HIGH
- **Mitigations:**
  - Use short-lived OIDC tokens for cloud deployments (not long-lived keys)
  - Apply least-privilege IAM roles scoped to the specific deployment action
  - Require manual approval for production deployments from the main branch
  - Enable branch protection to prevent force-push of malicious code to `main`

---

## 4. Risk Summary

| ID | Threat | Category | Risk | Status |
|----|--------|----------|------|--------|
| S-04 | JWT algorithm confusion / weak secret | Spoofing | CRITICAL | Open |
| T-01 | SQL / NoSQL injection | Tampering | CRITICAL | Open |
| I-04 | Secrets in source code / CI logs | Information Disclosure | CRITICAL | Open |
| E-01 | Broken access control | Elevation of Privilege | CRITICAL | Open |
| S-01 | Credential stuffing | Spoofing | HIGH | Open |
| S-02 | Session token theft | Spoofing | HIGH | Open |
| S-03 | OAuth CSRF / token replay | Spoofing | HIGH | Open |
| T-02 | Mass assignment | Tampering | HIGH | Open |
| T-03 | IDOR | Tampering | HIGH | Open |
| T-04 | Supply chain tampering | Tampering | HIGH | Open |
| I-01 | XSS | Information Disclosure | HIGH | Open |
| D-01 | Missing rate limiting | Denial of Service | HIGH | Open |
| E-02 | Insecure password reset | Elevation of Privilege | HIGH | Open |
| E-03 | SSRF | Elevation of Privilege | HIGH | Open |
| E-04 | Overprivileged CI/CD | Elevation of Privilege | HIGH | Open |
| R-01 | Missing audit trail | Repudiation | MEDIUM | Open |
| R-02 | PII in logs | Repudiation | MEDIUM | Open |
| I-02 | Verbose error messages | Information Disclosure | MEDIUM | Open |
| I-03 | Sensitive data in URLs | Information Disclosure | MEDIUM | Open |
| I-05 | Excessive API response data | Information Disclosure | MEDIUM | Open |
| D-02 | Unvalidated file uploads | Denial of Service | MEDIUM | Open |
| D-03 | Unbounded queries | Denial of Service | MEDIUM | Open |

---

## 5. Prioritized Mitigation Roadmap

### Phase 1 — Before First Commit (Foundational)
These controls must be in place before any code is written or committed:

1. **[I-04]** Configure `gitleaks` pre-commit hook + GitHub push protection for secrets
2. **[T-04]** Lock all dependency versions in package lockfiles; enable Dependabot
3. **[E-04]** Define least-privilege IAM roles for CI/CD; use OIDC where supported

### Phase 2 — During MVP Development (Build Right)
These must be implemented as features are built, not retrofitted:

4. **[T-01]** Enforce parameterized queries / ORM usage in all DB access code
5. **[E-01]** Implement RBAC middleware; require authorization tests for every role
6. **[S-01/S-02]** Set secure cookie attributes; implement login rate limiting and lockout
7. **[S-04]** Use RS256 JWTs with explicit algorithm enforcement (or switch to opaque session tokens)
8. **[T-02/T-03]** Implement input validation schemas and response DTOs for all endpoints
9. **[D-01]** Apply rate limiting middleware to all API routes

### Phase 3 — Pre-Launch Hardening
Before exposing to external users:

10. **[I-01]** Audit all user-rendered content for XSS; implement CSP header
11. **[S-03]** Verify PKCE and `state` validation in all OAuth flows
12. **[E-02]** Implement secure password reset (CSPRNG tokens, expiry, single-use)
13. **[E-03]** Audit any URL-fetching features for SSRF
14. **[R-01]** Implement audit log for auth events and admin actions
15. **[I-02/I-03/I-05]** Harden error handling and response serialization

---

## 6. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-12 | CEO Agent | Initial threat model (pre-code, assumption-based) |

---

## 7. References

- [OWASP STRIDE Threat Modeling](https://owasp.org/www-community/Threat_Modeling)
- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP ASVS 4.0](https://owasp.org/www-project-application-security-verification-standard/)
- [NIST SP 800-154: Data-Centric System Threat Modeling](https://csrc.nist.gov/publications/detail/sp/800-154/draft)
