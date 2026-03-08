# Skill: Observability and Monitoring

## Purpose
Make systems understandable from the outside. Observability is not something you add after problems occur — it is the practice of instrumenting systems so that any question about their internal state can be answered using the data they emit. An unobservable system in production is a liability.

## When to Use
Apply this skill when building any production service, when setting up a new environment, and when investigating reliability or performance problems. Observability tooling should be in place before the first production deployment.

## The Three Pillars of Observability

### 1. Logs — What happened
Logs are the timestamped record of discrete events. They answer: "what did this system do?"

**Structured Logging (required)**
- Always log in structured JSON format — never plain text in production.
- JSON logs can be queried, parsed, and aggregated by log management systems.
- Every log entry must include: `timestamp`, `level`, `message`, `service`, `environment`.
- Add contextual fields: `userId`, `requestId`, `traceId`, `spanId` for correlated search.

```json
{
  "timestamp": "2025-01-15T14:23:11.342Z",
  "level": "info",
  "message": "User login successful",
  "service": "auth-service",
  "environment": "production",
  "userId": "usr_abc123",
  "requestId": "req_xyz789",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "durationMs": 42
}
```

**Log Levels — use them consistently**
- `DEBUG`: detailed diagnostic information; disabled in production by default
- `INFO`: significant application events (user login, order created, job started/completed)
- `WARN`: unusual but handled situations (retried operation, deprecated API called, configuration fallback used)
- `ERROR`: failures that require investigation but the service continues running
- `FATAL`: critical failures causing the service to stop

**What to log**
- Service startup and shutdown (including configuration loaded — excluding secret values)
- Every incoming HTTP request: method, path, status code, duration, user ID
- Authentication events: login success/failure, token refresh, logout
- State-changing operations: create, update, delete of significant entities
- External API calls: service name, endpoint, status, duration
- Background job lifecycle: started, completed, failed
- Circuit breaker state changes: open, half-open, closed

**What NOT to log**
- Passwords, tokens, API keys, credit card numbers (PII/secrets)
- Full request/response bodies (may contain sensitive data)
- At DEBUG level in production (performance impact)

**Log Management Platforms**
- **Grafana Loki**: Cost-effective, integrates with Grafana dashboards.
- **Datadog Logs**: Enterprise-grade, excellent search and alerting.
- **Elastic Stack (ELK)**: Self-hosted option; powerful but operationally complex.
- **AWS CloudWatch Logs**: Native if running on AWS.
- **Logtail**: Simple and affordable SaaS.

### 2. Metrics — How the system behaves over time
Metrics are numeric measurements aggregated over time. They answer: "is the system healthy right now and how has it trended?"

**The Four Golden Signals (Google SRE)**
1. **Latency**: How long it takes to serve a request (track p50, p90, p95, p99 — not just averages).
2. **Traffic**: How many requests per second the system is handling.
3. **Errors**: The rate of requests that fail (5xx errors, timeouts, validation failures).
4. **Saturation**: How "full" the service is — CPU, memory, disk, connection pool, queue depth.

**Metric Types**
- **Counter**: Monotonically increasing value (total requests, total errors). Suitable for rate calculations.
- **Gauge**: Point-in-time value that can go up or down (active connections, queue depth, memory usage).
- **Histogram**: Distribution of values (request latency, response sizes). Used to calculate percentiles.
- **Summary**: Pre-computed percentiles (less flexible than histograms for aggregation).

**Essential Metrics to Instrument**

HTTP API:
```
http_requests_total{method, path, status_code}          # Counter
http_request_duration_seconds{method, path, status_code} # Histogram
http_requests_in_flight                                   # Gauge
```

Database:
```
db_query_duration_seconds{operation, table}  # Histogram
db_connection_pool_active                    # Gauge
db_connection_pool_idle                      # Gauge
db_errors_total{operation}                   # Counter
```

Background Jobs:
```
job_queue_depth{queue_name}      # Gauge
job_processing_duration_seconds  # Histogram
job_failures_total{job_type}     # Counter
job_success_total{job_type}      # Counter
```

Business Metrics (critical — often forgotten):
```
orders_created_total             # Counter
revenue_total_usd                # Counter
active_users_current             # Gauge
feature_flag_evaluations_total   # Counter
```

**Metrics Platforms**
- **Prometheus + Grafana**: Industry standard, open-source, excellent for Kubernetes environments.
- **Datadog**: Full-stack observability SaaS; excellent alerting and anomaly detection.
- **New Relic**: Similar to Datadog; strong APM capabilities.
- **AWS CloudWatch**: Native if running on AWS; cost-effective for AWS-heavy architectures.

### 3. Distributed Traces — How a request traveled through the system
Traces connect the dots across service calls, showing the complete path of a request from entry to exit, including timing for each hop. They answer: "why was this specific request slow?"

**OpenTelemetry (use this — it is the industry standard)**
- Instrument all services with the OpenTelemetry SDK.
- Propagate trace context via HTTP headers (`traceparent`, `tracestate` — W3C Trace Context standard).
- Every span should record: service name, operation name, start time, duration, status, key attributes.
- Link logs to traces using `traceId` and `spanId` in every log entry.

**What to trace**
- Every inbound HTTP request (auto-instrumented)
- Every outbound HTTP call to another service or external API
- Every database query
- Every cache operation (Redis GET/SET)
- Every message queue publish and consume
- Every significant business operation (with business context as span attributes)

**Tracing Platforms**
- **Jaeger**: Open-source, excellent for self-hosted environments.
- **Tempo (Grafana)**: Pairs with Loki and Prometheus for the full Grafana observability stack.
- **Datadog APM**: Enterprise-grade distributed tracing with automatic service map generation.
- **AWS X-Ray**: Native if running on AWS.

## SLIs, SLOs, and SLAs

**Service Level Indicators (SLIs)** — what you measure
Examples:
- Availability: `(successful_requests / total_requests) * 100`
- Latency: `% of requests completing in < 200ms`
- Error rate: `(error_responses / total_responses) * 100`

**Service Level Objectives (SLOs)** — your reliability targets
Examples:
- 99.9% of requests respond with 2xx or 4xx (not 5xx) within a rolling 30-day window
- 95% of requests complete in < 200ms measured at p95
- Zero data loss for user-submitted data

**Error Budget** = `100% - SLO target`
- 99.9% SLO → 0.1% error budget → ~43.8 minutes of downtime allowed per 30 days
- Track error budget burn rate; alert when burning too fast

**Service Level Agreements (SLAs)** — commitments to external parties
- SLAs must be weaker than SLOs (build in a safety margin)
- SLA breach has business/contractual consequences

## Alerting

### Alert Design Principles
- **Alert on symptoms, not causes**: alert on "high error rate" not "CPU is high" (unless CPU directly causes user impact).
- **Every alert must be actionable**: if the response is "nothing to do", the alert should not fire.
- **Alert fatigue kills incident response**: too many low-signal alerts trains engineers to ignore all alerts.
- **Severity levels**: define clear severity levels with corresponding response times.

### Alert Severity Levels

| Severity | Definition | Response Time | Notification Channel |
|----------|-----------|---------------|---------------------|
| P1 Critical | Users cannot use the product | 15 minutes, 24/7 | PagerDuty, phone call |
| P2 High | Significant degradation, most users affected | 30 minutes, business hours | PagerDuty, Slack |
| P3 Medium | Partial degradation, subset of users affected | 4 hours | Slack only |
| P4 Low | Minor issue, no immediate user impact | Next business day | Email / ticket |

### Essential Alerts to Configure
```
ERROR RATE       > 1% over 5 minutes                          → P2
ERROR RATE       > 5% over 1 minute                           → P1
LATENCY (p99)    > 2× baseline over 5 minutes                 → P2
LATENCY (p99)    > 5× baseline over 1 minute                  → P1
AVAILABILITY     < 99.5% over 5 minutes (rolling)             → P1
DB CONNECTIONS   > 80% of pool size                           → P2
QUEUE DEPTH      > 10,000 messages and growing                → P2
DISK USAGE       > 85%                                        → P3
DISK USAGE       > 95%                                        → P1
MEMORY USAGE     > 90% for > 5 minutes                        → P2
CERTIFICATE EXPIRY < 30 days                                  → P3
CERTIFICATE EXPIRY < 7 days                                   → P1
DEPLOYMENT FAILED                                             → P2
SLO ERROR BUDGET < 10% remaining in the 30-day window         → P2
```

## Dashboards

Design dashboards to answer specific operational questions, not to display all available data.

### Service Overview Dashboard
Every service should have a standard overview dashboard with:
1. **Request rate** (req/s, last 1 hour)
2. **Error rate** (% errors, last 1 hour)
3. **Latency** (p50/p95/p99, last 1 hour)
4. **Saturation** (CPU, memory, active connections)
5. **Active alerts** panel
6. **Recent deployments** marker on all time series charts

### Business Metrics Dashboard
Separate from technical metrics:
- Key business KPIs (daily active users, orders per hour, revenue rate)
- Funnel conversion rates (signup → activation → purchase)
- Feature adoption rates

## Health Check Standards

Every service must expose:

```
GET /health
→ 200 OK if the process is alive (liveness probe)
{
  "status": "ok",
  "service": "auth-service",
  "version": "1.4.2",
  "uptime": 3600
}

GET /health/ready
→ 200 OK only when ready to serve traffic (readiness probe)
→ 503 Service Unavailable when dependencies are unhealthy
{
  "status": "ok",
  "checks": {
    "database": { "status": "ok", "latencyMs": 2 },
    "redis": { "status": "ok", "latencyMs": 1 },
    "externalApi": { "status": "degraded", "latencyMs": 450, "message": "elevated latency" }
  }
}
```

## On-Call Runbook Requirements

Before going to production, write runbooks for the following scenarios:
1. Service is returning high error rate
2. Service is responding slowly (latency degraded)
3. Service is completely unreachable
4. Database is slow or unreachable
5. Job queue is backed up
6. Memory/CPU is abnormally high
7. Disk is filling up

Each runbook entry includes: how to detect, how to diagnose, how to mitigate, how to resolve, and who to escalate to.

## Deliverables
- Structured JSON logging configured for all services
- Log management platform configured (Loki, Datadog, or equivalent)
- Prometheus metrics endpoint (`/metrics`) on all services
- The four golden signals instrumented
- Business-critical custom metrics defined and instrumented
- Grafana / Datadog dashboards: service overview + business metrics
- Distributed tracing via OpenTelemetry configured
- Trace IDs propagated through all service calls and correlated in logs
- SLIs and SLOs defined for each critical service
- Alerts configured for all essential conditions
- On-call runbook written for top 7 incident scenarios
- Health check endpoints (`/health` and `/health/ready`) implemented
