# Migrations & Schema Versioning

## The expand / contract pattern

The only reliable way to change a schema under a running application. Every breaking change becomes a sequence of individually-safe deploys:

1. **Expand** — add the new structure. Nullable, defaulted, additive only. Old code is unaffected.
2. **Backfill** — populate the new structure in batches. Idempotent and resumable.
3. **Migrate reads** — deploy code that reads the new structure, still writing both.
4. **Migrate writes** — deploy code that writes only the new structure.
5. **Contract** — after a soak period (days, not minutes) and once rollback to the old code is off the table, drop the old structure.

Rule of thumb: **the deploy that stops using a column and the deploy that drops it are never the same deploy.** If you roll back the application, the column must still be there.

### Rename a column: worked example

Never `ALTER TABLE ... RENAME COLUMN` on a live table.

```sql
-- Migration 1 (expand)
ALTER TABLE users ADD COLUMN email_address TEXT;
```
```sql
-- Migration 2 (backfill, batched, run repeatedly until 0 rows affected)
UPDATE users SET email_address = email
WHERE email_address IS NULL AND id IN (
  SELECT id FROM users WHERE email_address IS NULL ORDER BY id LIMIT 5000
);
```
Application writes both columns during this window (a trigger works too, but application-level dual-write is easier to reason about and to roll back).
```sql
-- Migration 3 (after backfill verified: SELECT count(*) FROM users WHERE email_address IS NULL AND email IS NOT NULL; -- must be 0)
ALTER TABLE users ALTER COLUMN email_address SET NOT NULL;  -- see NOT NULL notes below
```
```sql
-- Migration 4 (contract, a later release)
ALTER TABLE users DROP COLUMN email;
```

## Operation safety table

"Safe" = no full rewrite, no long exclusive lock. Always verify against your version.

| Operation | PostgreSQL | MySQL 8 (InnoDB) |
|---|---|---|
| `ADD COLUMN` nullable, no default | Safe | Safe (INSTANT) |
| `ADD COLUMN` with constant default | Safe on 11+; rewrite before | INSTANT on 8.0.12+; else INPLACE rewrite |
| `ADD COLUMN` with volatile default (`now()`, `random()`) | **Rewrite** — add nullable, backfill, then set default | **Rewrite** |
| `ADD COLUMN NOT NULL` (no default) | Fails if rows exist | Fails if rows exist |
| `DROP COLUMN` | Safe (metadata only), but destroys data | INPLACE, rebuilds table |
| `RENAME COLUMN` / `RENAME TABLE` | Fast, but **breaks running code** | Fast, but **breaks running code** |
| Widen `VARCHAR(n)` → larger n | Safe | INPLACE if staying ≤255 or >255 bracket; crossing the 255-byte boundary rewrites |
| Narrow a type / change type | Rewrite + `ACCESS EXCLUSIVE` | Rewrite (COPY) |
| `int` → `bigint` on PK | Rewrite; use expand/contract with a new column | Rewrite; use gh-ost |
| `CREATE INDEX` | **Use `CONCURRENTLY`** — plain form blocks writes | INPLACE, allows concurrent DML |
| `DROP INDEX` | `DROP INDEX CONCURRENTLY` | INPLACE |
| Add `NOT NULL` | Backfill, then `ADD CONSTRAINT ... CHECK (c IS NOT NULL) NOT VALID` → `VALIDATE CONSTRAINT` → `SET NOT NULL` (PG 12+ recognizes the validated CHECK and skips the scan) | Rewrite; gh-ost for large tables |
| Add `FOREIGN KEY` | `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` (only `SHARE ROW EXCLUSIVE`) | INPLACE with `foreign_key_checks=0` risk; prefer validating separately |
| Add `UNIQUE` | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... USING INDEX` | Build via gh-ost or accept the rebuild |
| Change default | Safe, metadata only | Safe |
| Add `CHECK` | `NOT VALID` then `VALIDATE` | Rewrite unless `NOT ENFORCED` |

## Backfills

```sql
-- Batched, resumable, index-driven. Never a single UPDATE over 50M rows.
UPDATE orders SET status_v2 = status
WHERE id BETWEEN :lo AND :hi AND status_v2 IS NULL;
```

- Batch by primary key range, not `LIMIT`/`OFFSET` (which degrades quadratically).
- 1k–10k rows per batch; sleep 50–500ms between batches; watch replication lag and back off when it grows.
- Run backfills **outside** the migration tool when they take more than a minute — a migration that runs for an hour blocks your deploy pipeline and every other migration behind it. A one-off script or job is better.
- Make them idempotent (`WHERE new_col IS NULL`) so a crash means "run it again," not "figure out where it stopped."
- Verify completion with a count before adding any constraint that depends on it.

## Transactional DDL

- **PostgreSQL** wraps DDL in transactions — a failed migration rolls back cleanly. Exceptions that cannot run inside a transaction: `CREATE INDEX CONCURRENTLY`, `DROP INDEX CONCURRENTLY`, `ALTER TYPE ... ADD VALUE` (pre-12), `VACUUM`, `CREATE DATABASE`. Most tools have a flag for this (Alembic: separate migration with autocommit; Rails: `disable_ddl_transaction!`; golang-migrate: `-- +migrate NoTransaction`).
- **MySQL** has **no transactional DDL**. A migration that fails halfway leaves the schema partially changed. Therefore: one DDL statement per migration file, always, and a manual recovery path written down.

## Versioning conventions

- **Timestamp-prefixed filenames** (`20260815120000_add_email_address.sql`) beat sequential integers — parallel branches don't collide.
- **Immutable history.** Never edit a migration that has run anywhere but your laptop. Fix forward with a new migration.
- **Checksums.** Flyway/Liquibase verify that applied migrations haven't changed; treat a checksum failure as a real incident, not something to `repair` past reflexively.
- **Forward-only in production** is the honest default. Down migrations are useful in development and mostly a fiction in production (they can't restore dropped data). Write them anyway for reversible operations; for destructive ones, write a comment saying recovery is restore-from-backup.
- **One source of truth.** Either the migrations generate the schema, or a declarative schema file generates the migrations (Prisma, Atlas, Drizzle). Never both hand-edited.
- Commit a generated `schema.sql` / `structure.sql` snapshot alongside migrations — it makes schema diffs reviewable in PRs.

## Tooling notes

| Tool | Notes |
|---|---|
| **Alembic** (Python) | `--autogenerate` misses type changes, index renames, and server defaults. Always read and edit the generated file. Set `compare_type=True`, `compare_server_default=True`. |
| **Flyway** | `V` versioned, `R` repeatable, `U` undo (paid). Checksums enforced. Good for plain SQL shops. |
| **Liquibase** | XML/YAML changesets, DB-agnostic; verbose. Preconditions and contexts are its strength. |
| **Rails / Django / Laravel** | Framework migrations are fine; the danger is that `remove_column` / `RemoveField` are one line and irreversible. `strong_migrations` (Rails) and `django-safemigrate` catch unsafe patterns in CI. |
| **Prisma** | `migrate dev` for local, `migrate deploy` for production. Never `db push` against production. Shadow-database drift errors usually mean someone hand-edited the DB. |
| **Drizzle** | `drizzle-kit generate` diffs the TS schema; review the SQL, it will happily emit a drop-and-recreate. |
| **golang-migrate** | Paired `.up.sql`/`.down.sql`. Dirty state after a failure requires manual `force`. |
| **Atlas** | Declarative + versioned, has a real lint engine (`atlas migrate lint`) that flags destructive and blocking changes. Best-in-class for CI gating. |
| **gh-ost / pt-online-schema-change** | MySQL only. See `mysql.md`. |

## CI gates worth adding

1. **Lint migrations for destructive/blocking operations** — `atlas migrate lint`, `squawk` (PG), `strong_migrations` (Rails). Fail the build on `DROP`, missing `CONCURRENTLY`, or `ALTER TABLE` type changes unless a reviewer overrides.
2. **Apply every migration against a fresh DB** in CI, then apply against a restored production-shaped dump to catch data-dependent failures (duplicate rows breaking a new unique index).
3. **Diff the resulting schema** against the committed schema snapshot; fail on drift.
4. **Check backward compatibility**: run the previous release's test suite against the new schema. This is the single most effective gate against "the deploy rolled back and everything broke."
5. **Time the migration** against production-scale data and fail if it exceeds a threshold.

## Zero-downtime deploy ordering

```
deploy N   : expand migration → app code that tolerates both shapes
deploy N+1 : app code that uses the new shape
   ... soak ...
deploy N+2 : contract migration
```

The application must be forward- and backward-compatible with the schema for the entire window between N and N+2. Enforce this by asking, for every migration: *if the app rolls back to the previous version right now, does it still work?*
