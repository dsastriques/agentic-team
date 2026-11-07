# PostgreSQL and Redis

## 1. Schema design (PostgreSQL)

- One schema (or database) per bounded context / service; no cross-context foreign keys or joins — integrate through APIs or events.
- Normalize to 3NF by default; denormalize only for measured read needs (and keep it consistent via triggers-free application logic or materialized views).
- **Constraints are invariants**: `NOT NULL`, `CHECK`, `UNIQUE`, foreign keys, and **exclusion constraints** for overlaps (`EXCLUDE USING gist (item_id WITH =, period WITH &&)` with `btree_gist` for bookings that must not overlap).
- Types: `uuid` keys (UUIDv7 for index locality, generated in the app or by the DB where supported) or `bigint generated always as identity`; `timestamptz` always (store UTC); `numeric(12,2)` or integer minor units for money plus a currency column; `text` with `CHECK` over `varchar(n)`; enums as lookup tables or `CHECK` constraints (Postgres enums are hard to change); `jsonb` for genuinely schemaless attributes, not to avoid modelling; `tstzrange` / `daterange` for periods.
- Naming: `snake_case`, plural or singular consistently, `<table>_<cols>_idx`, `<table>_<col>_fkey`.
- Audit columns: `created_at`, `updated_at`, `version` (optimistic locking → ETag); soft delete only when the requirements need it (and partial unique indexes then).
- Multi-tenancy: `tenant_id` on every row with composite keys and **row-level security** policies, or schema / database per tenant when isolation requirements demand it (ADR).
- PII: identify columns, minimize, encrypt sensitive fields at the application level when required, retention jobs per the requirements (GDPR erasure).

## 2. Indexing and queries

- Index for the queries you run: foreign keys used in joins, filter + sort columns (composite index in `=` then range then sort order), partial indexes for hot subsets (`WHERE status = 'active'`), covering indexes (`INCLUDE`) for index-only scans, GIN for `jsonb` / arrays / full-text, BRIN for huge append-only time series, pgvector HNSW for embeddings.
- Read the plan: `EXPLAIN (ANALYZE, BUFFERS)` on realistic data for every new query on a large table; watch seq scans on big tables, misestimates, sorts spilling to disk.
- `pg_stat_statements` for the top queries; `auto_explain` in staging.
- Avoid N+1 (eager loading per use case), `SELECT *` in hot paths, `OFFSET` on large tables (keyset / cursor pagination: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT $3`).
- Set `statement_timeout` and `lock_timeout` per role / session; `idle_in_transaction_session_timeout`.

## 3. Transactions and concurrency

- Default `READ COMMITTED`; use `SERIALIZABLE` (with retry on serialization failures) or explicit locks where invariants span rows and no constraint can express them.
- Optimistic locking with a `version` column for user edits (`UPDATE … WHERE id = $1 AND version = $2`), mapped to `If-Match` / 412.
- Pessimistic: `SELECT … FOR UPDATE` (short), `FOR UPDATE SKIP LOCKED` for job queues and outbox relays; advisory locks for singleton jobs.
- `INSERT … ON CONFLICT` for idempotent upserts.
- Keep transactions short; never call external services inside one.

## 4. Migrations without downtime

Expand → migrate → contract:
1. **Expand**: add nullable columns / new tables / new indexes (`CREATE INDEX CONCURRENTLY`), add constraints as `NOT VALID` then `VALIDATE CONSTRAINT` separately; deploy code that writes both shapes.
2. **Migrate**: backfill in batches (bounded, resumable, throttled), outside the deploy transaction.
3. **Contract**: switch reads, then drop old columns / tables in a later release.

Avoid in one step: renaming columns in use, changing types with a table rewrite, adding `NOT NULL` without a validated check constraint first, long locks (`lock_timeout` on migrations, retry). Every migration has a tested rollback (or a documented forward fix when rollback is impossible). Devops runs migrations as a separate pipeline step before the new version receives traffic.

## 5. Operations awareness

- Connection budget: `max_connections` is small — use a pooler (PgBouncer in transaction mode, or the platform's) and size app pools per instance × instances.
- Autovacuum tuned for hot tables; watch bloat and long transactions.
- Partitioning (declarative, by time or tenant) for very large tables, with retention by dropping partitions.
- Read replicas for heavy reads that tolerate lag (route explicitly).
- Backups, PITR and restore tests are devops' — record RPO / RTO needs in the ops handoff.
- Extensions used (pgvector, pg_trgm, btree_gist, pgcrypto) listed in the ops handoff.

## 6. Redis

**Use it for** speed and coordination, not as the system of record (unless an ADR chooses Redis persistence for a specific use).

| Use | Pattern |
|---|---|
| Cache | cache-aside: read Redis → miss → load from DB → `SET key value EX ttl`; invalidate (delete) on write after commit; version keys (`v2:booking:{id}`) when the shape changes |
| Stampede protection | short lock / single-flight on miss, early refresh with jitter, stale-while-revalidate |
| Rate limiting | fixed / sliding window with `INCR` + `EXPIRE` or a Lua script / sorted sets; token bucket for bursts; return 429 with `Retry-After` |
| Idempotency | `SET idem:{key} <state> NX EX 86400` storing status and the response hash; same key + different body → 422 |
| Sessions / short-lived tokens | hashes with TTL |
| Distributed lock | `SET lock:{name} <token> NX PX ttl`, release with compare-and-delete script; use only for efficiency, not correctness — guard correctness with DB constraints or fencing tokens |
| Queues / jobs | Redis Streams with consumer groups (ack, pending, claim) or the job library's Redis backend |
| Pub/sub | ephemeral notifications only (no delivery guarantee); Streams when delivery matters |
| Counters, leaderboards, schedules | `INCR`, sorted sets |

Practices:
- Key naming `<service>:<entity>:<id>[:<field>]`, documented; always a TTL unless the key is intentionally permanent.
- Serialize with a stable format (JSON with schema version, or msgpack); never pickle.
- Eviction policy chosen for the role (`allkeys-lru` for pure cache, `noeviction` for queues / locks) — tell devops; separate instances or databases for cache vs coordination when policies differ.
- Pipelines / `MGET` to avoid round trips; avoid `KEYS` (use `SCAN`); avoid big keys and hot keys.
- Timeouts and a circuit breaker on the client; the app degrades to the DB when the cache is down.
- Cache invalidation tested; cached data respects authorization (no cross-tenant or cross-user leaks in keys).
