# Skill: Incident Response and Debugging

## Purpose
Diagnose and resolve production incidents quickly, systematically, and safely. The quality of an incident response determines the impact on users and the team's ability to prevent recurrence. Chaotic, undocumented incident response is itself a form of technical debt.

## When to Use
Invoke this skill when:
- A production alert fires
- Users report unexpected behavior
- A deployment causes degradation
- Performance or error metrics deviate from baseline
- A security event is suspected
- A bug needs to be diagnosed in a complex or distributed system

## Incident Response Framework

### Severity Classification

| Severity | Definition | Example | Response SLA |
|----------|-----------|---------|-------------|
| **P1 — Critical** | Complete service outage or data loss/corruption | Login is broken for all users; payments failing | 15 min, 24/7 |
| **P2 — High** | Significant degradation; majority of users affected | 50% of API calls failing; 5× latency spike | 30 min, business hours |
| **P3 — Medium** | Partial degradation; subset of users/features affected | One API endpoint returning errors; slow search | 4 hours |
| **P4 — Low** | Minor issue; no immediate user impact | Non-critical background job failing; cosmetic UI bug | Next business day |

### Incident Response Phases

#### Phase 1 — Detect (0–5 minutes)
- Confirm the incident is real (not a monitoring false positive).
- Classify the severity using the table above.
- Declare the incident if P1 or P2: create an incident in PagerDuty / Opsgenie, open an incident Slack channel, notify stakeholders.
- Assign an **Incident Commander** (IC): the one person who coordinates response; not necessarily the one fixing it.
- Assign a **Communications Lead**: handles stakeholder updates so the IC can focus on resolution.

#### Phase 2 — Contain (0–15 minutes for P1)
Do not try to fix the root cause yet. First, limit the damage.

Common containment actions:
- Roll back the recent deployment (most incidents are caused by recent changes)
- Enable maintenance mode or return a cached/degraded response
- Disable a feature flag that enabled the broken feature
- Block traffic from an abusive IP or source
- Increase rate limits to shed load
- Redirect traffic to a healthy region or instance

**Rule**: A degraded but working system is always better than a broken system being debugged.

#### Phase 3 — Diagnose (parallel with containment)
Work systematically from the outside in:

```
1. What changed recently?
   - Deployments in the last 2 hours
   - Configuration changes
   - Traffic pattern changes
   - Third-party dependency changes

2. What is the impact?
   - Which users are affected (all, subset, specific geography)?
   - Which endpoints/features are affected?
   - When did it start?
   - Is it getting worse, stable, or recovering?

3. What do the signals show?
   - Error rates: which endpoints, which error codes?
   - Latency: which operations are slow?
   - Logs: what error messages appear at the time of onset?
   - Traces: where is the request failing/slowing?
   - Infrastructure: CPU, memory, disk, network — anything anomalous?
   - Database: slow queries? Connection pool exhausted? Replication lag?
   - External dependencies: are third-party APIs returning errors?

4. Hypothesis and test:
   - Form a specific hypothesis ("I think the database connection pool is exhausted")
   - Find one observation that would prove or disprove it (check pool metrics)
   - Act on the finding
```

#### Phase 4 — Resolve
- Implement the fix or workaround.
- Verify the fix by monitoring metrics returning to baseline.
- Confirm with the Communications Lead that user impact has ended.
- Declare the incident resolved in the incident management system.
- Notify stakeholders of resolution.

#### Phase 5 — Post-Incident Review (within 48 hours)
Write a blameless post-mortem. The goal is to improve the system, not to assign blame.

**Post-Mortem Template:**
```markdown
# Incident Post-Mortem: [Title]

**Date**: YYYY-MM-DD
**Severity**: P1 / P2 / P3
**Duration**: X hours Y minutes
**Impact**: N% of users affected; X errors; $Y revenue impact (if known)
**On-Call**: [Names of responders]

## Timeline
- HH:MM — Alert fired / incident detected
- HH:MM — Incident declared; IC assigned
- HH:MM — Root cause identified as [X]
- HH:MM — Mitigation applied
- HH:MM — System restored to normal
- HH:MM — Incident resolved

## Root Cause
[One paragraph describing the underlying cause — not the symptoms]

## Contributing Factors
- [Factor 1]: [How it contributed]
- [Factor 2]: [How it contributed]

## What Went Well
- [Monitoring caught the issue within X minutes]
- [Team executed rollback procedure correctly]

## What Could Be Improved
- [Describe gaps in detection, response, or prevention]

## Action Items

| Action | Owner | Priority | Due Date |
|--------|-------|----------|----------|
| Add alert for X condition | @engineer | P1 | 2025-02-01 |
| Add test for Y scenario | @engineer | P2 | 2025-02-15 |
| Update runbook for Z | @engineer | P3 | 2025-03-01 |
```

## Debugging Techniques

### Systematic Debugging Protocol

Never debug by making random changes. Follow a scientific method:

1. **Reproduce** the bug in a controlled environment.
2. **Isolate** the minimum reproducible case (simplest input that triggers the bug).
3. **Hypothesize** the root cause based on evidence.
4. **Test** the hypothesis with a targeted observation or experiment.
5. **Fix** only after the root cause is confirmed.
6. **Verify** that the fix resolves the issue without introducing new problems.
7. **Prevent** recurrence with a regression test.

### Log Analysis

```bash
# Filter logs by error level and time range (structured JSON logs)
cat app.log | jq 'select(.level == "error" and .timestamp > "2025-01-15T14:00:00Z")'

# Find the most common error messages
cat app.log | jq -r 'select(.level == "error") | .message' | sort | uniq -c | sort -rn

# Trace all events for a specific request
cat app.log | jq 'select(.requestId == "req_abc123")'

# Find all events for a specific user across multiple services
grep '"userId":"usr_abc123"' *.log | jq '.'

# Count error rate over time (5-minute buckets)
cat app.log | jq -r 'select(.level == "error") | .timestamp[:16]' | \
  sed 's/T/ /' | cut -d: -f1,2 | sort | uniq -c
```

### Database Diagnostics

```sql
-- Find currently running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle';

-- Kill a specific query
SELECT pg_cancel_backend(pid);         -- Graceful cancel
SELECT pg_terminate_backend(pid);      -- Force terminate

-- Find tables with highest sequential scan rate (missing indexes)
SELECT relname, seq_scan, idx_scan, seq_scan / (seq_scan + idx_scan + 1.0) AS seq_ratio
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 100
ORDER BY seq_ratio DESC;

-- Find the most time-consuming queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Check connection pool usage
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;

-- Check table bloat (dead tuples requiring VACUUM)
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup / (n_live_tup + n_dead_tup + 1.0) * 100, 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;
```

### Memory Leak Detection

**Node.js:**
```javascript
// Enable heap snapshots
process.on('SIGUSR2', () => {
  const v8 = require('v8');
  const heapDump = v8.writeHeapSnapshot();
  console.log('Heap snapshot written to', heapDump);
});

// Or use clinic.js: clinic heapprofile -- node server.js
// Compare heap snapshots in Chrome DevTools Memory tab
```

**Python:**
```python
# Use tracemalloc to find memory leaks
import tracemalloc

tracemalloc.start()
# ... run the code ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:10]:
    print(stat)
```

### Distributed System Debugging

In a distributed system, the bug may not be in the service that is exhibiting the symptom.

```
1. Use distributed tracing to find the slow/failing span
   → Jaeger / Tempo / Datadog APM → find the trace → find the failing span

2. Check the service dependency graph
   → Which upstream service made the call that failed?
   → Which downstream service did the failing service call?

3. Compare: is this one request or all requests to this service?
   → If one: data-specific bug (check the payload, the database row)
   → If all: infrastructure or configuration problem

4. Check: did anything change in that service recently?
   → Recent deployment? Configuration change? Dependency version bump?

5. Network-level debugging
   tcpdump -i eth0 port 5432 -w /tmp/postgres.pcap   # Capture DB traffic
   ss -tnp                                             # Show TCP connections
   curl -v https://api.example.com/health              # Test HTTP reachability
```

### Performance Debugging in Production

```bash
# CPU profiling of a running Node.js process (no restart required)
kill -USR1 <pid>    # Enable inspector
# Attach Chrome DevTools → chrome://inspect

# Continuous profiling with 0x
npx 0x --output-dir /tmp/profile node server.js

# Python: py-spy for sampling profiler (no code changes)
py-spy top --pid <pid>
py-spy record -o /tmp/profile.svg --pid <pid>

# Go: pprof endpoint
curl http://localhost:6060/debug/pprof/goroutine?debug=2
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof

# Linux process diagnostics
strace -p <pid> -e trace=open,read,write   # System call trace
lsof -p <pid>                               # Open file descriptors
cat /proc/<pid>/status | grep VmRSS         # Resident memory
```

## Runbook Templates

### Service Is Down (P1 Runbook)
```
1. CHECK: Is the service process running?
   kubectl get pods -l app=service-name
   If not running → kubectl describe pod <pod-name> to see why

2. CHECK: Are pods healthy?
   kubectl get pods -l app=service-name -o wide
   Look for: CrashLoopBackOff, OOMKilled, Pending

3. CHECK: Recent deployments
   kubectl rollout history deployment/service-name
   If recent: kubectl rollout undo deployment/service-name

4. CHECK: Resource exhaustion
   kubectl top pods -l app=service-name
   kubectl describe node <node-name> | grep -A5 Allocated

5. CHECK: Upstream dependencies
   kubectl exec -it <pod> -- wget -qO- http://postgres-service:5432  (db reachable?)
   kubectl exec -it <pod> -- redis-cli -h redis-service ping

6. CHECK: Error logs
   kubectl logs -l app=service-name --since=10m --tail=100

ESCALATE TO: [Senior engineer / DBA / Platform team] if not resolved in 15 minutes
```

### High Error Rate (P2 Runbook)
```
1. IDENTIFY: Which endpoints?
   → Check Grafana: HTTP errors by endpoint
   → Note the error codes (5xx vs 4xx)

2. IDENTIFY: When did it start?
   → Correlate with recent deployments or config changes

3. CHECK: Error messages in logs
   → Find the most common error message since the onset time

4. CHECK: Downstream dependencies
   → Database: are queries slow? Is the connection pool exhausted?
   → External APIs: are they returning errors?

5. MITIGATE: If caused by recent deployment
   → kubectl rollout undo deployment/service-name

6. MITIGATE: If database-related
   → Check pg_stat_activity for blocking queries
   → Kill long-running queries if necessary

ESCALATE TO: [Database team] if database-related; [Platform team] if infrastructure
```

## Blameless Culture

Incidents are failures of systems, not failures of people. A blameless culture enables:
- Honest post-mortems that identify the real root cause
- Engineers who feel safe escalating problems instead of hiding them
- Continuous improvement instead of punishment cycles

Rules for blameless incident response:
- Post-mortems do not name individuals as the cause
- "Human error" is never a root cause — it is a symptom of a system that made the error easy to make
- The goal is to change the system, not the person
- On-call engineers acted with the best information available at the time

## Deliverables
- Incident severity classification defined and documented
- On-call rotation established with clear escalation paths
- Runbooks for top 5 most likely incident types
- Post-mortem template adopted by the team
- Incident tracking tool configured (PagerDuty, Opsgenie, or equivalent)
- Incident communication channel established (dedicated Slack channel)
- Post-mortem process: written within 48 hours, action items tracked to completion
- Alerting rules covering the four golden signals
- Common debugging commands documented in the runbook
