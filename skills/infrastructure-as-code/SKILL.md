# Skill: Infrastructure as Code

## Purpose
Manage all infrastructure — servers, networks, databases, queues, CDNs — as versioned, tested, and repeatable code. Infrastructure created through web consoles cannot be reproduced, audited, or reviewed. Infrastructure as Code (IaC) is the foundation of reliable, scalable, and secure operations.

## When to Use
Apply this skill when setting up any cloud or on-premises infrastructure, when automating deployments, when establishing CI/CD pipelines, and when ensuring environment parity between development, staging, and production.

## Core Principles
- **Everything in code**: if it was clicked in a web console, it does not exist as far as version control is concerned.
- **Idempotency**: running the same IaC configuration twice must produce the same result.
- **Environment parity**: dev, staging, and production should differ only in size and configuration, not in architecture.
- **Least privilege**: every service identity should have the minimum cloud permissions required.
- **Immutable infrastructure**: prefer replacing infrastructure over mutating it in place.
- **GitOps**: the repository is the single source of truth; the infrastructure must always converge to what is described in the repository.

## Infrastructure as Code Tools

### Terraform (recommended for cloud-agnostic infrastructure)
- Declarative: describe the desired state; Terraform figures out how to get there.
- Multi-provider: AWS, GCP, Azure, Kubernetes, Cloudflare, GitHub, and many more.
- State management: tracks current state in a state file (store remotely in S3/GCS, not locally).
- Plan before apply: `terraform plan` shows exactly what will change before committing.

### Pulumi (IaC with real programming languages)
- Use TypeScript, Python, Go, or C# to define infrastructure.
- Best when infrastructure logic is complex enough to benefit from real language features.

### AWS CDK / Google Cloud Deployment Manager / Azure Bicep
- Cloud-native; tight integration with the respective cloud provider.
- Use when deep cloud-native integration is preferred over portability.

### Ansible (configuration management)
- Procedural (unlike Terraform's declarative model).
- Best for configuring OS-level concerns, installing software on servers, and post-provisioning configuration.

## Containerization (Docker)

### Dockerfile Best Practices
```dockerfile
# Pin the base image to a specific digest for reproducibility
FROM node:20.11-alpine3.19 AS builder

# Set working directory
WORKDIR /app

# Copy dependency manifests first to maximize layer cache reuse
COPY package*.json ./
RUN npm ci --only=production --frozen-lockfile

# Copy source code
COPY . .

# Build production artifact
RUN npm run build && npm prune --production

# ---- Minimal production stage ----
FROM node:20.11-alpine3.19 AS production

# Never run containers as root
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001 -G nodejs
WORKDIR /app

# Copy only what is needed from the builder stage
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs

# Add built-in health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

Key rules:
- Use multi-stage builds to keep production images minimal
- Pin base images to specific version tags or digests (never `latest`)
- Run as a non-root user
- Copy only the minimum files needed
- Sort multi-line RUN commands alphabetically for diff readability
- Combine related RUN commands to reduce layers (`RUN apt-get update && apt-get install ...`)
- Add `.dockerignore` to exclude: `node_modules`, `.git`, `*.test.js`, `.env`

### Docker Compose (Local Development)
```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src   # Hot reload in development

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  postgres_data:
```

## Kubernetes (for production container orchestration)

### Core Concepts to Apply
- **Deployments**: manage stateless application pods with rolling updates and rollback
- **StatefulSets**: for stateful applications (databases, caches)
- **Services**: stable DNS and load balancing for pod discovery
- **Ingress**: HTTP routing and TLS termination
- **ConfigMaps**: non-sensitive configuration
- **Secrets**: sensitive configuration (prefer external secret operators)
- **HorizontalPodAutoscaler (HPA)**: auto-scale based on CPU/memory/custom metrics
- **PodDisruptionBudget (PDB)**: guarantee minimum availability during updates
- **NetworkPolicy**: restrict pod-to-pod communication (deny by default)
- **ResourceQuota and LimitRange**: prevent resource exhaustion

### Production-Ready Deployment Template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  labels:
    app: api-service
    version: "1.4.2"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # Zero-downtime deployment
  template:
    metadata:
      labels:
        app: api-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
      containers:
        - name: api-service
          image: registry.example.com/api-service:1.4.2
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
      topologySpreadConstraints:   # Spread pods across availability zones
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
```

### Kubernetes Security Checklist
- [ ] Pods run as non-root user
- [ ] `readOnlyRootFilesystem: true` applied
- [ ] `allowPrivilegeEscalation: false` applied
- [ ] All Linux capabilities dropped (`drop: ["ALL"]`)
- [ ] NetworkPolicies defined (deny all ingress/egress by default)
- [ ] Resource limits set on all containers
- [ ] Secrets sourced from External Secrets Operator, not baked into ConfigMaps
- [ ] Container images scanned for CVEs
- [ ] Pod Security Admission enforced

## Terraform Project Structure

```
infrastructure/
├── modules/
│   ├── networking/      # VPC, subnets, security groups
│   ├── database/        # RDS / Cloud SQL configuration
│   ├── kubernetes/      # EKS / GKE cluster configuration
│   ├── cache/           # ElastiCache / Memorystore
│   └── cdn/             # CloudFront / Cloudflare
├── environments/
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── backend.tf           # Remote state configuration
└── providers.tf
```

### Terraform Best Practices
```hcl
# Always pin provider versions
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
  required_version = ">= 1.6"

  # Always use remote state storage
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # State locking
  }
}

# Tag all resources for cost allocation and auditability
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = var.project_name
    Owner       = var.team
  }
}
```

### Terraform Workflow
```
# Developer workflow
terraform fmt -recursive        # Format code
terraform validate              # Validate configuration syntax
terraform plan -out=tfplan      # Preview changes
# Review plan carefully — every resource change is significant
terraform apply tfplan          # Apply changes

# CI/CD workflow
terraform fmt -check -recursive  # Fail if not formatted
terraform validate
terraform plan -out=tfplan        # Save plan
# Manual approval step for production
terraform apply tfplan
```

## CI/CD Pipeline Configuration (GitHub Actions)

### Complete CI/CD Pipeline
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v4

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - uses: github/codeql-action/analyze@v3

  build-image:
    needs: [lint-and-type-check, test, security-scan]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build-push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
      - uses: docker/build-push-action@v5
        id: build-push
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Scan the built image for vulnerabilities
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build-push.outputs.digest }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  deploy-staging:
    needs: [build-image]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          # Update Kubernetes deployment image tag
          kubectl set image deployment/api-service \
            api-service=${{ needs.build-image.outputs.image-tag }}
      - name: Verify staging deployment
        run: |
          kubectl rollout status deployment/api-service --timeout=5m
          # Run smoke tests
          npm run test:smoke -- --env staging

  deploy-production:
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.example.com
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/api-service \
            api-service=${{ needs.build-image.outputs.image-tag }}
      - name: Monitor deployment
        run: |
          kubectl rollout status deployment/api-service --timeout=10m
          npm run test:smoke -- --env production
```

## Environment Variable Management

### Local Development
```bash
# .env.example (committed to repository — template with no secrets)
DATABASE_URL=postgres://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
JWT_SECRET=          # Generate with: openssl rand -hex 32
PORT=3000

# .env (not committed — actual local values)
# Developers copy .env.example → .env and fill in values
```

### Staging and Production
- Use a secrets manager (Vault, AWS Secrets Manager, GitHub Secrets)
- Inject secrets as environment variables at runtime — not baked into container images
- Rotate secrets on a schedule and immediately after any suspected exposure
- Use OIDC/Workload Identity Federation to authenticate to cloud services without long-lived credentials

## Infrastructure Security

- Apply the principle of least privilege to all IAM roles and service accounts
- Use separate cloud accounts or projects per environment (production must be isolated)
- Enable CloudTrail / Audit Logs for all infrastructure changes
- Apply resource-based policies to limit blast radius of a compromised service
- Use private subnets for databases, caches, and internal services (no public internet access)
- Enable VPC Flow Logs for network forensics
- Configure WAF (Web Application Firewall) in front of public-facing services
- Enable automatic vulnerability patching for managed services (RDS, Lambda runtimes)
- Run infrastructure configuration compliance checks (Checkov, tfsec, KICS) in CI

## Deliverables
- All infrastructure defined in Terraform or equivalent IaC tool
- Remote state backend configured with state locking
- Environments separated (staging isolated from production)
- Dockerfile with multi-stage build, non-root user, and health check
- Docker Compose for local development
- Kubernetes manifests for production (or equivalent managed service config)
- CI/CD pipeline in GitHub Actions (or equivalent)
- Image vulnerability scanning in CI
- Environment variable management strategy documented
- IAM roles following least privilege principle
- Infrastructure changes tracked in version control with plan review before apply
