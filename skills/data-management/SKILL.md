# Skill: Data Management

## Purpose
Design and manage data persistence in a way that is reliable, performant, and safe. Data is the most valuable and most irreplaceable asset in any system. Data management decisions are among the hardest to reverse — invest in getting them right.

## When to Use
Apply this skill when designing a new data model, selecting a database, writing database queries, planning migrations, or addressing data reliability and performance concerns.

## Database Selection

### Relational Databases (SQL)
**Use when**: data has relationships, referential integrity matters, ACID transactions are required, query patterns are not fully known upfront.
- **PostgreSQL**: Default choice. Most feature-complete open-source RDBMS. Supports JSON, arrays, full-text search, time-series (TimescaleDB extension), geospatial (PostGIS).
- **MySQL / MariaDB**: Widely deployed; slightly simpler operational model; strong replication ecosystem.
- **SQLite**: For local/embedded applications, development, testing. Not for multi-process production use.

### Document Databases (NoSQL)
**Use when**: data is document-shaped with variable structure, schema flexibility is genuinely needed, and complex cross-document transactions are not required.
- **MongoDB**: Flexible schema; rich aggregation pipeline; horizontal scaling with sharding.
- Caution: MongoDB's flexible schema is a feature and a risk. Without discipline, it leads to inconsistent data that is difficult to query.

### Key-Value Stores
**Use when**: very fast, simple key-based access; caching; session storage; distributed locks; counters.
- **Redis**: In-memory; sub-millisecond latency; supports rich data structures (sorted sets, streams, pub/sub). Use for caching, rate limiting, session storage, job queues.
- **DynamoDB**: Managed, massively scalable key-value + document store. Best for high-throughput, single-digit-millisecond latency at internet scale.

### Time-Series Databases
**Use when**: data is timestamped measurements (metrics, IoT, financial data) that need efficient range queries and aggregations over time.
- **TimescaleDB**: PostgreSQL extension; familiar SQL; excellent performance for time-series.
- **InfluxDB**: Purpose-built; excellent for high-ingest-rate metrics.
- **ClickHouse**: Column-oriented; exceptional for analytics queries over large time-series datasets.

### Graph Databases
**Use when**: the relationships between entities are as important as the entities themselves (social networks, recommendation engines, fraud detection, knowledge graphs).
- **Neo4j**: Most mature graph database; Cypher query language.
- **Amazon Neptune**: Managed; supports Gremlin and SPARQL.

## Data Modeling

### Relational Design Principles
1. **Normalization first**: start with 3NF to eliminate data duplication and update anomalies.
2. **Denormalize deliberately**: denormalize only when you have measured a performance problem and denormalization is the solution. Document why.
3. **Surrogate keys**: use auto-generated UUIDs or sequential IDs as primary keys. Never use business data (email, username, phone number) as a primary key — business data changes.
4. **UUID vs. sequential IDs**: UUIDs are better for distributed systems and avoid exposing record counts; sequential integers are more compact and sort naturally.
5. **Timestamps on every table**: add `created_at` and `updated_at` to every table. Use `deleted_at` for soft deletes rather than hard deletes when audit trails are needed.
6. **Avoid NULLable columns** where possible; use separate tables for optional relationships.
7. **Foreign key constraints**: always define FK constraints at the database level (not just in application code). The database is the last line of defense.

### Schema Design Anti-Patterns to Avoid
- **God table**: one table with 100+ columns containing data for many different entity types.
- **EAV (Entity-Attribute-Value)**: using a key/value table instead of typed columns. Nearly always the wrong choice.
- **Polymorphic associations**: foreign keys that point to rows in different tables based on a `type` column. Hard to maintain referential integrity.
- **Storing arrays in comma-separated strings**: use proper array columns or a junction table.

## Indexing Strategy

Indexes are the single most impactful performance lever for relational databases.

### Rules
1. Every column used in a `WHERE` clause, `JOIN` condition, or `ORDER BY` clause is a candidate for indexing.
2. Every foreign key column should have an index (PostgreSQL does not create these automatically).
3. Composite indexes: column order matters — put the most selective column first.
4. An index that is never used is pure overhead (increases write time, consumes disk). Identify and remove unused indexes.
5. Partial indexes: index only the rows that satisfy a condition (e.g., `WHERE status = 'pending'`). Much smaller and faster for filtered queries.
6. Covering indexes: an index that contains all columns needed by a query avoids fetching the row entirely (index-only scan).

### Essential Indexes to Always Create
```sql
-- Primary key (auto-created)
-- Foreign key indexes (create explicitly)
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Lookup by unique business identifier
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Time-range queries
CREATE INDEX idx_events_created_at ON events(created_at DESC);

-- Status-filtered queries (partial index example)
CREATE INDEX idx_jobs_pending ON jobs(created_at) WHERE status = 'pending';

-- Composite index for common query pattern
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### Diagnosing Missing Indexes
```sql
-- Find slow queries in PostgreSQL
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Identify sequential scans on large tables
SELECT relname, seq_scan, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Analyze query plan
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

## Query Optimization

### N+1 Query Problem (most common performance bug)
```javascript
// BAD — issues N+1 database queries
const users = await User.findAll();
for (const user of users) {
  user.orders = await Order.findAll({ where: { userId: user.id } }); // N queries!
}

// GOOD — issues 2 queries (or 1 with a JOIN)
const users = await User.findAll({ include: [{ model: Order }] });
// OR
const users = await db.query(`
  SELECT u.*, o.id as order_id, o.total
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id
`);
```

### Pagination
```sql
-- BAD: Offset pagination is O(n) — slow on large tables
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;

-- GOOD: Cursor-based pagination is O(log n)
SELECT * FROM orders
WHERE created_at < :cursor
ORDER BY created_at DESC
LIMIT 20;
```

### Query Best Practices
- `SELECT` only the columns you need — never `SELECT *` in production queries.
- Avoid `DISTINCT` as a crutch for fixing duplicates — fix the root cause (usually a missing JOIN condition).
- `COUNT(*)` is faster than `COUNT(column)` when checking existence.
- Use `EXISTS` instead of `COUNT(*)` when checking for the existence of a row.
- Run `EXPLAIN ANALYZE` on every query added to the codebase.

## Caching Strategy

Caching is a performance optimization with correctness trade-offs. Plan cache invalidation before implementing caching.

### Cache Layers
1. **Application cache (in-process)**: LRU cache in memory. For immutable/rarely changing reference data. Fastest but not shared across instances.
2. **Distributed cache (Redis)**: shared across all instances. For session data, computed results, rate limiting counters.
3. **CDN cache**: for publicly cacheable HTTP responses (static assets, public API responses).
4. **Database query cache**: materialized views, indexed computed columns. For expensive recurring queries.

### Cache Invalidation Strategies
- **TTL-based**: cache expires after a set time. Simple; may serve stale data.
- **Write-through**: update cache on every write. Cache is always consistent; higher write overhead.
- **Cache-aside (lazy loading)**: read from cache; on miss, read from database and populate cache. Most common; eventual consistency.
- **Event-driven invalidation**: invalidate cache entries when domain events occur. Most correct; most complex.

### Redis Data Structures for Common Patterns
```
Sessions:              HASH  key: session:{sessionId}
Rate limiting:         SORTED SET or String with INCR + EXPIRE
Leaderboard:           SORTED SET with ZADD/ZRANGE
Job queue:             List with LPUSH/BRPOP or Streams
Pub/Sub:               PUBLISH/SUBSCRIBE
Distributed lock:      String with SET NX PX (or Redlock for HA)
Caching:               String with SET EX
User online status:    SETEX with 30s TTL, refreshed on activity
```

## Database Migrations

Migrations are the highest-risk operation in deployment. Treat them with extreme care.

### Migration Rules
1. **Forward-only**: never write migrations that require backwards execution. Plan the rollback as a separate forward migration.
2. **Backward-compatible**: every migration must be compatible with both the currently deployed and the about-to-be-deployed application version. This enables zero-downtime deployments.
3. **Expand-Contract pattern**:
   - Phase 1 (Expand): add new column/table while keeping old column
   - Phase 2 (Migrate): deploy code using new column; backfill data
   - Phase 3 (Contract): remove old column in a subsequent deployment
4. **Non-blocking**: for large tables, use `ADD COLUMN ... DEFAULT NULL` (not `DEFAULT value`), build indexes `CONCURRENTLY`, and rename/drop columns with `LOCK TIMEOUT`.
5. **Idempotent**: migrations should be safe to run multiple times.
6. **Tested**: run migrations against a copy of production data in staging before production.
7. **Timed**: measure migration duration on production-scale data before applying.

### Migration Tools
- **Flyway** (JVM): battle-tested, version-based migrations
- **Liquibase**: XML/YAML/SQL-based, cross-platform
- **node-pg-migrate** / **db-migrate** (Node.js)
- **Alembic** (Python/SQLAlchemy)
- **golang-migrate** (Go)
- **Prisma Migrate** (Node.js + TypeScript)

### Long-Running Migration Checklist
For migrations on tables with > 1M rows:
- [ ] Test duration against production-scale data in staging
- [ ] Use `CONCURRENTLY` for index creation
- [ ] Set a `lock_timeout` to avoid blocking the entire application
- [ ] Schedule during low-traffic window
- [ ] Have the DBA or senior engineer on standby
- [ ] Know the rollback plan before starting

## Backup and Recovery

### Backup Strategy (3-2-1 Rule)
- **3** copies of the data
- **2** different storage media
- **1** offsite/off-cloud copy

### Backup Requirements
- **Full backup**: daily, retained for 30 days
- **Incremental/WAL backup**: every 5–15 minutes (point-in-time recovery)
- **Pre-migration backup**: always, immediately before any schema migration
- **Backup validation**: test restores weekly; a backup that cannot be restored is not a backup

### Recovery Objectives
- **RTO (Recovery Time Objective)**: how long can the system be down? (e.g., < 1 hour)
- **RPO (Recovery Point Objective)**: how much data loss is acceptable? (e.g., < 5 minutes)

Managed database services (AWS RDS, Supabase, Neon) handle backups automatically but you must still verify that restore works.

### Testing Backup Restores
```bash
# Monthly restore test procedure
1. Restore latest backup to a test environment
2. Verify data integrity (row counts, spot checks)
3. Run application integration tests against restored data
4. Document restore time — compare against RTO
5. Log the test result (pass/fail, restore time, data freshness)
```

## Data Privacy and Compliance

### PII Management
- Classify all data: public, internal, confidential, restricted (PII/sensitive)
- Encrypt all PII at rest
- Implement data access logging for sensitive fields
- Implement data retention policies: delete data that is no longer needed
- Support "right to erasure" (GDPR) — design for deletion from day one

### Data Masking
- Mask PII in all non-production environments: dev, staging, analytics
- Never copy production data to development without masking
- Tools: Faker-based data generation, Anonymizer scripts, cloud-native masking services

## Deliverables
- Database schema with documented design decisions
- All foreign key indexes created
- All query access patterns have corresponding indexes
- Slow query monitoring configured (pg_stat_statements)
- Migration scripts versioned and tested in CI
- Caching strategy documented and implemented
- Redis data structures chosen for each caching use case
- Backup configuration validated with a tested restore
- RTO/RPO defined and achievable with current backup strategy
- PII data classified and encryption strategy documented
- Data retention policies defined
- Non-production environments use masked data
