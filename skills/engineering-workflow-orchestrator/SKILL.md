# Skill: Engineering Workflow Orchestrator

## Purpose
Act as the central intelligence that coordinates the complete software development lifecycle. This skill does not implement code directly — it reasons about the problem, selects the appropriate sub-skills, sequences work logically, detects risks early, and ensures that the output of each phase is verified before the next one begins.

Think of this skill as a Staff Engineer or Engineering Lead who has seen hundreds of projects succeed and fail, and knows exactly what to do, in what order, and when to stop and re-plan.

## When to Use
Invoke this skill as the entry point for every engineering task, regardless of size. Even a "simple" feature request benefits from structured thinking. This skill fires before any implementation begins.

## Operating Principles

1. **Understand before building**: Never begin writing code without a clear understanding of what success looks like and how it will be measured.
2. **Make decisions explicit**: Every significant architectural or implementation choice must be recorded as an Architecture Decision Record (ADR), not left implicit in the code.
3. **Fail fast on blockers**: Identify and surface blockers, ambiguities, and risks immediately. Do not proceed on assumptions that could invalidate the work.
4. **Verify continuously**: Each phase produces verifiable outputs. Do not advance to the next phase if the current phase's verification fails.
5. **Adapt, do not blindly execute**: If new information emerges during implementation that changes the understanding of the problem, pause and revise the plan before continuing.
6. **Optimize for reversibility**: Prefer decisions that can be changed later over ones that lock in an approach. The most dangerous decisions are the quiet ones made without documentation.
7. **Think about the second engineer**: Every artifact produced (code, docs, config) must be comprehensible to a different engineer who joins the project tomorrow.

## Execution Workflow

### Phase 0 — Task Classification
Before any work begins, classify the incoming task:

| Type | Description | Approach |
|------|-------------|----------|
| Greenfield project | Entirely new system | Full workflow from Phase 1 |
| New feature on existing system | Adding capability to a running product | Start from Phase 1, but leverage existing architecture decisions |
| Bug fix | Correcting unintended behavior | Use debugging skill, then targeted fix + regression test |
| Refactor | Improving structure without changing behavior | Characterization tests first, then refactor |
| Performance optimization | Improving speed/throughput/resource usage | Profile first, then targeted fix |
| Security fix | Addressing a vulnerability | Assess blast radius, implement fix, verify, then deploy with urgency |
| Incident / hotfix | Production issue requiring immediate action | Use incident-response skill; speed over process |

### Phase 1 — Requirements Analysis
**Invoke skill**: `analyze-requirements`

Execute the full requirements analysis procedure. Do not shortcut this phase.

**Gates before advancing:**
- [ ] All functional requirements are listed, prioritized, and unambiguous
- [ ] Non-functional requirements (performance, security, scalability, availability) are defined with measurable targets
- [ ] All assumptions are documented and flagged for validation
- [ ] All open questions are listed and either resolved or explicitly deferred
- [ ] Acceptance criteria are defined for every P0 requirement

**Adaptive decision point**: If analysis reveals that the problem is significantly larger or more complex than initially stated, pause and communicate scope to stakeholders before proceeding.

### Phase 2 — Architecture and Planning
**Invoke skill**: `architecture-and-planning`

Design the system architecture and create a concrete implementation plan.

**Gates before advancing:**
- [ ] Architecture Decision Records written for all significant choices
- [ ] System component diagram complete
- [ ] Database schema designed with indexing strategy
- [ ] API contracts defined (OpenAPI or equivalent)
- [ ] Security model defined (auth strategy, data classification, trust boundaries)
- [ ] Observability strategy defined (logging, metrics, tracing)
- [ ] Development roadmap established with milestones

**Adaptive decision point**: If a required technology is unfamiliar or its behavior is uncertain, schedule a time-boxed spike (max 2 days) to validate the approach before committing the full project to it.

### Phase 3 — Security Design
**Invoke skill**: `security-hardening` + `authentication-and-authorization`

Embed security into the design, not as an afterthought.

**Gates before advancing:**
- [ ] Authentication model selected and documented
- [ ] Authorization model (RBAC/ABAC) defined with explicit deny-by-default
- [ ] Sensitive data classified and encryption strategy defined
- [ ] OWASP Top 10 risks assessed for this application type
- [ ] Secrets management strategy defined

### Phase 4 — API Design
**Invoke skill**: `api-design`

Design the API contract before implementing it.

**Gates before advancing:**
- [ ] All endpoints defined with request/response schemas
- [ ] Error codes and error response format standardized
- [ ] Pagination strategy selected
- [ ] Versioning strategy applied
- [ ] Rate limiting strategy defined per endpoint category

### Phase 5 — Data Layer
**Invoke skill**: `data-management`

Design and implement the data persistence layer.

**Gates before advancing:**
- [ ] Schema migrations written and tested
- [ ] All query access patterns have appropriate indexes
- [ ] Caching strategy defined for read-heavy data
- [ ] Data retention and archival policies defined
- [ ] Backup and recovery procedure documented

### Phase 6 — Backend Implementation
**Invoke skill**: `backend-service-development`

Implement the backend service according to the architecture and API design.

**Gates before advancing:**
- [ ] All endpoints implemented and returning correct responses
- [ ] Input validation applied to all endpoints
- [ ] Authentication and authorization middleware active on protected routes
- [ ] Rate limiting applied
- [ ] Circuit breakers and timeouts on all external calls
- [ ] Health check and readiness endpoints implemented
- [ ] Structured logging with correlation IDs in place
- [ ] Graceful shutdown implemented

### Phase 7 — Frontend Implementation (if applicable)
**Invoke skill**: `frontend-application-development`

Build the user interface and integrate with the backend API.

**Gates before advancing:**
- [ ] All user journeys implemented
- [ ] API integration complete and error states handled
- [ ] Form validation implemented
- [ ] Accessibility requirements met (WCAG 2.1 AA minimum)
- [ ] Responsive design verified on target screen sizes
- [ ] Performance budget validated (Core Web Vitals)

### Phase 8 — Testing and Validation
**Invoke skill**: `testing-and-validation`

Verify that the implementation satisfies all acceptance criteria.

**Gates before advancing:**
- [ ] Unit test coverage ≥ target threshold for critical business logic
- [ ] Integration tests cover all API endpoints (happy path + error path)
- [ ] Authentication and authorization flows tested
- [ ] End-to-end tests cover all P0 user journeys
- [ ] Performance tests meet defined SLA targets
- [ ] Security tests pass (OWASP checklist)
- [ ] All acceptance criteria from Phase 1 have a corresponding test that passes

### Phase 9 — Code Quality Review
**Invoke skill**: `code-quality-and-review`

Ensure the code meets long-term maintainability standards.

**Gates before advancing:**
- [ ] No linter errors or warnings
- [ ] No TypeScript/type errors
- [ ] Dependency audit passed (no HIGH/CRITICAL CVEs)
- [ ] No hardcoded secrets
- [ ] Code complexity within acceptable limits
- [ ] TODO/FIXME items triaged and tracked

### Phase 10 — Observability
**Invoke skill**: `observability-and-monitoring`

Ensure the system can be operated and diagnosed in production.

**Gates before advancing:**
- [ ] Structured logging in place with appropriate log levels
- [ ] Key metrics exposed (request rate, error rate, latency, business metrics)
- [ ] Distributed tracing instrumented for cross-service calls
- [ ] Alerts configured for SLA-threatening conditions
- [ ] Dashboards created for operational visibility
- [ ] On-call runbook created

### Phase 11 — Infrastructure and Deployment
**Invoke skill**: `infrastructure-as-code` + `deployment-and-release`

Deploy the application safely and repeatably.

**Gates before advancing:**
- [ ] CI/CD pipeline configured and passing
- [ ] Docker image builds successfully and passes security scan
- [ ] Staging deployment verified (all smoke tests pass)
- [ ] Deployment strategy selected (rolling / blue-green / canary)
- [ ] Database migration procedure tested on staging
- [ ] Rollback procedure documented and tested
- [ ] Production deployment successful
- [ ] Post-deployment health checks pass
- [ ] Error rate and latency within expected bounds post-deploy

### Phase 12 — Documentation
**Invoke skill**: `project-documentation`

Produce the documentation that allows the next engineer to understand, operate, and extend the system.

**Deliverables:**
- [ ] README with setup instructions, architecture overview, and deployment guide
- [ ] API documentation (OpenAPI spec or equivalent)
- [ ] Architecture Decision Records finalized
- [ ] Environment variable reference (`.env.example` annotated)
- [ ] Runbook for common operational tasks
- [ ] CHANGELOG updated for this release

## Error Recovery and Adaptive Planning

### When a phase fails its gate
1. Do not advance to the next phase.
2. Identify the root cause of the failure.
3. Determine whether it requires a plan change (significant) or is a fixable implementation issue (minor).
4. For significant plan changes: re-execute the relevant upstream phases.
5. For fixable issues: fix the issue, re-run verification, and advance only when the gate passes.

### When a blocker is discovered mid-phase
1. Stop and document the blocker explicitly.
2. Assess whether work can continue in parallel while the blocker is resolved.
3. If no progress is possible, escalate and wait for resolution rather than making undocumented assumptions.

### When scope changes during development
1. Treat scope change as a new requirement, not an extension of the current task.
2. Re-execute Phase 1 for the new scope.
3. Assess impact on the existing architecture, timeline, and risk profile.
4. Do not silently absorb scope creep — make it explicit.

## Task-Specific Workflows

### Bug Fix Workflow
1. Reproduce the bug with a failing test before making any code changes.
2. Identify the root cause (do not fix symptoms).
3. Implement the minimal fix.
4. Verify the failing test now passes.
5. Check for similar bugs in adjacent code.
6. Add to regression suite.

### Refactoring Workflow
1. Write characterization tests for the code being refactored (capture current behavior).
2. Make the refactoring change.
3. Verify all characterization tests still pass.
4. Review for improved readability, reduced complexity, and better separation of concerns.
5. Update documentation if the interface changed.

### Performance Optimization Workflow
1. Define the performance target (specific, measurable: "p95 latency < 100ms under 1000 RPS").
2. Profile to identify the actual bottleneck — never optimize without data.
3. Implement one optimization at a time.
4. Benchmark before and after each change.
5. Stop when the target is met — over-optimization has its own costs.

### Security Fix Workflow
1. Assess severity and blast radius immediately.
2. Determine if the vulnerability is actively exploitable.
3. If actively exploitable: deploy a temporary mitigation before the full fix.
4. Implement the proper fix.
5. Write a regression test that would have caught this vulnerability.
6. Review adjacent code for similar vulnerabilities.
7. Document in a security advisory if applicable.

## Quality Standards (Non-Negotiable)

These standards apply to all work products regardless of timeline pressure:

- No secrets committed to version control — ever.
- No SQL string concatenation with user input — ever.
- No production deployment without a tested rollback path.
- No feature shipped without acceptance criteria being met.
- No external call without a timeout.
- No sensitive data in logs.
- No undocumented architecture decisions for significant choices.
