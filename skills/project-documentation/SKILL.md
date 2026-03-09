---
name: project-documentation
description: Produce accurate, immediately useful technical documentation. Use when writing READMEs, API reference docs, architecture decision records (ADRs), runbooks, onboarding guides, or any technical documentation for a project.
---

# Skill: Technical Project Documentation

## Purpose
Produce documentation that is honest, accurate, and immediately useful — not documentation that exists to look complete. Good documentation is the difference between a project that can be operated and extended by anyone, and one that only the original author understands.

## When to Use
Apply this skill at the end of each development milestone, not just at project completion. Documentation debt compounds just like technical debt.

## Core Principles
- Document decisions, not just outcomes. Future engineers need to know *why* a choice was made, not just *what* was chosen.
- Write for your most junior audience without dumbing down technical accuracy.
- Keep documentation close to the code — docs that are far from what they document drift and become wrong.
- Every documented procedure must be tested — if you cannot run it and verify it works, do not publish it.
- Prefer examples over abstract descriptions.

## Documentation Structure

### 1. README.md (Root)
The README is the front door to your project. It must answer these questions within the first 30 seconds of reading:

```markdown
# Project Name

One paragraph: what this project is and why it exists.

## Architecture Overview
Brief description of the major components and how they relate.
[Link to detailed architecture docs or diagram]

## Prerequisites
- Node.js >= 20
- Docker and Docker Compose
- PostgreSQL 15 (or use the provided docker-compose.yml)

## Quick Start (Local Development)
Step-by-step, copy-pasteable commands that work on a fresh checkout:

    git clone <repo>
    cd <project>
    cp .env.example .env          # Edit .env with your local values
    docker-compose up -d          # Start database and Redis
    npm install
    npm run db:migrate            # Run database migrations
    npm run dev                   # Start development server with hot reload

## Environment Variables
See .env.example for the full list with descriptions.
Required: DATABASE_URL, JWT_SECRET, ...

## Running Tests
    npm test            # Unit + integration tests
    npm run test:e2e    # End-to-end tests (requires running server)

## Deployment
See docs/deployment.md for the complete deployment guide.

## Contributing
See CONTRIBUTING.md.
```

### 2. Architecture Decision Records (`docs/adr/`)
Store ADRs as numbered markdown files: `docs/adr/001-title.md`

ADR Template:
```markdown
# ADR-001: [Short Title]

**Date**: YYYY-MM-DD
**Status**: Accepted | Superseded by ADR-XXX | Deprecated

## Context
What situation, constraint, or uncertainty required a decision?

## Decision
What was decided?

## Rationale
Why was this option chosen over the alternatives?

## Alternatives Considered
- **Alternative A**: Why it was rejected.
- **Alternative B**: Why it was rejected.

## Consequences
- Positive outcomes.
- Negative trade-offs or technical debt introduced.
- Follow-on decisions triggered.
```

### 3. API Documentation
- Maintain an OpenAPI 3.x spec in `docs/api/openapi.yaml` or auto-generate it from code annotations.
- Include: endpoint description, request parameters, request body schema, response schemas (all status codes), authentication requirement, example request/response pairs.
- Host interactive API docs using Swagger UI or Redoc.
- Document API versioning policy and supported versions.
- Document rate limits for each endpoint group.

### 4. Database Documentation (`docs/database.md`)
- Entity-Relationship diagram or schema description.
- Description of each table and its purpose.
- Explanation of non-obvious column names or constraints.
- Description of important indexes and why they exist.
- Description of any stored procedures, triggers, or materialized views.
- Migration history summary.

### 5. Environment Configuration Reference (`docs/configuration.md` or annotated `.env.example`)
Document every environment variable:
```
DATABASE_URL          # Required. PostgreSQL connection string. Example: postgres://user:pass@host:5432/db
JWT_SECRET            # Required. Min 32 chars. Rotate monthly in production.
JWT_EXPIRY            # Optional. Default: 15m. Access token TTL. Format: <number><unit> (15m, 1h, 7d)
REDIS_URL             # Required in production. Optional in development (falls back to in-memory).
SENTRY_DSN            # Optional. Enables error tracking. Omit to disable.
```

### 6. Deployment Guide (`docs/deployment.md`)
Covers each environment separately:
- **Staging**: how to deploy, how to verify, how to rollback
- **Production**: same, plus pre-deployment checklist and post-deployment verification
- Infrastructure requirements (CPU, memory, disk, network)
- Database migration procedure
- Secrets rotation procedure
- Rollback procedure (step-by-step)

### 7. Operational Runbook (`docs/runbook.md`)
Covers the most common operational scenarios:
- **Service is down**: How to check, diagnose, and restore.
- **High error rate**: How to identify which endpoint, how to triage.
- **Database is slow**: Query to find long-running queries, how to kill them.
- **Disk space running low**: Where to find large files, log rotation.
- **Scaling up**: How to add capacity, how to scale down.
- **Restarting the service**: Safe restart procedure without dropping in-flight requests.
- **Emergency rollback**: Step-by-step procedure.

### 8. CHANGELOG (`CHANGELOG.md`)
Follow the [Keep a Changelog](https://keepachangelog.com) format:
```markdown
# Changelog

## [Unreleased]

## [1.2.0] - 2025-01-15
### Added
- User MFA enrollment flow
- API rate limiting per user

### Changed
- Improved error messages for failed auth attempts

### Fixed
- Fixed session not invalidated on password change

### Security
- Upgraded dependencies to resolve CVE-2024-XXXX
```

### 9. Contributing Guide (`CONTRIBUTING.md`)
- Development environment setup (mirrors README Quick Start)
- Branching strategy (e.g., `feature/`, `fix/`, `chore/`)
- Pull request checklist (tests pass, linting passes, documentation updated, CHANGELOG entry)
- Code review expectations (response time SLA, what reviewers look for)
- Commit message format (Conventional Commits recommended)
- How to report security vulnerabilities (responsible disclosure)

### 10. Security Documentation (`docs/security.md`)
- Authentication mechanisms in use
- Authorization model and role definitions
- Data classification: which data is PII, which is sensitive, how each is protected
- Encryption at rest and in transit
- Known security limitations and compensating controls
- Incident response contact and procedure

## Documentation Quality Standards

Every document must pass this checklist before publishing:

- [ ] All commands in the document have been run by the author and produce the expected output.
- [ ] No placeholders left in the document (e.g., `TODO: fill this in`, `FIXME`).
- [ ] Dates, version numbers, and environment names are accurate.
- [ ] Code examples are syntactically valid and reflect the current codebase.
- [ ] Links are valid (no 404s).
- [ ] A developer who has never seen this project can follow this documentation to complete the stated goal.

## Deliverables
- `README.md` with Quick Start that works on a fresh clone
- Architecture Decision Records for all significant decisions
- OpenAPI spec for all API endpoints
- Database schema documentation
- Annotated `.env.example` with all environment variables
- Deployment guide for staging and production
- Operational runbook for common scenarios
- `CHANGELOG.md` following Keep a Changelog format
- `CONTRIBUTING.md` with branching and PR guidelines
- `docs/security.md` describing the security model
