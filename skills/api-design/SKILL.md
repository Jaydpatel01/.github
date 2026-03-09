---
name: api-design
description: Design production-grade APIs that are consistent, versioned, and safe to evolve. Use before implementing any API endpoint, REST resource, GraphQL schema, or service interface.
---

# Skill: API Design

## Purpose
Design APIs that are a pleasure to use, easy to understand, and safe to evolve. A well-designed API is a product in itself. A poorly designed API creates technical debt for every consumer of it — and API changes are among the hardest to make without breaking things.

## When to Use
Apply this skill before implementing any API. Design the contract first, validate it with stakeholders and consumers, then implement. Never implement first and document later.

## Core Principles
- **Consistency is more important than perfection.** An API with consistently applied conventions (even imperfect ones) is far easier to consume than an inconsistent one.
- **Design for the consumer.** The API exists to serve its consumers, not to mirror internal implementation details.
- **Be explicit about semantics.** The naming of endpoints, fields, and error codes should leave no ambiguity about what they do.
- **Stable contracts.** Breaking changes to a public API are a significant cost to consumers. Prefer additive changes; version for breaking ones.
- **Documentation is part of the API.** An undocumented API does not exist from the consumer's perspective.

## API Style Selection

### REST (Representational State Transfer)
**Use when**: resource-oriented CRUD operations, public APIs, HTTP caching benefits are needed, standard client libraries suffice.
- Resources are nouns; HTTP methods express the action.
- Stateless: each request contains all context needed to process it.
- Cacheable: GET responses should be cacheable by default.
- Uniform interface: consistent URL patterns and HTTP semantics.

### GraphQL
**Use when**: multiple clients with different data needs (mobile app vs. web app), avoiding over-fetching/under-fetching, complex entity graphs that vary in query depth per consumer.
- Flexible query language; clients request exactly what they need.
- Single endpoint; schema defines the entire API.
- Caution: N+1 query problem requires DataLoader pattern; authorization is more complex; caching requires extra work.

### gRPC
**Use when**: internal service-to-service communication, low latency requirements, streaming data, multi-language polyglot environments.
- Protocol Buffers (binary serialization): compact and fast.
- Strongly typed contracts via `.proto` files.
- Supports unary, server-streaming, client-streaming, and bidirectional streaming.
- Not suitable for direct browser consumption without a proxy layer (gRPC-Web).

### WebSocket / Server-Sent Events
**Use when**: real-time bidirectional communication (WebSocket) or server push (SSE).
- WebSocket: chat, collaborative editing, live game state.
- SSE: live dashboards, notification feeds, progress updates. Simpler than WebSocket; HTTP-compatible.

## REST API Design Standards

### URL Design
Resources are **nouns**, not verbs:

```
✅ GET    /api/v1/users
✅ POST   /api/v1/users
✅ GET    /api/v1/users/{userId}
✅ PATCH  /api/v1/users/{userId}
✅ DELETE /api/v1/users/{userId}
✅ GET    /api/v1/users/{userId}/orders

❌ GET    /api/v1/getUsers
❌ POST   /api/v1/createUser
❌ POST   /api/v1/deleteUser/{userId}
```

URL conventions:
- Use lowercase, hyphen-separated path segments: `/user-profiles`, not `/userProfiles` or `/user_profiles`
- Plural resource names: `/orders`, `/users`, `/products`
- Hierarchical relationships: `/users/{userId}/orders/{orderId}`
- Do not exceed 3 levels of nesting — flatten deeper relationships

### HTTP Methods — Use Them Correctly
| Method | Semantics | Idempotent | Safe | Body |
|--------|-----------|-----------|------|------|
| GET | Retrieve resource | Yes | Yes | No |
| POST | Create resource or trigger action | No | No | Yes |
| PUT | Replace resource entirely | Yes | No | Yes |
| PATCH | Partially update resource | No | No | Yes |
| DELETE | Remove resource | Yes | No | Optional |
| HEAD | Same as GET but no body | Yes | Yes | No |
| OPTIONS | Describe capabilities (CORS preflight) | Yes | Yes | No |

### HTTP Status Codes — Use Them Precisely

**2xx Success:**
- `200 OK`: successful GET, PATCH, or DELETE with a response body
- `201 Created`: successful resource creation (include `Location` header pointing to new resource)
- `202 Accepted`: request accepted for async processing; not yet complete
- `204 No Content`: successful operation with no response body (common for DELETE)

**3xx Redirection:**
- `301 Moved Permanently`: resource URL has permanently changed
- `304 Not Modified`: conditional GET; client cache is still valid

**4xx Client Errors:**
- `400 Bad Request`: malformed request syntax, invalid parameters, missing required fields
- `401 Unauthorized`: authentication required or authentication failed
- `403 Forbidden`: authenticated but not authorized to perform this action
- `404 Not Found`: resource does not exist
- `405 Method Not Allowed`: HTTP method not supported for this endpoint
- `409 Conflict`: state conflict (e.g., duplicate unique field, optimistic lock violation)
- `410 Gone`: resource existed but has been permanently deleted
- `422 Unprocessable Entity`: syntactically valid but semantically invalid (business rule violation)
- `429 Too Many Requests`: rate limit exceeded (include `Retry-After` header)

**5xx Server Errors:**
- `500 Internal Server Error`: unexpected server-side error
- `502 Bad Gateway`: upstream service returned invalid response
- `503 Service Unavailable`: server cannot handle the request (overloaded, maintenance); include `Retry-After` header
- `504 Gateway Timeout`: upstream service timed out

### Consistent Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields failed validation",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Must be between 18 and 120"
      }
    ],
    "requestId": "req_abc123xyz",
    "documentationUrl": "https://docs.example.com/errors/VALIDATION_ERROR"
  }
}
```

Error code conventions:
- Use `SCREAMING_SNAKE_CASE` for machine-readable error codes
- Codes should be stable — consumers may match on them
- Include a `requestId` for every error response to enable cross-system debugging
- Optionally include a link to documentation for the error

### Consistent Success Response Format
```json
// Single resource
{
  "data": {
    "id": "usr_abc123",
    "email": "user@example.com",
    "createdAt": "2025-01-15T14:23:11Z"
  },
  "meta": {
    "requestId": "req_xyz789"
  }
}

// Collection
{
  "data": [...],
  "meta": {
    "requestId": "req_xyz789",
    "pagination": {
      "cursor": "eyJpZCI6MjB9",
      "hasNextPage": true,
      "totalCount": 142
    }
  }
}
```

## Pagination

### Cursor-Based Pagination (preferred)
```
GET /api/v1/orders?limit=20&after=eyJpZCI6MjB9

Response:
{
  "data": [...],
  "meta": {
    "pagination": {
      "cursor": "eyJpZCI6NDB9",
      "hasNextPage": true
    }
  }
}
```
Advantages: O(log n) performance; stable under concurrent writes; works for infinite scroll.

### Offset-Based Pagination (simple but limited)
```
GET /api/v1/orders?page=3&pageSize=20
```
Disadvantages: O(n) database cost at high offsets; unstable under concurrent writes (items can skip or duplicate).

Use offset pagination only for small datasets or admin interfaces. Use cursor pagination for everything else.

## API Versioning

Version from day one — adding versioning retroactively is painful.

### URL Versioning (recommended)
```
/api/v1/users
/api/v2/users
```
Pros: explicit, easy to route, easy to cache, easy to test.

### Header Versioning
```
Accept: application/vnd.myapi.v2+json
```
Pros: clean URLs. Cons: harder to test in browsers, cache requires `Vary` header.

### Versioning Policy
- Maintain at least one previous major version after releasing a new one.
- Announce deprecation at least 6 months before removing a version.
- Return `Deprecation` and `Sunset` headers on deprecated endpoints.
- Never make breaking changes in a minor version.

**Breaking changes** (require a new major version):
- Removing or renaming a field
- Changing a field's type
- Changing HTTP method for an endpoint
- Adding a required field to a request body
- Changing error codes
- Removing an endpoint

**Non-breaking changes** (safe in the same version):
- Adding optional fields to requests
- Adding new fields to responses
- Adding new endpoints
- Adding new optional query parameters
- Adding new values to an enum (use with caution — consumers may switch on enum values)

## Rate Limiting

Apply rate limits to protect the API from abuse and overload.

### Algorithms
- **Token Bucket**: allows burst capacity; smooth rate over time. Preferred.
- **Sliding Window**: accurate; prevents boundary exploitation of fixed windows.
- **Fixed Window**: simple to implement; susceptible to boundary bursts.

### Headers to Return
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1705327200
Retry-After: 3600          (on 429 response)
```

### Rate Limit Tiers
```
Anonymous requests:    100 req/hour per IP
Authenticated users:   1,000 req/hour per user
Write operations:      100 req/minute per user
Auth endpoints:        5 req/minute per IP (strict)
Bulk operations:       10 req/minute per user
```

## Idempotency

Idempotency prevents duplicate operations when requests are retried.

```
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

# The same payment will not be processed twice for the same Idempotency-Key
# within a 24-hour window.
```

Apply idempotency keys to:
- Payment operations
- Order creation
- Email sending
- Any operation with financial or physical side effects that cannot be undone

Implementation: store the `(key, response)` pair in Redis/DB with a TTL; return the cached response for duplicate requests.

## Request and Response Design

### Request Bodies
- Accept `application/json` as default.
- Ignore unknown fields (future-proof deserialization).
- Require explicit opt-in for destructive operations (confirmation fields, reason codes).

### Response Fields
- Use `camelCase` for JSON field names.
- Use ISO 8601 format for all dates and times: `2025-01-15T14:23:11Z` (always UTC).
- Use string representation for large integers (> Number.MAX_SAFE_INTEGER).
- Return amounts in smallest currency unit (cents, pence) as integers — never floating-point for money.
- Never return `null` for collections — return an empty array `[]`.
- Omit `null` optional fields rather than including them as `null` (reduces payload size).

### Filtering, Sorting, and Search
```
GET /api/v1/orders?status=pending&userId=usr_abc&sort=-createdAt&limit=20
GET /api/v1/products?minPrice=10&maxPrice=100&category=electronics
GET /api/v1/users?search=john&fields=id,email,name
```
Conventions:
- Prefix sort field with `-` for descending order, no prefix for ascending.
- `fields` parameter for sparse fieldsets (return only requested fields).
- Use range parameters for numeric and date ranges.

## API Documentation

### OpenAPI 3.x Specification
- Write the OpenAPI spec before (or alongside) implementation.
- Include for every endpoint: summary, description, security requirements, all parameters (path, query, header), request body schema, all response schemas (including errors), example request and response.
- Validate the spec with `redocly lint` or `spectral` in CI.
- Host interactive documentation with Redoc or Swagger UI.

### Developer Experience
- Provide a getting-started guide with working curl/code examples.
- Provide an SDK or code generation guide (OpenAPI Generator).
- Provide a Postman/Insomnia collection.
- Provide a changelog with all API changes (breaking and non-breaking).
- Provide a status page showing API uptime and current issues.

## API Security

- Require HTTPS for all endpoints; reject HTTP connections.
- Validate `Content-Type: application/json` header on all request bodies.
- Set `Content-Type: application/json` on all JSON responses.
- Apply CORS correctly: whitelist specific origins; do not use `*` for authenticated APIs.
- Strip sensitive response headers: `Server`, `X-Powered-By`.
- Validate all IDs for ownership (IDOR prevention).
- Apply input validation on every endpoint (see `security-hardening` skill).

## Deliverables
- OpenAPI 3.x specification for all endpoints
- Consistent URL structure and naming conventions applied
- Consistent error response format
- Consistent success response format
- Pagination implemented (cursor-based for collections)
- API versioning applied (`/api/v1/`)
- Rate limiting configured per endpoint category
- Idempotency keys for non-idempotent write operations
- Interactive API documentation hosted
- API changelog maintained
