# Skill: Application Deployment and Release

## Purpose
Ship software to production reliably, safely, and repeatably. Deployment is not an afterthought — the deployment process is part of the product. Manual, undocumented deployments are a liability.

## When to Use
Apply this skill when setting up deployment infrastructure for the first time, when automating a previously manual deployment, when establishing a CI/CD pipeline, or when preparing a release strategy for a new feature.

## Core Principles
- **Everything as code**: infrastructure, configuration, and deployment pipelines must be version-controlled.
- **Repeatable**: any environment (dev / staging / production) must be reproducible from the repository.
- **Automated**: no manual deployment steps in production. If it is not automated, it will be done wrong eventually.
- **Observable**: you cannot call a deployment successful unless you can see the application is healthy afterward.
- **Reversible**: every deployment must have a tested rollback path.
- **Parity**: dev, staging, and production must be as identical as possible to eliminate "works on my machine" failures.

## Deployment Environments

Define at minimum these three environments:

| Environment | Purpose | Deployment Trigger |
|-------------|---------|-------------------|
| Development | Local developer iteration | Manual / docker-compose |
| Staging | Pre-production integration and QA | Merge to `main` or `staging` branch |
| Production | Live users | Tagged release or manual promotion from staging |

Each environment must have:
- Its own isolated database instance
- Its own set of secrets and environment variables
- Its own monitoring and alerting configuration

## CI/CD Pipeline Design

### Continuous Integration (every push / PR)
```
Trigger: Push to any branch or PR opened/updated

Jobs (run in parallel where possible):
1. lint          — ESLint, Prettier, type check (tsc --noEmit)
2. test          — Unit tests + integration tests with test database
3. build         — Production build artifact compilation
4. security-scan — npm audit / pip-audit / Snyk; fail on HIGH or CRITICAL CVEs
5. docker-build  — Build Docker image (no push, just validate it builds)

Gate: All jobs must pass before merging to main
```

### Continuous Deployment (merge to main → staging)
```
Trigger: Merge to main branch

Jobs (sequential):
1. Run full CI pipeline
2. Build and push Docker image tagged with commit SHA
3. Deploy to staging environment
4. Run smoke tests against staging (key endpoint health checks)
5. Notify team of successful / failed deployment (Slack, email)
```

### Production Release
```
Trigger: Git tag pushed (e.g., v1.2.3) or manual approval

Jobs:
1. Promote staging image to production (no rebuild — same SHA)
2. Apply database migrations (with rollback on failure)
3. Deploy using rolling or blue-green strategy
4. Run post-deployment health checks
5. Monitor error rate for 10 minutes (automated rollback on threshold breach)
6. Update release notes and notify stakeholders
```

## Containerization

### Dockerfile Best Practices
```dockerfile
# Use specific version tags — never "latest"
FROM node:20.11-alpine AS builder

WORKDIR /app

# Copy dependency manifests first (layer caching)
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# --- Minimal production image ---
FROM node:20.11-alpine AS production

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER appuser

# Health check built into the image
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Docker Compose (Local Development)
- Mirror production services locally: app, database, Redis, queue.
- Use named volumes for database data persistence.
- Use `.env` file for local configuration (never committed).
- Include a separate `docker-compose.test.yml` for integration test isolation.

## Deployment Strategies

### Rolling Deployment (Default for most services)
- Replace instances one at a time while maintaining availability.
- New version and old version run simultaneously during rollout.
- Requires backward-compatible API and database changes.
- Suitable for stateless services.

### Blue-Green Deployment (Zero-downtime, easy rollback)
- Maintain two identical environments (blue = current, green = new).
- Deploy entirely to green, run smoke tests, then switch traffic.
- Rollback = switch traffic back to blue (seconds, not minutes).
- Higher infrastructure cost; justified for high-availability systems.

### Canary Release (Risk reduction for large user bases)
- Route a small percentage (1–5%) of production traffic to the new version.
- Monitor error rates, latency, and business metrics.
- Gradually increase traffic if metrics are healthy.
- Immediately roll back if any metric degrades beyond threshold.

### Feature Flags (Decouple deploy from release)
- Deploy code to production in a disabled state.
- Enable the feature incrementally without redeployment.
- Roll back by disabling the flag, not by reverting code.
- Tools: LaunchDarkly, Unleash, Flagsmith, custom Redis-backed implementation.

## Database Migration Strategy

Migrations are the riskiest part of deployment. Apply these rules:

1. **Backward compatible first**: all schema changes must be backward compatible with the currently deployed code.
2. **Expand-Contract pattern**: add new column → deploy app using new column → remove old column in a subsequent deployment.
3. **Never drop columns or tables in the same migration that adds replacements** — always two-phase.
4. **Test migrations against production data volume** in staging before production deployment.
5. **Migration rollback plan**: document the reverse migration and test it.
6. **Run migrations before deploying new code** (not after) so old code can read new schema.

## Health Checks

Every service must expose:

**Liveness probe** (`GET /health`)
- Returns 200 OK if the process is alive.
- Does not check dependencies (if dependencies are down, the process may still be healthy).

**Readiness probe** (`GET /health/ready`)
- Returns 200 OK only when the service can handle traffic.
- Checks: database connectivity, cache connectivity, required config loaded.
- Used by load balancers and Kubernetes to route traffic.

**Startup probe** (Kubernetes only)
- Used for slow-starting services.
- Prevents premature liveness failures during startup.

## Post-Deployment Validation

Always validate after every deployment:

1. **Smoke tests**: automated tests against live endpoints (critical paths only, fast).
2. **Health check polling**: confirm all instances report healthy within 5 minutes.
3. **Error rate monitoring**: confirm error rate has not increased above baseline.
4. **Latency monitoring**: confirm p95 response time has not regressed.
5. **Business metric check**: confirm key business metrics (signups, orders, etc.) are within normal range.
6. **Log inspection**: scan logs for unexpected error patterns in the first 10 minutes.

## Rollback Procedure

**Automated rollback triggers:**
- Error rate > threshold (e.g., 1% 5xx errors over a 5-minute window)
- Health check failures for > 2 minutes
- Post-deployment smoke tests fail

**Manual rollback steps (documented in runbook):**
1. Identify the last known good Docker image tag.
2. Trigger deployment of the previous image (no code change required).
3. Verify health checks pass for previous version.
4. If database migration was applied, assess whether it needs reverting (prefer forward-fix over rollback for migrations).
5. Notify team and create incident report.

## Infrastructure Platforms

### Frontend
- **Vercel**: Best for Next.js; excellent DX; automatic preview deployments per PR.
- **Netlify**: Strong for static sites and Jamstack; built-in forms and edge functions.
- **Cloudflare Pages**: Best performance globally; excellent for static and edge-rendered sites.

### Backend API
- **Render**: Simple managed deployments; auto-deploy on push; free tier available.
- **Railway**: Fast setup; good for hobbyist to small-team projects.
- **Fly.io**: Global edge deployment; excellent for low-latency services; supports Docker.
- **AWS ECS / Fargate**: Production-grade; no server management; integrates with full AWS ecosystem.
- **Google Cloud Run**: Serverless containers; scales to zero; pay per request.
- **Kubernetes (self-managed or GKE/EKS/AKS)**: Full control; justified for complex, large-scale systems.

### Databases
- **Neon**: Serverless PostgreSQL; branching for dev/test; free tier.
- **Supabase**: Managed PostgreSQL with built-in auth, storage, and real-time.
- **Railway PostgreSQL**: Simple, low-cost managed PostgreSQL.
- **AWS RDS / Aurora**: Production-grade; automatic backups; multi-AZ for high availability.
- **PlanetScale**: MySQL-compatible; schema branching; no foreign key constraints (caveat).

## Secrets Management in CI/CD
- Store all secrets in the CI/CD platform's secret store (GitHub Secrets, GitLab CI Variables, etc.).
- Never print secrets in log output (`::add-mask::` in GitHub Actions).
- Rotate secrets after any suspected exposure.
- Use OIDC / Workload Identity Federation to authenticate to cloud providers without long-lived credentials.

## Release Process
1. Update `CHANGELOG.md` with changes in this release (keep a changelog format).
2. Bump version following Semantic Versioning (`MAJOR.MINOR.PATCH`).
3. Create a Git tag and GitHub Release with release notes.
4. Deploy to production using the tagged image.
5. Verify production health post-deployment.
6. Notify stakeholders.

## Deliverables
- CI/CD pipeline configuration (GitHub Actions or equivalent)
- Dockerfile with multi-stage build and non-root user
- Docker Compose file for local development
- Deployment configuration per environment (staging, production)
- Deployment strategy documented (rolling / blue-green / canary)
- Database migration procedure documented
- Health check endpoints implemented
- Post-deployment smoke test suite
- Rollback runbook
- Secrets management approach documented
- Release process and versioning policy documented
