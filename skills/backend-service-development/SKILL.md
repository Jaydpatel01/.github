---
name: backend-service-development
description: Build production-grade backend API services that are reliable, maintainable, observable, and secure. Use when developing server-side APIs, microservices, or backend business logic.
---

# Skill: Backend Service Development

## Purpose
Build a production-grade backend API service that is reliable, maintainable, observable, and secure — not just functionally correct. A backend that works on a developer laptop but fails at scale or under adversarial conditions is not production-ready.

## When to Use
Apply this skill when building any new backend service, extending an existing one, or performing a significant refactor of backend code.

## Technology Selection Guidelines

### Node.js
- **Express.js**: Maximum ecosystem compatibility, explicit middleware control. Best for teams already familiar with it.
- **Fastify**: 2–3× faster than Express, built-in schema validation and serialization. Preferred for new greenfield projects.
- **NestJS**: Opinionated, Angular-inspired framework with TypeScript first-class support. Best for large teams needing strong conventions.

### Python
- **FastAPI**: Async-first, automatic OpenAPI docs, excellent for data-heavy services. Preferred for ML/AI integration.
- **Django REST Framework**: Battle-tested, rich ORM, admin panel. Best for content-heavy applications.

### Go
- **Gin / Echo / Chi**: Excellent for high-throughput, low-latency services where resource efficiency matters.

### Database Layer
- **PostgreSQL**: Default choice for relational data. Use `pg` (Node) or `asyncpg` / SQLAlchemy (Python).
- **MongoDB**: Only when document flexibility genuinely adds value; avoid for data with complex relationships.
- **Redis**: For caching, session storage, pub/sub, rate limiting, distributed locks, and job queues.
- **ORM Usage**: Use an ORM for CRUD operations. Drop to raw SQL for performance-sensitive queries. Never use raw SQL with string concatenation (SQL injection risk).

## Project Structure

Enforce strict separation of concerns. No business logic in route handlers; no HTTP concepts in services.

```
src/
├── api/
│   ├── routes/         # Route registration only (no logic)
│   ├── controllers/    # HTTP request parsing → service call → HTTP response
│   └── validators/     # Request schema validation (Joi, Zod, class-validator)
├── services/           # Business logic; no HTTP, no DB drivers
├── repositories/       # All database queries; no business logic
├── models/             # Data models, DTOs, domain types
├── middleware/
│   ├── auth.js         # JWT/session verification
│   ├── rateLimiter.js  # Rate limiting per endpoint
│   ├── requestLogger.js # Structured request/response logging
│   ├── errorHandler.js # Centralized error handling
│   └── validate.js     # Schema validation middleware
├── config/
│   ├── index.js        # Load and validate all environment variables at startup
│   └── database.js     # Database connection setup
├── services/
│   └── queue/          # Background job producers and consumers
├── events/             # Domain event publishers (for event-driven flows)
└── utils/              # Pure, stateless utility functions
```

## Development Process

### Step 1 — Project Initialization
1. Initialize with a package manager (npm, yarn, pnpm, poetry, etc.).
2. Add TypeScript (strongly recommended) or JSDoc types for JavaScript projects.
3. Configure ESLint + Prettier with a strict ruleset from day one.
4. Set up `.env.example` with all required environment variable names (never actual values).
5. Add `.gitignore` for `node_modules`, `dist`, `.env`, and build artifacts.
6. Add `husky` pre-commit hooks to run lint and type checks before every commit.

### Step 2 — Configuration and Secrets Management
1. Load all configuration from environment variables — never hardcode values.
2. Validate all required environment variables at startup with clear error messages. Fail fast if any are missing.
3. Use a configuration validation library (e.g., `envalid`, `joi`, `zod` for schemas).
4. Separate configuration by concern: `database`, `auth`, `queue`, `thirdParty`.
5. Never log configuration values that include secrets.

### Step 3 — Database Layer
1. Set up connection pooling (PgBouncer for PostgreSQL in production, or ORM pool settings).
2. Configure connection timeout, pool min/max, and idle timeout.
3. Implement graceful database disconnect on application shutdown.
4. Implement a database health check endpoint (`/health/db`).
5. Write all schema migrations as versioned, forward-only migration files.
6. Test migrations in CI before merging.

### Step 4 — Service Layer (Business Logic)
1. Implement all business logic in service classes/functions — isolated from HTTP and DB concerns.
2. Services receive plain objects; they return plain objects or throw typed errors.
3. Use domain-specific error types (e.g., `NotFoundError`, `ValidationError`, `ConflictError`).
4. Ensure all services are independently unit-testable without a running HTTP server or database.

### Step 5 — API Layer
1. Define all route schemas with a validation library before implementing handlers.
2. Validate all inputs (body, query params, path params, headers) at the request boundary.
3. Never trust client-supplied data. Whitelist, do not blacklist.
4. Use consistent HTTP status codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests, 500 Internal Server Error.
5. Use a consistent error response format across all endpoints.
6. Implement API versioning from day one (`/api/v1/`).
7. Return `Location` header on 201 responses pointing to the created resource.
8. Implement idempotency keys for write operations that must not be executed twice.

### Step 6 — Middleware

**Authentication Middleware**
- Verify JWT signature, expiry, issuer, and audience on every protected route.
- Attach the authenticated user context to the request object.
- Return 401 for missing or invalid tokens; return 403 for valid tokens without permission.

**Rate Limiting**
- Implement per-IP and per-user rate limits using a Redis-backed sliding window or token bucket.
- Return `429 Too Many Requests` with `Retry-After` header.
- Apply stricter limits to auth endpoints and write operations.

**Request Logging**
- Log every request with: method, path, status code, response time, user ID (if authenticated), correlation ID.
- Use structured JSON logging (Pino, Winston, structlog).
- Generate and propagate a unique `X-Request-ID` / `X-Correlation-ID` on every request.
- Never log request bodies containing passwords, tokens, or PII.

**Error Handler**
- Catch all unhandled errors in a central middleware.
- Map domain error types to appropriate HTTP status codes.
- Log the full error stack trace for 5xx errors; do not expose stack traces to clients.
- Return a correlation ID in error responses so clients can reference support logs.

**Security Headers**
- Apply `helmet` (Node.js) or equivalent to set: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Content-Security-Policy`, `Referrer-Policy`.

### Step 7 — Resilience Patterns

**Circuit Breaker**
- Wrap every external service call (third-party APIs, downstream microservices) in a circuit breaker.
- States: Closed (normal) → Open (failing, reject fast) → Half-Open (probe recovery).
- Use `opossum` (Node.js), `resilience4j` (JVM), or `pybreaker` (Python).

**Retry with Exponential Backoff and Jitter**
- Retry transient failures (network timeouts, 503s) with: initial delay 100ms, multiplier 2, max 3 retries, jitter ±25%.
- Never retry non-idempotent operations without an idempotency key.

**Timeouts**
- Set explicit timeouts on every outbound HTTP call and database query.
- Default: 5s for external APIs, 30s for database queries (adjust per operation).
- A missing timeout is a production outage waiting to happen.

**Bulkhead**
- Limit concurrent connections to each downstream dependency to prevent one failing service from exhausting all threads/connections.

### Step 8 — Background Jobs and Queues
1. Use a job queue (BullMQ + Redis, Celery + Redis/RabbitMQ) for async tasks.
2. Define job schemas with validation.
3. Implement idempotent job handlers — jobs may be retried on failure.
4. Set job TTL, retry limits, and backoff strategy per job type.
5. Implement dead-letter queues for jobs that exceed retry limits.
6. Expose job queue metrics (pending, active, failed, completed counts).

### Step 9 — Observability
1. Structured JSON logging with severity levels (debug, info, warn, error).
2. Emit application metrics (request rate, error rate, latency histograms, queue depth).
3. Instrument code with distributed tracing spans (OpenTelemetry).
4. Expose a `/metrics` endpoint in Prometheus format.
5. Expose `/health` (liveness) and `/health/ready` (readiness) endpoints.

### Step 10 — Graceful Shutdown
```
process.on('SIGTERM', async () => {
  // 1. Stop accepting new requests (close HTTP server)
  // 2. Wait for in-flight requests to complete (with a timeout)
  // 3. Drain job queues
  // 4. Close database connections
  // 5. Exit with code 0
});
```

## API Response Format

Adopt a consistent envelope format:

**Success:**
```json
{
  "success": true,
  "data": { ... },
  "meta": { "requestId": "...", "timestamp": "..." }
}
```

**Error:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description",
    "details": [ ... ],
    "requestId": "..."
  }
}
```

## Security Checklist
- [ ] All inputs validated and sanitized
- [ ] Parameterized queries used everywhere (no SQL injection)
- [ ] Authentication and authorization enforced on all protected endpoints
- [ ] Rate limiting applied to all endpoints
- [ ] Secrets loaded from environment, not hardcoded
- [ ] Security headers applied
- [ ] Dependencies scanned for known CVEs (`npm audit`, `pip-audit`, `snyk`)
- [ ] CORS configured correctly (whitelist specific origins, not `*` for authenticated APIs)

## Deliverables
- Fully functional REST API with consistent request/response format
- Structured project with clear separation of controllers, services, and repositories
- Input validation on all endpoints
- Authentication and authorization middleware
- Rate limiting on all endpoints
- Circuit breakers and timeouts for all external calls
- Centralized error handling with structured logging
- Health check and readiness endpoints
- Graceful shutdown implementation
- Background job queue setup if applicable
- Environment variable validation at startup
- Database migrations (versioned, tested in CI)
