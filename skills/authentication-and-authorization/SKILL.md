# Skill: Authentication and Authorization

## Purpose
Implement a production-hardened, standards-compliant authentication and authorization system. Security cannot be bolted on after the fact — it must be designed into the system from the start.

## When to Use
Apply this skill whenever building or modifying any system that has users, API consumers, or sensitive data. This includes greenfield implementations, extending existing auth systems, and auditing existing implementations.

## Core Principles
- Never roll your own cryptography. Use established libraries and protocols.
- Assume breach: design so that a stolen token causes minimal damage.
- Apply the principle of least privilege everywhere — users and services should have only the access they need.
- Defense in depth: multiple overlapping security controls, not a single point of protection.
- All authentication failures must be logged with enough context to investigate but without logging credentials.

## Authentication Patterns

### 1. Session-Based Authentication (Traditional Web Apps)
- Use cryptographically signed, server-side sessions stored in Redis or a database.
- Set `HttpOnly`, `Secure`, and `SameSite=Strict` flags on session cookies.
- Rotate session ID on privilege escalation (login, role change).
- Implement absolute and sliding session timeouts.
- Invalidate sessions server-side on logout — do not rely on client clearing the cookie.

### 2. Token-Based Authentication (APIs and SPAs)
**Access Tokens (JWT)**
- Use short expiry: 15 minutes for sensitive applications, up to 1 hour for general use.
- Sign with RS256 (asymmetric) in production so public keys can be distributed without sharing the secret.
- Store access tokens in memory (JavaScript variable), never in `localStorage` or `sessionStorage` (XSS risk).
- Validate: signature, expiry (`exp`), issuer (`iss`), audience (`aud`), and token type on every request.

**Refresh Tokens**
- Use long-lived (7–30 days) refresh tokens to obtain new access tokens.
- Implement refresh token rotation: issue a new refresh token on every use; invalidate the old one.
- Store refresh tokens in `HttpOnly`, `Secure`, `SameSite=Strict` cookies or a secure server-side store.
- Implement refresh token family invalidation: if a reused (already rotated) refresh token is presented, invalidate the entire token family (potential theft detected).

### 3. OAuth 2.0 + OpenID Connect (Third-Party Identity Providers)
- Implement the Authorization Code flow with PKCE for all web and mobile clients.
- Never implement the Implicit flow (deprecated and insecure).
- Never implement Resource Owner Password Credentials flow unless integrating with a legacy system that cannot be changed.
- Supported providers: Google, GitHub, Microsoft, Apple, Okta, Auth0.
- Always validate the ID token signature, `iss`, `aud`, and `nonce` claims.
- Store the `state` parameter in the server-side session to prevent CSRF during the OAuth flow.

### 4. API Key Authentication (Service-to-Service / Developer APIs)
- Generate API keys with at least 32 bytes of cryptographic randomness.
- Store only the hashed value (SHA-256 or bcrypt) of the API key; never store the plaintext after issuance.
- Display the full key to the user only once at creation time.
- Allow API keys to be scoped to specific permissions and IP ranges.
- Support key rotation without downtime: allow two valid keys during the rotation window.
- Log all API key usage with key ID (not value), timestamp, IP, and endpoint.

## Password Handling
- Hash passwords with **bcrypt** (cost factor ≥ 12), **Argon2id** (preferred), or **scrypt**.
- Never use MD5, SHA-1, or SHA-256 directly for password hashing (not designed for this purpose).
- Enforce password policy: minimum 12 characters, check against known-breached password lists (e.g., HaveIBeenPwned API).
- Send password reset tokens that expire in ≤ 1 hour, are single-use, and are invalidated on use.
- Prevent username enumeration in login and password reset flows — return the same response regardless of whether the account exists.

## Multi-Factor Authentication (MFA)
- Implement TOTP (Time-based One-Time Password) using RFC 6238 (e.g., Google Authenticator, Authy).
- Support WebAuthn / FIDO2 passkeys as a phishing-resistant second factor.
- Offer recovery codes (one-time use, hashed storage) for account recovery when MFA device is lost.
- Rate-limit MFA code attempts (max 5 attempts before temporary lockout).
- Notify users via email when MFA is enabled, disabled, or when a new device is trusted.

## Authorization Models

### Role-Based Access Control (RBAC)
- Define roles at the application level (e.g., `admin`, `editor`, `viewer`).
- Assign roles to users; check roles on every protected operation.
- Store role assignments in the database; include role claims in JWTs for stateless checks.
- Never trust role claims from a client-supplied token without verification.

### Attribute-Based Access Control (ABAC)
- Use when authorization depends on resource attributes (e.g., "users can only edit their own posts").
- Implement as policy functions: `canPerform(user, action, resource) => boolean`.
- Keep policies centralized and testable independently of business logic.
- Consider dedicated policy engines (Open Policy Agent) for complex multi-tenant systems.

### Scope-Based Access Control (OAuth Scopes)
- Define fine-grained permission scopes (e.g., `read:profile`, `write:orders`).
- Request minimal scopes needed for each client.
- Validate scopes on the resource server for every protected endpoint.

## Implementation Process

1. Define the authentication model required (session, JWT, OAuth 2.0, or combination).
2. Implement user registration with password hashing (Argon2id or bcrypt).
3. Implement login: validate credentials, issue access token and refresh token.
4. Implement token refresh endpoint with rotation and replay detection.
5. Implement logout: invalidate refresh token server-side; instruct client to discard access token.
6. Implement authentication middleware that validates tokens on every protected request.
7. Implement authorization layer (RBAC and/or ABAC) separate from authentication.
8. Implement rate limiting on all auth endpoints: login (5/min per IP), registration (10/min per IP), password reset (3/hour per email).
9. Implement account lockout: temporary lockout after N failed attempts with exponential backoff.
10. Implement MFA enrollment and verification flow.
11. Implement secure password reset flow (time-limited, single-use tokens).
12. Implement security event logging: login success/failure, password change, MFA change, logout.
13. Test all OWASP Authentication and Session Management controls.

## Security Headers and Transport
- Enforce HTTPS everywhere; redirect HTTP to HTTPS.
- Set `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`.
- Validate `Content-Type` headers on all API requests.
- Set `X-Frame-Options: DENY` and `Content-Security-Policy` headers.

## Secrets Management
- Never hardcode secrets in source code or commit them to version control.
- Load secrets from environment variables, a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager), or a CI/CD secret store.
- Rotate secrets regularly; have an emergency rotation runbook.
- Use separate secrets per environment (dev / staging / production).

## OWASP Top 10 Authentication Checklist
- [ ] A01: Broken Access Control — enforce authorization on every endpoint; deny by default
- [ ] A02: Cryptographic Failures — HTTPS everywhere, strong password hashing, encrypted sensitive data at rest
- [ ] A03: Injection — parameterized queries, never concatenate user input into queries
- [ ] A07: Identification and Authentication Failures — MFA, token rotation, secure session management
- [ ] A09: Security Logging and Monitoring — log all auth events, alert on anomalies

## Deliverables
- Secure registration and login flows
- Token issuance, validation, refresh, and revocation
- Authentication middleware for all protected routes
- Authorization layer with RBAC and/or ABAC policies
- MFA enrollment and verification
- Secure password reset flow
- Rate limiting and account lockout on auth endpoints
- Security event audit log
- Documented secrets management approach
- Auth flow documented in project README
