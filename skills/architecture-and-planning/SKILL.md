---
name: architecture-and-planning
description: Design scalable, maintainable system architecture and project planning before implementation. Use when starting a new project, major feature, or when making significant architectural decisions.
---

# Skill: System Architecture and Project Planning

## Purpose
Design a production-grade, scalable, maintainable architecture before implementation begins. Architecture decisions made early are the hardest to change later — invest the time to get them right.

## When to Use
Use this skill after requirements analysis is complete and before any implementation starts. Revisit this skill when requirements change significantly or when scaling challenges emerge.

## Objectives
- Select the appropriate architectural pattern for the problem at hand
- Choose a technology stack aligned with team skills, project constraints, and long-term maintainability
- Define system components, service boundaries, and data flows
- Produce Architecture Decision Records (ADRs) for all non-trivial decisions
- Establish folder structure, module boundaries, and coding conventions
- Design the data model and API contracts up front
- Plan for observability, security, and operational concerns from day one

## Architecture Pattern Selection

### Choose the right pattern for the context

**Monolith (Modular)**
- Best for: early-stage products, small teams, unclear service boundaries, time-constrained MVPs
- Avoid when: team size > 8 engineers, independent scaling needed per component, multiple tech stacks required

**Microservices**
- Best for: large teams, independent deployment cadences, heterogeneous scaling needs
- Avoid when: distributed systems complexity is not justified by scale; prefer starting with a well-structured monolith and extracting services only when necessary
- Warning: Premature microservices are a primary cause of project failure. Earn microservices by first proving the service boundary with a modular monolith.

**Event-Driven Architecture**
- Best for: decoupled integrations, audit trails, real-time data pipelines, workflows with long processing times
- Key components: message broker (Kafka, RabbitMQ, AWS SQS/SNS), event schema registry, dead letter queues

**CQRS + Event Sourcing**
- Best for: audit-heavy domains, complex state transitions, reporting that diverges from transactional model
- Avoid when: the added complexity cannot be justified by actual requirements

**Serverless / Functions-as-a-Service**
- Best for: event-driven tasks, irregular workloads, minimal operational overhead requirement
- Avoid when: cold start latency is unacceptable, or long-running processes are needed

## Technology Stack Guidelines

### Selecting a Stack
Evaluate every technology choice against these criteria:
1. **Fitness**: Does it solve the problem well?
2. **Maturity**: Is it production-proven? What is the community and support story?
3. **Team familiarity**: Can the team maintain and debug it under pressure?
4. **Operational cost**: What is the cost to run, monitor, and upgrade?
5. **Escape hatch**: How hard is it to migrate away if needed?

### Reference Stack Patterns

**Full-Stack Web Application (General Purpose)**
- Frontend: React 18+ or Vue 3+ with TypeScript, Tailwind CSS, Vite
- Backend: Node.js (Express / Fastify / NestJS) or Python (FastAPI)
- Database: PostgreSQL (primary), Redis (cache / sessions / queues)
- Auth: OAuth 2.0 + OIDC or JWT with secure refresh token rotation
- Infrastructure: Docker + Docker Compose (local), Kubernetes or managed container service (production)
- Deployment: Vercel / Netlify (frontend), Render / Railway / Fly.io / AWS ECS (backend)
- CI/CD: GitHub Actions

**Data-Intensive / Analytics Platform**
- Ingestion: Kafka or AWS Kinesis
- Processing: Apache Flink, Spark, or Beam
- Storage: PostgreSQL (OLTP), ClickHouse or Redshift (OLAP), S3 (data lake)
- Orchestration: Airflow or Prefect

**High-Throughput API Service**
- Language: Go or Node.js (Fastify)
- Database: PostgreSQL with PgBouncer connection pooling; Redis for hot data
- Rate limiting: Redis-backed token bucket or sliding window
- Caching: Redis L1 cache, CDN for static/cacheable responses

## Design Process

### Step 1 — Architecture Decision Records (ADRs)
For every significant non-trivial decision, write an ADR containing:
- Context: what is the situation forcing a decision?
- Decision: what was chosen?
- Rationale: why was it chosen over alternatives?
- Consequences: what are the trade-offs, risks, and follow-on decisions?

Store ADRs in `docs/adr/` as numbered markdown files (e.g., `001-use-postgresql.md`).

### Step 2 — System Component Diagram
1. Draw the high-level component diagram showing all services, databases, caches, queues, and external integrations.
2. Label every synchronous call (HTTP/gRPC) and asynchronous event flow.
3. Identify every trust boundary (public internet, internal network, private subnet).
4. Identify where TLS termination happens and what is encrypted at rest.

### Step 3 — Service Boundary Definition
1. Define each service/module by the data it owns (do not share databases between services).
2. Define the public API contract of each service (inputs, outputs, error codes).
3. Identify shared libraries vs. services (prefer libraries for pure logic with no I/O).
4. Apply the single-responsibility principle at the service level.

### Step 4 — Data Architecture
1. Design the relational or document schema for each service.
2. Define indexes for all query patterns (never rely on full-table scans in production).
3. Identify read-heavy vs. write-heavy tables and apply appropriate caching strategy.
4. Plan database migrations strategy (forward-only, versioned, tested in CI).
5. Define data retention policies and archival strategy.
6. Identify whether you need a read replica for reporting/analytics queries.

### Step 5 — API Contract Design
1. Define all REST or GraphQL endpoints with request/response schemas before implementation.
2. Use OpenAPI 3.x (Swagger) for REST APIs.
3. Adopt API versioning from day one (URL versioning: `/api/v1/...`).
4. Define pagination strategy (cursor-based preferred over offset for large datasets).
5. Define error response format consistently across all endpoints.

### Step 6 — Cross-Cutting Concerns
Plan the following before writing a single feature:
- **Authentication and Authorization**: which endpoints are public, which require auth, what RBAC/ABAC model is used
- **Logging**: structured JSON logs, correlation IDs, log levels
- **Metrics**: what counters, gauges, and histograms are needed
- **Distributed tracing**: trace context propagation across service calls
- **Rate limiting**: per-user, per-IP, or per-API-key limits
- **Error handling**: consistent error codes, graceful degradation, circuit breakers
- **Feature flags**: how to safely roll out new behavior without deploying new code
- **Configuration management**: environment variables, secrets management (Vault, AWS Secrets Manager, etc.)

### Step 7 — Folder Structure
Establish a consistent folder structure that enforces separation of concerns:

```
project-root/
├── src/
│   ├── api/           # Route handlers / controllers
│   ├── services/      # Business logic (no HTTP/DB knowledge)
│   ├── repositories/  # Database access layer
│   ├── models/        # Data models and DTOs
│   ├── middleware/    # Auth, logging, rate limiting, error handling
│   ├── config/        # Configuration loading and validation
│   ├── events/        # Event publishers and subscribers
│   └── utils/         # Pure utility functions
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
│   └── adr/
├── infra/             # Terraform / Kubernetes / Docker
├── scripts/           # Developer tooling scripts
├── .github/
│   └── workflows/     # CI/CD pipelines
└── docker-compose.yml
```

### Step 8 — Development Roadmap
1. Define milestones aligned with independently deliverable value.
2. Sequence milestones to reduce risk: build the hardest/most uncertain parts first.
3. Identify what can be built in parallel and what is on the critical path.
4. Define a spike/prototype phase for any genuinely unknown technical components.

## Operational Readiness Planning
Explicitly plan for:
- **Health checks**: liveness and readiness probes for every service
- **Graceful shutdown**: in-flight request draining on SIGTERM
- **Startup validation**: fail fast if required config or dependencies are unavailable
- **Dependency failure**: define behavior when database, cache, or third-party API is unavailable
- **Runbook stub**: document how to restart, scale, and diagnose the system

## Deliverables
- Architecture Decision Records for major choices
- System component diagram with trust boundaries labeled
- Service/module boundary definitions with owned data
- Database schema with indexing strategy
- API contract definitions (OpenAPI spec or equivalent)
- Project folder structure with module responsibility descriptions
- Cross-cutting concerns implementation plan
- Development roadmap with milestones, critical path, and parallelizable work
- Operational readiness checklist
