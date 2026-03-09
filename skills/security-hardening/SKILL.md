---
name: security-hardening
description: Build systems that are secure by default throughout the entire development lifecycle. Use when auditing for vulnerabilities, implementing security controls, hardening configurations, addressing OWASP concerns, or performing threat modeling.
---

# Skill: Security Hardening

## Purpose
Build systems that are secure by default, not secure by configuration. Security is not a feature to add at the end — it is a property that must be maintained throughout the entire software development lifecycle. The cost of a security incident vastly exceeds the cost of building securely.

## When to Use
Apply this skill continuously throughout development. Specifically invoke it during architecture design (threat modeling), during implementation (secure coding), before deployment (security review), and on a scheduled basis for running systems (security audits).

## Core Security Mindset
- **Assume breach**: design systems as if an attacker already has a foothold. Minimize blast radius.
- **Defense in depth**: implement multiple overlapping security controls. No single control should be the last line of defense.
- **Principle of least privilege**: every user, service, and process should have the minimum access necessary, and nothing more.
- **Fail securely**: when something goes wrong, default to denying access, not granting it.
- **Security is a shared responsibility**: every engineer is responsible for the security of the code they write.

## Threat Modeling

Before building, identify threats using the STRIDE model:

| Threat | Description | Example |
|--------|-------------|---------|
| **S**poofing | Impersonating a user or system | Forging a JWT, pretending to be another user |
| **T**ampering | Modifying data without authorization | Changing an order amount in a request |
| **R**epudiation | Denying an action occurred | User claims they did not make a purchase |
| **I**nformation Disclosure | Exposing data to unauthorized parties | Leaking PII in error responses |
| **D**enial of Service | Preventing legitimate users from accessing the system | Flooding the login endpoint |
| **E**levation of Privilege | Gaining higher access than authorized | Accessing admin endpoints as a regular user |

For each threat:
1. Identify the attack vector.
2. Assess likelihood (Low / Medium / High).
3. Assess impact (Low / Medium / High / Critical).
4. Define the mitigation control.
5. Assign ownership.

## OWASP Top 10 Implementation Controls

### A01: Broken Access Control
- Deny by default: all endpoints require explicit permission grants.
- Enforce access control at the server side — never trust client-side access decisions.
- Implement Insecure Direct Object Reference (IDOR) protection: validate that the requesting user owns or is authorized to access the resource identified by the ID in the request.
- Log all access control failures; alert on repeated failures (potential enumeration attack).
- Do not expose internal IDs in URLs; use opaque identifiers or UUIDs.

### A02: Cryptographic Failures
- Encrypt all sensitive data at rest (PII, financial data, health data) using AES-256.
- Use TLS 1.2+ for all data in transit; disable older versions.
- Use strong hashing for passwords: Argon2id (preferred), bcrypt (cost ≥ 12), scrypt.
- Never use MD5 or SHA-1 for security purposes (collision attacks are practical).
- Do not invent custom encryption schemes. Use established cryptographic libraries.
- Encrypt backups containing sensitive data.
- Use `Strict-Transport-Security` header to enforce HTTPS.

### A03: Injection
- **SQL Injection**: use parameterized queries or prepared statements exclusively. Never concatenate user input into SQL.
- **NoSQL Injection**: use query builder APIs, not raw query strings.
- **Command Injection**: never pass user input to shell commands; use structured APIs instead.
- **LDAP / XPath / Expression Injection**: apply input validation and parameterization.
- **Server-Side Template Injection**: use sandboxed template engines; never evaluate user-supplied templates.
- Validate all input against an allowlist (type, format, length, range).

### A04: Insecure Design
- Perform threat modeling at architecture time (see above).
- Apply the principle of least privilege in the design, not retrofitted later.
- Design for secure defaults: accounts should be created with minimal permissions; features should be disabled by default.
- Implement rate limiting and anti-automation controls in the design phase.

### A05: Security Misconfiguration
- Use separate secrets and configurations per environment.
- Disable all debug/development features before production deployment.
- Remove default accounts, credentials, and example content from production.
- Disable directory listing on web servers.
- Disable server version headers (`Server`, `X-Powered-By`).
- Apply security headers (see below).
- Automate configuration scanning with tools like Checkov, tfsec, or KICS.

### A06: Vulnerable and Outdated Components
- Run `npm audit` / `pip-audit` / `snyk test` / `govulncheck` on every CI build.
- Fail CI on HIGH or CRITICAL CVEs.
- Enable automated dependency update PRs (Dependabot, Renovate).
- Track all dependencies and their current versions in a Software Bill of Materials (SBOM).
- Remove unused dependencies — they cannot have vulnerabilities if they are not present.

### A07: Identification and Authentication Failures
- Implement account lockout or progressive delays after failed login attempts.
- Implement MFA for all privileged accounts; offer MFA to all users.
- Use secure session management (see `authentication-and-authorization` skill).
- Implement refresh token rotation with family invalidation.
- Never expose session tokens in URLs.
- Implement secure password recovery (time-limited, single-use tokens).

### A08: Software and Data Integrity Failures
- Verify the integrity of CI/CD pipelines — do not trust unreviewed external actions.
- Use signed commits (GPG or SSH signatures) for sensitive repositories.
- Pin dependencies to exact versions; verify checksums.
- Use Sigstore/cosign to sign and verify Docker images.
- Protect against deserialization vulnerabilities: never deserialize data from untrusted sources with unsafe deserializers.

### A09: Security Logging and Monitoring Failures
- Log all authentication events (success, failure, MFA, logout).
- Log all authorization failures (access denied events).
- Log all privileged operations (admin actions, permission changes, data exports).
- Centralize logs in a tamper-evident system that is separate from the application.
- Alert on: multiple failed logins from the same IP, login from new geographic location, privilege escalation, bulk data access.
- Do not log sensitive data (passwords, tokens, PII, credit card numbers).

### A10: Server-Side Request Forgery (SSRF)
- Validate and allowlist all server-side outbound URLs.
- Block requests to internal IP ranges (169.254.0.0/16, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) and metadata endpoints (169.254.169.254).
- Use a DNS resolution layer that returns only pre-approved IP ranges for outbound requests.
- Do not forward raw user-supplied URLs to any server-side HTTP client.

## HTTP Security Headers

Apply these headers to all responses from web applications:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; ...
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

Use [securityheaders.com](https://securityheaders.com) to validate your headers.

## Input Validation Framework

At every trust boundary (API endpoints, message queue consumers, file uploads):

1. **Type validation**: is the value the expected type?
2. **Format validation**: does it match the expected pattern (e.g., email regex, UUID format)?
3. **Length validation**: is it within acceptable min/max length bounds?
4. **Range validation**: for numbers, is the value within acceptable bounds?
5. **Allowlist validation**: for enum-like fields, is the value in the set of allowed values?
6. **Business rule validation**: does the value make sense in the application domain?

Never use denylist/blacklist validation — attackers will find the value you did not block.

## Secrets Management

### Rules
- Never hardcode secrets in source code.
- Never commit secrets to version control, even in private repositories.
- Use different secrets for every environment (dev / staging / prod).
- Rotate secrets regularly: quarterly for long-lived secrets, immediately after any suspected exposure.
- Audit who has access to each secret.

### Secret Stores
- **HashiCorp Vault**: industry standard for secret management; dynamic secrets; fine-grained access control.
- **AWS Secrets Manager / GCP Secret Manager / Azure Key Vault**: cloud-native; integrates with IAM.
- **GitHub Actions Secrets**: for CI/CD; scoped to repository or organization.
- **Doppler**: developer-friendly secret sync across environments.

### Detecting Secrets in Code
- Run **git-secrets**, **detect-secrets**, or **TruffleHog** on every commit and in CI.
- Add a pre-commit hook that scans for common secret patterns (API keys, private keys, passwords).
- If a secret is ever accidentally committed: rotate it immediately, then remove it from history using `git filter-repo`.

## Static Application Security Testing (SAST)

Run SAST tools in CI:
- **Semgrep**: fast, rule-based, supports many languages. Run with the `p/owasp-top-ten` rule set.
- **CodeQL**: deep semantic analysis; GitHub native; excellent at finding complex vulnerabilities.
- **Bandit** (Python): Python-specific security linter.
- **ESLint security plugins** (JS/TS): `eslint-plugin-security`, `eslint-plugin-no-unsanitized`.

## Dynamic Application Security Testing (DAST)

Run against staging environments:
- **OWASP ZAP**: open-source; integrate in CI with ZAP baseline scan.
- **Burp Suite**: manual and automated scanning; comprehensive coverage.
- Target scans at: authentication endpoints, file upload endpoints, user input fields, API endpoints.

## Container and Infrastructure Security

- **Image scanning**: scan Docker images for CVEs with Trivy, Snyk Container, or Grype in CI. Fail on HIGH/CRITICAL.
- **Non-root containers**: never run containers as root.
- **Read-only filesystem**: use `--read-only` for containers that do not need write access.
- **No privileged containers**: avoid `--privileged` flag.
- **Network policies**: implement Kubernetes NetworkPolicies to restrict pod-to-pod communication.
- **Resource limits**: set CPU and memory limits on all containers to prevent resource exhaustion.
- **Image signing**: sign Docker images with Cosign; verify signatures before deployment.
- **Base image**: use minimal base images (Alpine, distroless). Smaller attack surface.

## Penetration Testing and Bug Bounty

For systems handling sensitive data or serving many users:
- Conduct internal penetration testing before major launches.
- Engage an external penetration testing firm annually.
- Consider a bug bounty program (HackerOne, Bugcrowd) for continuous security research.
- Define a responsible disclosure policy (security.txt, `SECURITY.md`).

## Security Review Checklist (Pre-Deployment)

Run this checklist before every production deployment:

- [ ] No hardcoded secrets in code or configuration
- [ ] All inputs validated and sanitized at trust boundaries
- [ ] Parameterized queries used for all database operations
- [ ] Authentication and authorization enforced on all protected endpoints
- [ ] Dependency audit passing (no HIGH/CRITICAL CVEs)
- [ ] SAST scan passing (no HIGH/CRITICAL findings)
- [ ] Security headers configured
- [ ] TLS configured with strong cipher suites
- [ ] All debug/development features disabled in production configuration
- [ ] Logging captures all security-relevant events
- [ ] Rate limiting active on all auth endpoints
- [ ] Secrets loaded from secret store, not environment variables with plaintext values
- [ ] Docker image scanned and passing

## Incident Response for Security Events

When a security incident is detected:
1. **Contain**: immediately revoke compromised credentials, tokens, or keys.
2. **Assess**: determine what was accessed, what was modified, and for how long.
3. **Notify**: inform affected users and regulators within required timeframes (GDPR: 72 hours).
4. **Preserve**: preserve logs and evidence before remediation for forensic investigation.
5. **Remediate**: fix the vulnerability; deploy with priority.
6. **Review**: conduct a post-incident review to understand root cause and prevent recurrence.

## Deliverables
- Threat model documented for the system
- OWASP Top 10 controls implemented and verified
- HTTP security headers configured
- Input validation at all trust boundaries
- Dependency vulnerability scanning in CI with failure threshold
- SAST scan in CI (Semgrep or CodeQL)
- Secret detection pre-commit hook and CI scan
- Secrets stored in a proper secret store (not .env files in production)
- Docker image vulnerability scanning in CI
- Security review checklist completed before production deployment
- Responsible disclosure policy (SECURITY.md)
- Security event logging and alerting configured
