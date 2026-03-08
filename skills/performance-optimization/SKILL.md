# Skill: Performance Optimization

## Purpose
Make systems fast enough to meet user and business requirements — no faster, no slower. Premature optimization is the root of much evil. But unaddressed performance problems are the root of many production incidents. The discipline is knowing when to optimize, what to optimize, and stopping when the target is met.

## When to Use
Apply this skill when:
- A performance SLA or benchmark target has been defined and is not being met
- Users are complaining about slowness
- Infrastructure costs are unexpectedly high due to inefficiency
- A load test reveals the system cannot handle expected traffic
- A performance regression has been detected in CI

Do NOT apply this skill proactively without data — measure first, then optimize.

## Core Principles
- **Measure before optimizing.** The bottleneck is almost never where you think it is. Profile first.
- **Optimize the bottleneck.** Optimizing anything that is not the bottleneck does not improve overall performance (Amdahl's Law).
- **Set a target and stop.** Optimization has diminishing returns and increasing complexity cost. Define "fast enough" and stop there.
- **One change at a time.** Change one thing, measure the impact, then decide on the next change. Multiple simultaneous changes make it impossible to understand what helped.
- **Correct first, fast second.** A fast program that gives wrong answers is worse than a slow correct one.

## Performance Optimization Workflow

### Step 1 — Define the Target
Without a target, optimization has no end.
- **Latency target**: "p95 response time < 200ms under 500 concurrent users"
- **Throughput target**: "Handle 10,000 requests per second with < 1% error rate"
- **Resource target**: "CPU < 60% at peak load"
- **Cost target**: "Infrastructure cost < $X/month"

### Step 2 — Establish a Baseline
Measure current performance before making any changes:
- Run a realistic load test and record p50/p90/p95/p99 latency, throughput, error rate, CPU, and memory.
- Capture the baseline in a reproducible benchmark that can be re-run to measure improvement.

### Step 3 — Profile to Find the Bottleneck
```
Backend profiling:
  Node.js:  0x, clinic.js, --prof flag, async_hooks
  Python:   cProfile, py-spy, Scalene
  Go:       pprof (cpu, memory, goroutine), trace
  JVM:      async-profiler, JFR, YourKit

Database profiling:
  PostgreSQL: EXPLAIN ANALYZE, pg_stat_statements
  MongoDB:    explain(), db.setProfilingLevel(1)
  Redis:      SLOWLOG, redis-cli --latency-history

Frontend profiling:
  Chrome DevTools Performance tab
  Lighthouse
  WebPageTest
  Core Web Vitals (LCP, INP, CLS)
```

### Step 4 — Optimize the Identified Bottleneck
Apply the appropriate technique from the sections below.

### Step 5 — Measure Again
Re-run the benchmark. Quantify the improvement. If the target is not yet met, return to Step 3.

### Step 6 — Document
Record what was changed, why, and the measured before/after numbers. This is critical — optimizations without documentation get reversed by well-meaning engineers.

## Backend Performance Techniques

### Database Query Optimization (usually the biggest win)

**N+1 Queries** — eliminate first
```javascript
// BAD — 1 + N queries
const users = await User.findAll();
for (const user of users) {
  user.orders = await Order.findAll({ where: { userId: user.id } });
}

// GOOD — 2 queries
const users = await User.findAll({ include: [Order] });
```

**Cursor Pagination over Offset**
```sql
-- BAD — O(n) cost, slow at high page numbers
SELECT * FROM events ORDER BY id DESC LIMIT 20 OFFSET 500000;

-- GOOD — O(log n) using an index
SELECT * FROM events WHERE id < :cursor ORDER BY id DESC LIMIT 20;
```

**Indexes** — most impactful change per line of code
- Add indexes for every WHERE, JOIN, and ORDER BY column
- Use partial indexes for filtered queries
- Use covering indexes to avoid heap fetches
- Build indexes CONCURRENTLY to avoid table lock

**Connection Pooling**
- Without a connection pool, each request opens a new database connection (expensive)
- Use PgBouncer (PostgreSQL) in transaction mode for high-concurrency applications
- Set pool size to: (number of CPU cores * 2) + number of spindles (disks)

**Read Replicas**
- Route read-heavy queries (reports, analytics, search) to a read replica
- Never run expensive analytics queries against the primary database during peak hours

**Query Simplification**
- Avoid SELECT * — select only the columns needed
- Avoid correlated subqueries — use JOINs or CTEs
- Use EXPLAIN ANALYZE on every new query before shipping

### Caching

**Cache Hit Rate** — the primary caching metric; target > 95% for hot paths

**Where to Cache**
```
Request → [CDN Cache] → [Application Cache] → [Redis Cache] → Database
                L1 (edge)      L2 (in-process)    L3 (shared)
```

**What to Cache**
- Expensive computed results (recommendation scores, report aggregations)
- Frequently read, rarely written reference data (categories, config, feature flags)
- Authentication token validation results (cache valid tokens in Redis for 1 minute)
- Rendered HTML fragments (for server-side rendered pages)
- External API responses (with appropriate TTL based on data freshness requirement)

**Cache Stampede Prevention**
When a popular cache entry expires, thousands of requests hit the database simultaneously:
```javascript
// Solution 1: Probabilistic Early Expiration (PER)
// Solution 2: Mutex lock (only one request rebuilds; others wait)
// Solution 3: Background refresh (stale-while-revalidate)

// Example: stale-while-revalidate with Redis
async function getCachedData(key, fetchFn, staleTTL, staleWhileRevalidateTTL) {
  const cached = await redis.get(key);
  if (cached) {
    const { data, expiresAt } = JSON.parse(cached);
    if (Date.now() > expiresAt) {
      // Serve stale data; refresh in background
      fetchFn().then(fresh => redis.setex(key, staleWhileRevalidateTTL, JSON.stringify({
        data: fresh, expiresAt: Date.now() + staleTTL * 1000
      })));
    }
    return data;
  }
  const fresh = await fetchFn();
  await redis.setex(key, staleWhileRevalidateTTL, JSON.stringify({
    data: fresh, expiresAt: Date.now() + staleTTL * 1000
  }));
  return fresh;
}
```

### Asynchronous Processing
Move work that does not need to happen synchronously (within the request) out of the request path:
- Sending emails → job queue
- Generating reports → job queue + polling or webhooks
- Processing uploaded files → job queue
- Sending notifications → job queue
- Updating analytics counters → fire-and-forget or batched writes

### Reducing Payload Sizes
- Enable gzip / Brotli compression on all text responses (JSON, HTML, CSS, JS)
- Use Brotli (better compression ratio than gzip) for static assets
- Paginate large response collections — never return unbounded lists
- Use `fields` parameter for sparse fieldsets in APIs
- Return only changed fields in PATCH responses (or nothing with 204)

### Connection and Resource Management
- Set explicit timeouts on all outbound HTTP calls
- Implement connection pooling for all I/O (database, Redis, HTTP clients)
- Reuse HTTP keep-alive connections; avoid creating a new connection per request
- Use streaming for large data transfers instead of buffering entire response in memory

## Frontend Performance Techniques

### Core Web Vitals Optimization

**LCP (Largest Contentful Paint) — target < 2.5s**
- Preload the LCP resource (`<link rel="preload" as="image" href="hero.webp">`)
- Serve images in modern formats (WebP, AVIF)
- Use a CDN with edge caching for the origin server
- Minimize render-blocking scripts and styles

**INP (Interaction to Next Paint) — target < 200ms**
- Avoid long tasks (> 50ms) on the main thread
- Use `scheduler.yield()` or `requestIdleCallback` for non-critical work
- Debounce or throttle expensive event handlers
- Use web workers for CPU-intensive operations

**CLS (Cumulative Layout Shift) — target < 0.1**
- Always set `width` and `height` attributes on images and videos
- Reserve space for dynamically loaded content
- Avoid inserting content above existing content
- Use `font-display: swap` with web fonts and set explicit font dimensions

### Bundle Size Optimization
```
Tools: vite-bundle-analyzer, webpack-bundle-analyzer, bundlephobia

Techniques:
- Code split at route boundaries (lazy-loaded routes)
- Dynamic imports for large optional features
- Tree-shake unused exports (requires ES modules)
- Replace heavy libraries with lighter alternatives
  (e.g., date-fns instead of moment.js, day.js)
- Externalize rarely used large dependencies (use CDN for production)
- Analyze and remove unused CSS (PurgeCSS with Tailwind)

Target: < 200KB JavaScript (gzipped) for initial load
```

### Image Optimization
- Use WebP or AVIF (30–50% smaller than JPEG/PNG at similar quality)
- Serve appropriately sized images for the display size (`srcset`, `sizes`)
- Lazy load below-the-fold images: `loading="lazy"`
- Use a CDN with automatic image optimization and resizing (Cloudinary, Imgix, Cloudflare Images)
- Set long-lived cache headers for images (they should be content-addressed)

### JavaScript Execution Optimization
- Profile with Chrome Performance tab before optimizing
- Memoize pure functions that are called frequently with the same inputs
- Virtualize long lists (react-virtual, @tanstack/virtual) — do not render > 100 DOM nodes for a list
- Avoid unnecessary re-renders: React.memo, useMemo, useCallback (only where profiling shows benefit)
- Debounce search inputs, resize/scroll handlers (250–500ms)

### Caching and CDN
- Cache static assets with `Cache-Control: public, max-age=31536000, immutable` (use content-addressed URLs)
- Cache HTML with `Cache-Control: no-cache` (revalidate on each request, but serve stale while revalidating)
- Use a CDN (Cloudflare, Fastly, CloudFront) to serve assets from the nearest edge node
- Implement service worker caching for repeat visitors and offline use

## Infrastructure Performance

### Horizontal Scaling
- Design services to be stateless so they can be scaled horizontally without coordination
- Use a load balancer (AWS ALB, Nginx, Traefik) to distribute traffic
- Scale based on metrics (CPU, memory, request rate, queue depth) with auto-scaling rules
- Test the system at 2× expected peak before launch

### CDN Configuration
```
Static assets (JS, CSS, images): Cache-Control: public, max-age=31536000, immutable
API responses (public, cacheable): Cache-Control: public, max-age=60, stale-while-revalidate=300
HTML pages:                         Cache-Control: no-cache
Private/authenticated data:         Cache-Control: private, no-store
```

### Database Scaling
- Vertical: increase RDS instance size (quick, expensive, limited ceiling)
- Horizontal read: add read replicas (good for read-heavy workloads)
- Connection pooling: PgBouncer in front of PostgreSQL (multiplexes thousands of app connections to a few DB connections)
- Partitioning: range/list/hash partitioning for large tables (improves query performance and maintenance)
- Archival: move old data to cold storage (reduces working set size)

## Performance Testing

### Load Testing Tools
- **k6**: JavaScript-based, excellent for complex scenarios, great CI integration
- **Artillery**: YAML/JS-based, good for API testing
- **Locust**: Python-based, good for behavioral simulations

### Load Test Scenarios to Run
1. **Baseline**: expected average load for 10 minutes → verify all SLAs met
2. **Peak**: expected peak load (Black Friday, launch day) for 5 minutes → verify graceful degradation
3. **Soak**: sustained load for 2+ hours → detect memory leaks and connection pool exhaustion
4. **Spike**: sudden 10× traffic spike for 1 minute → verify auto-scaling and load shedding
5. **Stress**: gradually increase load until the system breaks → find the failure point

### Performance Budget in CI
```yaml
# k6 performance test that fails CI if SLAs are breached
thresholds:
  http_req_duration: ['p(95)<200', 'p(99)<500']
  http_req_failed: ['rate<0.01']
  http_reqs: ['rate>100']
```

## Deliverables
- Performance targets defined and documented (latency, throughput, cost)
- Baseline performance measurement established
- Profiling completed; bottleneck identified before any optimization
- Specific optimizations implemented (documented with before/after numbers)
- Performance tests in CI with defined SLA thresholds
- Bundle size analysis for frontend; budget enforced in CI
- Caching strategy implemented at appropriate layers
- Database query plans reviewed; missing indexes added
- CDN configured for static assets
- Load test results documented
