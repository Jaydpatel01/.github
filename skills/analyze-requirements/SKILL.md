---
name: analyze-requirements
description: Analyze requirements and decompose tasks before implementation begins. Use when given a task description, user story, feature request, change request, or project specification — regardless of apparent simplicity.
---

# Skill: Requirements Analysis and Task Decomposition

## Purpose
Provide a rigorous, senior-engineer-grade analysis of any incoming assignment, feature request, or project specification before a single line of implementation begins. This skill prevents costly re-work by ensuring complete understanding up front.

## When to Use
Invoke this skill as the very first step whenever a task description, user story, feature request, change request, or project brief is provided — regardless of apparent simplicity. Even a "small" change can hide non-obvious constraints.

## Objectives
- Establish a shared and unambiguous understanding of what needs to be built
- Identify functional requirements and separate them from assumptions
- Surface non-functional requirements (performance, security, scalability, availability)
- Detect hidden dependencies, risks, and constraints early
- Define acceptance criteria that can be turned into verifiable tests
- Decompose the work into independently deliverable engineering tasks
- Identify compliance, regulatory, or legal obligations
- Expose scope creep risks and create explicit boundaries (in-scope / out-of-scope)

## Procedure

### Phase 1 — Problem Understanding
1. Read the full specification at least twice before drawing any conclusions.
2. State back the problem in your own words to confirm understanding.
3. List every assumption made — never treat implied requirements as confirmed.
4. Identify and call out any ambiguity or contradiction; resolve it before proceeding.
5. Ask clarifying questions when critical information is missing (user scale, data volumes, SLA targets, compliance needs).

### Phase 2 — Stakeholder and Actor Mapping
1. Enumerate all human and system actors who interact with the solution.
2. Define the primary user persona and their goals.
3. Identify admin, operator, and third-party integration roles.
4. Map actor permissions and trust levels.

### Phase 3 — Functional Requirements
1. List every capability the system must provide (use "the system shall" phrasing).
2. Separate must-have (P0) from should-have (P1) and nice-to-have (P2) features.
3. Define input/output contracts for every major operation.
4. Document expected user journeys and workflows end-to-end.
5. Identify idempotency requirements (which operations must be safe to retry).

### Phase 4 — Non-Functional Requirements
1. **Performance**: Define latency targets (p50 / p95 / p99), throughput (requests per second), and batch processing windows.
2. **Scalability**: Identify expected user base now and at 10× growth; define horizontal vs. vertical scaling needs.
3. **Availability**: Define uptime SLA (e.g., 99.9% = ~8.7 h/year downtime). Define RTO and RPO.
4. **Security**: Identify PII/sensitive data, authentication requirements, authorization model, data-at-rest and in-transit encryption needs.
5. **Compliance**: Identify applicable regulations (GDPR, HIPAA, SOC 2, PCI-DSS, CCPA, etc.).
6. **Observability**: Define logging, metrics, and alerting requirements.
7. **Maintainability**: Define code quality standards, documentation expectations, and on-call requirements.
8. **Portability**: Define environment constraints (cloud provider, OS, runtime version locks).
9. **Cost**: Identify budget ceilings and cost-sensitive architectural choices.
10. **Accessibility**: Identify WCAG 2.1 compliance level requirement if UI is involved.

### Phase 5 — Data Modeling and Entity Identification
1. Identify all core domain entities and their attributes.
2. Define relationships between entities (one-to-one, one-to-many, many-to-many).
3. Identify data ownership and lifecycle (create, update, archive, delete).
4. Identify data that is write-heavy vs. read-heavy.
5. Identify time-series or append-only data patterns.
6. Identify data that requires strong consistency vs. eventual consistency.

### Phase 6 — Integration and Dependency Mapping
1. List all external systems, APIs, and services the solution depends on.
2. Document third-party SLAs and what happens when a dependency is unavailable.
3. Identify internal platform dependencies (shared databases, message buses, auth services).
4. Define data contracts for each integration point.

### Phase 7 — Risk and Constraint Identification
1. Identify technical risks (new technology, uncertain performance, tight deadlines).
2. Identify business risks (regulatory exposure, user data sensitivity, vendor lock-in).
3. Identify known constraints (deadline, budget, team skill set, existing system limitations).
4. Assess risk severity (High / Medium / Low) and propose mitigation strategies.

### Phase 8 — Acceptance Criteria Definition
1. Write at least one testable acceptance criterion per functional requirement.
2. Use Given / When / Then (BDD-style) format for behavioral criteria.
3. Define edge cases and failure scenarios explicitly.
4. Specify performance thresholds as acceptance criteria where applicable.

### Phase 9 — Task Decomposition
1. Group requirements into logical implementation milestones.
2. Break each milestone into discrete engineering tasks sized for 1–2 day delivery.
3. Identify critical path dependencies between tasks.
4. Flag tasks that can be parallelized by different team members.
5. Estimate relative complexity (S / M / L / XL) for each task.

## Output Format

The analysis must produce all of the following:

### Functional Requirements
Numbered list of system capabilities with priority (P0/P1/P2) and acceptance criteria.

### Non-Functional Requirements
Structured table covering: performance, scalability, availability, security, compliance, observability, maintainability, cost.

### Domain Model
List of key entities, their core attributes, and relationships.

### Application Workflows
Step-by-step description of the primary user journeys and system workflows, including happy paths and error paths.

### Integration Map
List of external and internal dependencies with their contracts and failure modes.

### Risk Register
Identified risks with severity, likelihood, and mitigation strategy.

### Scope Boundary
Explicit list of what is in-scope and what is explicitly out-of-scope for this iteration.

### Task Breakdown
Prioritized, dependency-ordered list of engineering tasks with size estimates.

### Open Questions
Numbered list of outstanding questions that must be resolved before or during implementation.

## Senior Engineering Principles to Apply
- Never build what was not explicitly required (YAGNI — You Aren't Gonna Need It).
- Prefer reversible decisions over irreversible ones when uncertain.
- Design for failure from day one — every external call can fail.
- Separate concerns: data storage, business logic, and presentation must be independently reasoned about.
- Make implicit requirements explicit before they become bugs in production.
