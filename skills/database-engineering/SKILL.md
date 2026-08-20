---
name: database-engineering
description: Design schemas, write and review database migrations, set up migration tooling and versioning, and diagnose or fix database performance problems. Covers PostgreSQL, MySQL/MariaDB, and Redis. Use when the task involves DDL, ALTER TABLE, indexes, a migration file or migration tool (Alembic, Flyway, Liquibase, Prisma, Drizzle, Rails/Django/Laravel migrations, golang-migrate, Atlas), connection pooling, slow queries, EXPLAIN plans, locking/deadlocks, replication lag, cache key design, TTL/eviction policy, or choosing column types and keys.
---

# Database Engineering

You are executing the `database-engineering` skill. Read this file fully, then load only the reference files relevant to the task.

## Core stance

A database change is a **production change with no undo button**. Data loss and multi-minute table locks are the two failure modes that matter; almost every rule below exists to prevent one of them.

Two non-negotiables:

1. **Never write a migration that destroys data in the same deploy that stops using it.** Drops and renames are separate, later deploys. See `references/migrations.md`.
2. **Never claim a query or migration is fast without evidence.** Run `EXPLAIN (ANALYZE, BUFFERS)` / `EXPLAIN ANALYZE`, or state plainly that you could not measure it.

## Step 1 — Establish the environment before writing anything

Do not guess the engine, version, or tooling. Determine:

| Question | How to find out |
|---|---|
| Engine and exact version | `SELECT version();` (PG), `SELECT VERSION();` (MySQL), `redis-server --version` / `INFO server`. Failing live access: Docker Compose, CI config, `Dockerfile`, cloud provider docs. |
| Migration tool and version convention | Look for `migrations/`, `db/migrate/`, `alembic/`, `prisma/`, `flyway.conf`, `atlas.hcl`, `liquibase.properties` |
| Table size and traffic | `SELECT reltuples FROM pg_class` / `information_schema.TABLES`. A rule that is optional at 10k rows is mandatory at 100M. |
| Managed or self-hosted | RDS/Aurora/Cloud SQL/PlanetScale each restrict superuser ops, extensions, and some `ALTER` forms. |

Version matters more than people expect: `ADD COLUMN ... DEFAULT` is instant on PG 11+ and a full rewrite before it; MySQL 8.0.12+ has instant DDL for some operations, 5.7 does not.

If the engine is genuinely unknowable and the answer differs by engine, say so and give the answer for the most likely one rather than blocking.

## Step 2 — Route to the right reference

Load only what you need:

- **Any migration, DDL, or versioning question** → `references/migrations.md` (expand/contract, safety table per operation, tooling, rollback, CI gates)
- **PostgreSQL specifics** → `references/postgresql.md` (types, indexes, lock levels, `pg_stat_*`, vacuum, partitioning, extensions)
- **MySQL / MariaDB specifics** → `references/mysql.md` (InnoDB, online DDL vs. gh-ost/pt-osc, charsets, replication, `performance_schema`)
- **Redis** → `references/redis.md` (data structure choice, key design, TTL/eviction, persistence, cluster, cache patterns, anti-patterns)
- **Slow queries, indexes, plans, pooling, capacity** → `references/performance.md` (a diagnosis order that works, index design, N+1, connection pools)

## Step 3 — Schema design defaults

Apply unless the task gives a reason not to.

**Keys.** Surrogate primary key on every table. Prefer `BIGINT` identity/auto-increment, or UUIDv7 (time-ordered) when IDs must be client-generated or non-enumerable. Avoid UUIDv4 as a clustered/primary key — random insert order fragments the B-tree badly, especially in InnoDB where the PK *is* the table. Add a separate unique constraint for the real business key.

**Types.**
- Money → `NUMERIC`/`DECIMAL`, never `FLOAT`/`DOUBLE`.
- Timestamps → `TIMESTAMPTZ` (PG) or `DATETIME` storing UTC (MySQL; `TIMESTAMP` caps at 2038 and applies session timezone conversion). Store UTC, convert at the edges.
- Text → PG `TEXT` with a `CHECK` on length if needed; MySQL `VARCHAR(n)` with `utf8mb4` (never `utf8`, which is 3-byte and cannot hold emoji or many CJK characters).
- Enumerated values → a lookup table with an FK, or a `CHECK`/`ENUM`. PG native `ENUM` is fine but adding values is easy while removing/reordering them is not; MySQL `ENUM` reorders are worse. When the set will churn, use a lookup table.
- Booleans → real `BOOLEAN` (MySQL: `TINYINT(1)`), not `'Y'`/`'N'`.
- JSON → `JSONB` (PG) / `JSON` (MySQL) for genuinely open-ended attributes only. If you query or index a field regularly, promote it to a real column. JSON columns are where schemas go to rot.

**Constraints.** `NOT NULL` by default; nullability should be a deliberate decision, since `NULL` silently poisons `=`, aggregates, and unique semantics. Foreign keys on by default with explicit `ON DELETE` behavior — omit them only for a stated reason (extreme write throughput, sharding, cross-service ownership), and then document how integrity is enforced instead. Add `CHECK` constraints for invariants the application believes in.

**Naming.** Follow whatever the existing schema does. Absent a convention: `snake_case`, plural table names, `<table>_id` for FKs, `idx_<table>_<cols>` / `uq_<table>_<cols>` / `fk_<table>_<ref>` for constraints. Never rely on auto-generated constraint names in migrations you plan to reverse.

**Soft deletes.** Only when the product actually needs recovery or audit. `deleted_at TIMESTAMPTZ NULL` plus partial indexes (`WHERE deleted_at IS NULL`), and be aware every query and unique constraint now has to account for it.

## Step 4 — Writing the change

1. Write the forward migration and the rollback together. If the rollback is "restore from backup," write that in a comment in the migration file — do not leave it implied.
2. State the lock the migration takes and for how long, in a comment at the top. For anything that touches a large table, this is required, not a nicety.
3. One logical change per migration file. Mixed DDL/DML migrations are hard to reason about and often can't run in a single transaction anyway.
4. Backfills go in their own step, batched with a sleep between batches, never as one `UPDATE` over the whole table.
5. Set a lock timeout so a blocked migration fails fast instead of queueing every reader behind it:
   - PG: `SET lock_timeout = '5s'; SET statement_timeout = '...';`
   - MySQL: `SET SESSION lock_wait_timeout = 5;`
6. Test on a copy with production-scale data, or say explicitly that you have not.

## Step 5 — Report honestly

When you finish, state:

- What locks the change takes and the expected duration at the table's real size.
- Whether it is backward compatible with the currently deployed application code.
- The rollback procedure, and whether it is lossless.
- What you measured versus what you estimated. Do not present an estimate as a measurement.

If you could not verify something — no DB access, unknown row counts, no way to run `EXPLAIN` — say which check is outstanding and what the user should run. An unverified migration presented as safe is worse than one flagged as unverified.

## Red flags — stop and raise these

- `DROP TABLE`, `DROP COLUMN`, or `TRUNCATE` in a migration whose application code was deployed in the same release
- A rename of any column or table still referenced by running code
- Adding a `UNIQUE` or `NOT NULL` constraint to a populated table without a validated backfill first
- A migration that will hold `ACCESS EXCLUSIVE` / a metadata lock on a hot table for more than a couple of seconds
- `FLUSHALL`, `KEYS *`, or an unbounded `SMEMBERS`/`HGETALL` against production Redis
- Any index created without `CONCURRENTLY` (PG) or `ALGORITHM=INPLACE`/gh-ost (MySQL) on a live table
- Credentials, connection strings, or dumps about to be written to a repo or an external service
