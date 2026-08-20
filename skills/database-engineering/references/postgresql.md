# PostgreSQL

## Locks: what each DDL actually takes

Every `ALTER TABLE` takes `ACCESS EXCLUSIVE` unless noted. That lock blocks *everything*, including `SELECT`. The danger is rarely the DDL's own duration — it is that the DDL waits behind one long-running transaction, and every subsequent query queues behind the DDL. A 5ms `ALTER` can take down a site for minutes this way.

Always:
```sql
SET lock_timeout = '3s';
SET statement_timeout = '30s';
ALTER TABLE ...;
```
Failing fast and retrying is correct behavior. Retry in a loop if needed.

| Lock | Blocks | Taken by |
|---|---|---|
| `ACCESS EXCLUSIVE` | everything | most `ALTER TABLE`, `DROP`, `TRUNCATE`, `REINDEX`, `VACUUM FULL` |
| `SHARE ROW EXCLUSIVE` | writes + DDL | `ADD FOREIGN KEY`, `CREATE TRIGGER` |
| `SHARE UPDATE EXCLUSIVE` | DDL + vacuum only | `CREATE INDEX CONCURRENTLY`, `VALIDATE CONSTRAINT`, `ANALYZE`, autovacuum |
| `ROW EXCLUSIVE` | DDL | `INSERT`/`UPDATE`/`DELETE` |

Find what is blocking a migration:
```sql
SELECT pid, state, wait_event_type, wait_event,
       now() - xact_start AS xact_age, left(query, 120) AS query
FROM pg_stat_activity
WHERE state <> 'idle' AND backend_type = 'client backend'
ORDER BY xact_start;

-- blocking tree
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       left(blocked.query,80) AS blocked_query, left(blocking.query,80) AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```
`idle in transaction` sessions are the usual culprit. Set `idle_in_transaction_session_timeout`.

## Indexes

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_created
  ON orders (customer_id, created_at DESC);
```

`CONCURRENTLY` cannot run inside a transaction, takes roughly 2–3× as long, and can leave an `INVALID` index if it fails. Check and clean up:
```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
-- then: DROP INDEX CONCURRENTLY <name>; and recreate
```

**Choosing an index type:**

| Type | Use for |
|---|---|
| B-tree (default) | equality, ranges, sorting, `LIKE 'prefix%'` |
| GIN | `JSONB` containment (`@>`), array membership, full-text search, `pg_trgm` fuzzy matching |
| GiST | geometric, ranges, exclusion constraints, nearest-neighbour |
| BRIN | huge tables with strong physical correlation (append-only time-series). Tiny index, cheap, only helps when correlation holds. |
| Hash | equality only; rarely worth it over B-tree |

**Composite index column order:** equality predicates first, then the range/sort column. `WHERE customer_id = ? AND created_at > ?` wants `(customer_id, created_at)`, not the reverse. A composite index on `(a, b)` also serves queries on `a` alone — so don't create both.

**Partial indexes** are the highest-leverage and most under-used feature:
```sql
CREATE INDEX CONCURRENTLY idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';        -- tiny, hot, exactly the queries you run
CREATE UNIQUE INDEX CONCURRENTLY uq_users_email_active ON users (lower(email))
  WHERE deleted_at IS NULL;        -- uniqueness that respects soft deletes
```

**Covering indexes** enable index-only scans: `CREATE INDEX ... ON t (a, b) INCLUDE (c)`. Only pays off when the visibility map is current — i.e. the table is well-vacuumed.

**Find unused and duplicate indexes** (each one taxes every write):
```sql
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes JOIN pg_index USING (indexrelid)
WHERE idx_scan < 50 AND NOT indisunique
ORDER BY pg_relation_size(indexrelid) DESC;
```
Check `stats_reset` on `pg_stat_database` first — a recent reset makes every index look unused. And check replicas separately; an index unused on the primary may serve read traffic.

## Reading EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, FORMAT TEXT) SELECT ...;
```
`ANALYZE` executes the query — wrap writes in a transaction you roll back.

What to look for, in order:
1. **Estimated vs. actual rows** off by >10× → stale statistics, or a correlation the planner can't see. Fix with `ANALYZE`, a higher `default_statistics_target` on the column, or `CREATE STATISTICS` for correlated columns.
2. **`Seq Scan` on a large table** with a selective predicate → missing or unusable index (function on the column, type mismatch, `LIKE '%x'`).
3. **`Rows Removed by Filter`** high → the index isn't selective enough; consider a composite or partial index.
4. **`Nested Loop`** with a large outer row count → usually a bad estimate; the fix is the estimate, not `enable_nestloop=off`.
5. **Sort with `Sort Method: external merge Disk`** → raise `work_mem` (per-node, per-connection — be careful) or index the sort.
6. **`Buffers: read=` large** → not in cache; that's your real cost.
7. **Parallel workers launched < planned** → `max_parallel_workers_per_gather` exhausted.

Paste plans into explain.dalibo.com or explain.depesz.com for large ones.

## Statistics and vacuum

Autovacuum has two jobs: reclaim dead tuples, and update the visibility map / statistics. Neglected autovacuum is the most common cause of a database that "got slow for no reason."

```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup,0), 3) AS dead_ratio,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
```

- Default `autovacuum_vacuum_scale_factor = 0.2` means a 100M-row table waits for 20M dead tuples. Lower it per-table on big/hot tables: `ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.02, autovacuum_analyze_scale_factor = 0.01);`
- `VACUUM FULL` rewrites the table under `ACCESS EXCLUSIVE`. Never on production without a window; use `pg_repack` instead.
- Watch transaction ID wraparound: `SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;` — past ~1.5B, treat it as an incident.
- Long-running transactions and stale replication slots prevent vacuum from removing dead rows anywhere in the DB. Check `pg_replication_slots` for inactive slots.

Run `ANALYZE` explicitly after a large backfill or bulk load — autoanalyze may not fire in time and the next queries will plan against pre-load statistics.

## Bloat and `pg_stat_statements`

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- also needs shared_preload_libraries

SELECT left(query, 100) AS query, calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
```
Optimize by **total** time, not mean. A 3ms query run 10M times costs more than a 20-second report run once.

## Types worth knowing

- `TIMESTAMPTZ` always; it stores UTC and converts on display. `TIMESTAMP` (without tz) is a source of bugs.
- `TEXT` == `VARCHAR` in performance; use `TEXT` + `CHECK`.
- `NUMERIC` for money; `DOUBLE PRECISION` never.
- `JSONB` (binary, indexable, deduplicates keys) over `JSON` (text, preserves order/whitespace) in essentially all cases.
- Arrays are fine for small, bounded, read-mostly lists; a join table is right whenever you need to query, index, or constrain the elements.
- Range types (`tstzrange`) + `EXCLUDE USING gist` gives you real "no double booking" constraints in the database.
- `uuid` type, not `TEXT`. Generate v7 in the app, or `gen_random_uuid()` (pgcrypto/PG13+ builtin) for v4.
- Domains for reused constrained types (`CREATE DOMAIN email AS TEXT CHECK (VALUE ~ '^[^@]+@[^@]+$')`).

## Partitioning

Worth it above roughly 100GB per table, or when you need to drop old data cheaply.

```sql
CREATE TABLE events (id bigserial, occurred_at timestamptz NOT NULL, ...)
  PARTITION BY RANGE (occurred_at);
CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```
- The partition key must be in the primary key and in every unique constraint. This is the constraint that most often kills the idea.
- Dropping a partition is instant; deleting a month of rows is not. That's usually the real motivation.
- Queries must filter on the partition key to get pruning — otherwise every partition is scanned and you've made things worse.
- Use `pg_partman` to automate creation/retention rather than hand-rolling cron.

## Extensions worth reaching for

`pg_stat_statements` (always), `pg_trgm` (fuzzy/ILIKE search), `btree_gin`/`btree_gist` (mixed-type composite GIN/GiST), `pgcrypto`, `postgis`, `pg_repack` (online bloat removal), `pgvector` (embeddings), `pg_partman`, `auto_explain` (log plans for slow queries in production). Managed providers restrict the list — check before designing around one.

## Connections

Postgres uses a process per connection; each costs ~5–10MB and context-switching. Beyond a few hundred, throughput falls off a cliff.

- Use **PgBouncer** (or the provider's pooler) in `transaction` mode. Under transaction pooling, session-level features break: prepared statements (unless PgBouncer 1.21+ with `max_prepared_statements`), `SET` outside a transaction, advisory locks, `LISTEN/NOTIFY`, temp tables.
- Size pools by `connections ≈ cores × 2 + effective_spindle_count`, not by "how many app servers do we have."
- Total connections across all app instances must stay under `max_connections`. Count them — running out mid-incident is a classic self-inflicted outage.

## Replication

- Streaming replicas: monitor `pg_stat_replication.replay_lag` and `write_lag`.
- Read-after-write on a replica returns stale data. Either route the read to the primary, or use `pg_current_wal_lsn()` / `pg_last_wal_replay_lsn()` to wait for the LSN.
- `hot_standby_feedback = on` prevents query cancellation on replicas but lets replica queries hold back vacuum on the primary. Pick your poison deliberately.
- Logical replication for version upgrades and selective table replication; note it does not replicate DDL, sequences' current values, or `TRUNCATE` by default.
