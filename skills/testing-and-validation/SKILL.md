---
name: testing-and-validation
description: Build test suites that give the team confidence to ship quickly and safely. Use when writing unit tests, integration tests, end-to-end tests, setting up testing infrastructure, or increasing test coverage for existing code.
---

# Skill: Automated Testing and System Validation

## Purpose
Build a test suite that gives the team confidence to ship quickly and safely. Tests are not a bureaucratic requirement — they are the safety net that enables fearless refactoring and rapid iteration. An untested system is an unknown system.

## When to Use
Apply this skill continuously throughout development. Do not treat testing as a final phase — tests should be written in parallel with, or before (TDD), the code they validate.

## Core Principles
- **Test behavior, not implementation**: Tests should survive refactoring of internals. If renaming a private function breaks tests, the tests are testing the wrong thing.
- **Fast feedback loop**: Developers must be able to run the relevant test subset in under 30 seconds locally. A slow test suite is not run.
- **Tests as documentation**: A well-named test is the most accurate specification of intended behavior. Read the tests to understand what the code is supposed to do.
- **Test isolation**: Each test must set up its own state and not depend on the outcome of another test. Tests must run in any order.
- **Determinism**: Tests must produce the same result every time they run. Flaky tests are worse than no tests — they erode trust in the entire suite.

## Test Pyramid

```
            ▲
           /|\
          / | \
         /  E2E \        Few, slow, high-value user journey tests
        / -------- \
       / Integration \   Moderate number, test component interaction
      / ------------- \
     /   Unit Tests    \  Many, fast, test isolated logic
    /-------------------\
```

### Unit Tests (Foundation — fastest, most numerous)
**Test**: Individual functions, classes, and modules in isolation.
**Mock**: All I/O (database, HTTP, filesystem, time, random).
**Coverage target**: 80%+ line coverage for business logic, 100% for critical/complex algorithms.
**Speed target**: Entire unit test suite completes in < 30 seconds.

What to unit test:
- All business logic functions and service methods
- Validation and transformation functions
- Error handling paths
- Authorization logic (permission checks)
- Complex calculations and domain logic
- Edge cases: empty inputs, boundary values, null/undefined, maximum values

### Integration Tests (Middle layer)
**Test**: Multiple components working together, with real infrastructure (test database, test cache).
**Use**: Real database (not mocked), isolated test database that is reset between test runs.
**Test**: All API endpoints (request → middleware → controller → service → repository → response).

What to integration test:
- All REST API endpoints: happy path + every defined error path
- Authentication and authorization flows
- Database read and write operations
- Message queue publish and consume cycles
- Third-party integrations (against sandbox/mock environments)
- Database migration correctness

### End-to-End Tests (Apex — slowest, highest confidence)
**Test**: The full system from the user's perspective, through the real browser against a running application.
**Tool**: Playwright (preferred) or Cypress.
**Target**: Cover all P0 user journeys only — not every edge case (that is unit/integration test territory).

What to E2E test:
- User registration and login flow
- Core business workflow (the main thing the application does)
- Payment or checkout flow (if applicable)
- Email/notification delivery (use a test mailbox like Mailhog)
- Admin critical paths

## Testing Stack by Platform

### Node.js / TypeScript Backend
- **Test runner**: Vitest (preferred — fast, native ESM, Jest-compatible API) or Jest
- **HTTP testing**: Supertest for integration tests against the Express/Fastify app
- **Database**: Test database seeded via factories/fixtures; reset between tests with transactions
- **Mocking**: `vi.mock()` (Vitest) or `jest.mock()` for unit tests; avoid mocking in integration tests
- **Assertions**: Built-in `expect` with `@types/jest` matchers

### Python Backend
- **Test runner**: Pytest
- **HTTP testing**: `httpx.AsyncClient` with `fastapi.testclient.TestClient`
- **Database**: Test database with `pytest-asyncio` and transaction rollback per test
- **Mocking**: `unittest.mock.patch` or `pytest-mock`
- **Fixtures**: Pytest fixtures for test data setup

### React / Vue Frontend
- **Unit/Integration**: Vitest + Testing Library (`@testing-library/react` or `@testing-library/vue`)
- **E2E**: Playwright
- **Accessibility**: `jest-axe` in component tests
- **Visual regression**: Chromatic or Percy (for design systems)

## Test Writing Guidelines

### Naming
Use the format: `it should <expected behavior> when <condition>`:
```
it should return 401 when the request has no Authorization header
it should send a welcome email when a new user registers
it should not allow a viewer to delete a post authored by another user
```

### AAA Structure (Arrange, Act, Assert)
```javascript
it('should return 404 when user does not exist', async () => {
  // Arrange
  const nonExistentUserId = 'user_does_not_exist';

  // Act
  const response = await request(app)
    .get(`/api/v1/users/${nonExistentUserId}`)
    .set('Authorization', `Bearer ${validAdminToken}`);

  // Assert
  expect(response.status).toBe(404);
  expect(response.body.error.code).toBe('USER_NOT_FOUND');
});
```

### Test Data Management
- Use factories (not fixtures) to create test data — factories keep tests independent.
- Use libraries like `fishery`, `factory-bot`, or custom factory functions.
- Never share mutable state between tests.
- Reset all state (database, Redis, mock call counts) between tests.

## Specialized Test Types

### Contract Testing (for microservices)
- Ensure that service A's expectations of service B's API match what service B actually provides.
- Use **Pact** for consumer-driven contract testing.
- Run contract verification in CI for both consumer and provider.
- Prevents integration failures from being discovered only in staging.

### Load and Performance Testing
**When to run**: Before major launches, after significant architecture changes, when SLAs are defined.
**Tools**: k6 (preferred), Artillery, Locust (Python).

```javascript
// k6 example: verify p95 < 200ms at 100 concurrent users
export let options = {
  vus: 100,
  duration: '60s',
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],  // < 1% error rate
  },
};
```

Scenarios to load test:
- Sustained normal load (expected peak throughput for 10 minutes)
- Burst load (2× expected peak for 2 minutes)
- Soak test (expected load for 2+ hours — catches memory leaks)
- Spike test (sudden 10× traffic spike)

### Security Testing
1. Run `npm audit` / `pip-audit` / `snyk test` in CI — fail on HIGH or CRITICAL CVEs.
2. Run OWASP ZAP or Burp Suite scanner against the staging environment.
3. Test for OWASP Top 10 manually: SQL injection, XSS, IDOR, auth bypass, SSRF.
4. Verify that authentication and authorization behave correctly for all roles.
5. Verify that all auth-bypass edge cases are tested: expired tokens, tampered tokens, missing tokens, tokens for wrong service.

### Chaos Engineering (for production-critical services)
1. Test system behavior when the database is unavailable.
2. Test system behavior when a downstream API times out.
3. Test system behavior when Redis is unavailable.
4. Test graceful degradation: what does the user see when a non-critical service is down?
5. Tools: Chaos Monkey, Gremlin, or manual `tc netem` network fault injection.

### Mutation Testing (advanced — validates test quality)
- Run the code through a mutation testing tool that introduces small bugs.
- A good test suite must catch these mutations (kill the mutants).
- Tools: **Stryker** (JavaScript/TypeScript), **mutmut** (Python), **PIT** (Java).
- Run on critical business logic modules to validate that tests are not just achieving coverage without actually catching bugs.

## CI Integration

Every pull request must pass:
```yaml
# Required CI checks
- lint              # ESLint + Prettier
- type-check        # tsc --noEmit
- test:unit         # Fast; must complete in < 2 minutes
- test:integration  # Medium; must complete in < 10 minutes
- security-audit    # npm audit / pip-audit
```

Merge to main also runs:
```yaml
- test:e2e          # Against deployed staging environment
- load-test:smoke   # 1-minute load test at expected normal load
```

## Coverage Reporting
- Track coverage trends in CI — alert if coverage drops below threshold.
- Do not chase 100% coverage blindly. Focus on meaningful coverage of business logic.
- Exclude from coverage: auto-generated code, configuration files, type definitions.
- Minimum thresholds (adjust per project):
  - Lines: 80%
  - Branches: 75%
  - Functions: 85%

## Deliverables
- Unit tests covering all business logic and edge cases
- Integration tests for all API endpoints (happy path + error paths)
- Authentication and authorization flow tests
- E2E tests for all P0 user journeys
- Load test scripts with defined SLA thresholds
- Security scan passing in CI
- Test data factories for clean test setup
- CI configuration running tests on every PR
- Coverage report with thresholds enforced
- Testing strategy document explaining what is tested at which layer and why
