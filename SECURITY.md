# Security Policy

## Supported Versions

Only the latest release on `main` and the current `release/vX.x` branches receive security updates.

| Version | Supported |
|---------|-----------|
| `main` (latest) | ✅ |
| `release/v1.x` (current) | ✅ |
| Older branches | ❌ |

## Reporting a Vulnerability

**Do not report security vulnerabilities through public GitHub issues.**

Security vulnerabilities in Axions360 may affect financial data, tenant isolation, authentication, or personal data. Please treat them with appropriate confidentiality.

### How to report

Send an email to **security@atanax.com** with:

1. **Description** — what the vulnerability is and where it exists
2. **Impact** — what an attacker could do by exploiting it
3. **Reproduction steps** — how to reproduce the issue
4. **Affected versions** — which branches or releases are affected
5. **Suggested fix** (optional) — if you have one

Encrypt your message using our PGP key if the vulnerability is critical:
`security@atanax.com` — key available at [atanax.com/security](https://atanax.com/security)

### What to expect

- **Acknowledgement** within 48 hours
- **Initial assessment** within 5 business days
- **Fix timeline** communicated based on severity
- **Credit** in the security advisory if you wish

### Severity levels

We follow the [CVSS v3.1](https://www.first.org/cvss/v3.1/specification-document) severity scale:

| Severity | Response time |
|----------|--------------|
| Critical (9.0–10.0) | 24–48 hours |
| High (7.0–8.9) | 5 business days |
| Medium (4.0–6.9) | 30 days |
| Low (0.1–3.9) | Next release cycle |

## Security Architecture

Axions360 is built with security as a foundational principle, not an afterthought.

Key security properties:

- **Tenant isolation** enforced at application scope, query level, and database Row Level Security
- **PII encryption** — sensitive personal data encrypted AES-256-GCM at application level
- **Blind indexes** — searchable encrypted fields via HMAC-SHA256
- **Append-only audit logs** — `audit_logs` and `domain_events` cannot be modified or deleted
- **ULID identifiers** — no integer ID enumeration possible via API
- **Idempotency keys** — all financial operations protected against double-execution
- **Post-quantum cryptography** — ML-KEM and ML-DSA in the security module (Rust)
- **mTLS** — mutual TLS between critical internal services

## Scope

In scope for security reports:

- Tenant data isolation bypass
- Authentication and session vulnerabilities
- Authorization (RBAC) bypass
- SQL injection or data exfiltration
- Financial data integrity issues
- PII exposure or encryption weaknesses
- API security issues

Out of scope:

- Issues requiring physical access to servers
- Social engineering attacks
- Vulnerabilities in third-party dependencies (report to them directly)
- Issues in non-production or development configurations

---

*ATANAX Inc. — Montreal, Canada*
*security@atanax.com*
