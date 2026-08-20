# Database Performance

## Diagnose in this order

Skipping straight to "add an index" is how you end up with 14 indexes and the same latency. Work down the list:

1. **Measure where time actually goes.** Application APM trace first: is it query time, connection acquisition, serialization, or N+1 round trips? Query time is often not the answer.
2. **Find the top queries by total time** — `pg_stat_statements`, or `events_statements_summary_by_digest`. Sort by cumulative time, never by mean.
3. **Read the plan** for the worst offenders (`EXPLAIN (ANALYZE, BUFFERS)` / `EXPLAIN ANALYZE`).
4. **Check the obvious pathologies**: missing index, N+1, `SELECT *` pulling large columns, `OFFSET` deep pagination, unnecessary `DISTINCT`/`ORDER BY`, a query fetching 10k rows to display 20.
5. **Check statistics and bloat** before believing a bad plan is the planner's fault.
6. **Check contention**: locks, deadlocks, connection pool saturation, replication lag.
7. **Only then** consider hardware, config, caching, denormalization, or sharding.

## The five problems that cause most incidents

**N+1 queries.** 1 query for the list, then 1 per row. Invisible in single-query metrics, devastating in aggregate. Fix with eager loading (`JOIN`, `IN (...)`, ORM `select_related`/`joinedload`/`includes`/`with`) or a DataLoader-style batcher. Detect by counting queries per request in tests and failing above a threshold.

**Missing or unusable index.** An index exists but the query can't use it:
- Function on the column: `WHERE lower(email) = ?` needs an index on `lower(email)` (PG expression index, MySQL generated column).
- Type mismatch: `WHERE varchar_col = 123` or a `bigint` column compared to a `text` parameter forces a cast and disables the index.
- Leading wildcard: `LIKE '%foo'` cannot use a B-tree. Use trigram (PG `pg_trgm`) or a search engine.
- Collation mismatch on a join (MySQL) — silently disables index use.
- `OR` across columns often can't use either index; `UNION ALL` of two indexed queries usually can.

**Deep pagination.** `LIMIT 20 OFFSET 100000` reads and discards 100,000 rows. Use keyset pagination:
```sql
SELECT * FROM orders WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC LIMIT 20;
```
Requires an index on `(created_at DESC, id DESC)`. Costs you random page access; worth it above a few thousand rows.

**Over-fetching.** `SELECT *` on a table with a large `TEXT`/`JSONB` column, or fetching full rows when you need a count. Also: counting exactly (`SELECT count(*)`) when an estimate would do — `reltuples` (PG) or a maintained counter is O(1).

**Lock contention.** Long transactions, an `UPDATE` without a usable index locking far more rows than intended, hot-row contention (a single counter row every request updates), and DDL queued behind a long reader. Fix by shortening transactions, indexing `WHERE` clauses of writes, sharding hot counters, and setting `lock_timeout`/`lock_wait_timeout`.

## Index design rules

1. Index what you **filter, join, and sort on** — in that order of priority.
2. **Composite order**: equality columns first, then the range/sort column. `(status, created_at)` for `WHERE status = 'x' ORDER BY created_at`.
3. **Prefix rule**: an index on `(a, b, c)` serves `a`, `(a,b)`, `(a,b,c)` — but not `b` alone. So don't create `(a)` when `(a,b)` exists.
4. **Selectivity**: an index on a boolean with a 50/50 split is nearly useless as a standalone index — but valuable as a *partial* index predicate.
5. **Every index taxes every write** and consumes cache. Drop unused ones (check on replicas too).
6. **Foreign keys need indexes on the child side** — most databases don't create them automatically (MySQL does; PostgreSQL does not). Missing ones make parent deletes and updates catastrophically slow and cause lock escalation.
7. Prefer one well-chosen composite index over three single-column ones.

## Connection pooling

The most common production failure isn't a slow query — it's connection exhaustion turning a slow query into a total outage.

- Pool size ≈ `cores × 2` for the database, not per app instance. Little's Law: with 5ms queries, a pool of 10 serves 2000 req/s. Large pools mostly add queueing and context-switching.
- Count the total: `app_instances × pool_size + workers + cron + admin` must fit under `max_connections` with headroom.
- Set an **acquisition timeout** so a saturated pool fails fast rather than piling up.
- PostgreSQL needs PgBouncer at scale (process per connection); MySQL tolerates more connections but still benefits from ProxySQL.
- Serverless/Lambda breaks pooling assumptions entirely — use a proxy (RDS Proxy, PgBouncer, Data API) or you will exhaust connections during a scale-out.
- Set `statement_timeout` / `max_execution_time` globally so a runaway query can't hold resources forever.

## Query rewriting that reliably helps

- `EXISTS` instead of `IN (subquery)` when the subquery is large; `IN` with a literal list is fine and fast.
- `UNION ALL` instead of `UNION` unless you actually need deduplication (which requires a sort).
- Push `LIMIT` down into subqueries before joining.
- Aggregate in the database, not the application — but return aggregates, not the rows behind them.
- Replace correlated subqueries in the `SELECT` list with a `LATERAL` join (PG) or a pre-aggregated join.
- Batch writes: one `INSERT ... VALUES (...), (...), (...)` or `COPY`/`LOAD DATA INFILE` beats a thousand single inserts by orders of magnitude.
- Use `INSERT ... ON CONFLICT DO UPDATE` (PG) / `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL) instead of select-then-insert, which races.
- Materialize expensive aggregates (PG materialized views with `REFRESH ... CONCURRENTLY`, or a summary table updated incrementally) when the query is read far more often than the data changes.

## Denormalization and caching — when it's actually justified

In order of preference, exhaust each before the next:

1. Fix the query, index, or access pattern.
2. Cache the result (Redis) with a TTL and a clear invalidation rule.
3. Maintain a derived column or summary table, updated in the same transaction as the source (or by a reliable job with a reconciliation check).
4. Read replicas for read-heavy workloads — accepting stale reads, and routing read-after-write to the primary.
5. Partitioning, for tables where time-based retention or pruning is the real need.
6. Sharding. Last resort: it costs you cross-shard joins, distributed transactions, and rebalancing forever.

Every step past 1 adds a way for data to be wrong. Say so when recommending one.

## Load testing and verification

- Test against **production-scale data**. A plan on 1,000 rows tells you nothing about 100 million — the planner switches strategies entirely.
- Test with realistic **concurrency**; lock contention and pool exhaustion only appear under parallel load.
- Measure p95/p99, not the mean. The mean hides the queries that generate support tickets.
- Compare plans before and after a change; verify the index is actually used, don't assume it.
- Warm the cache before measuring, or measure cold deliberately — but know which you did and say so.

## Metrics worth alerting on

| Signal | PostgreSQL | MySQL |
|---|---|---|
| Query latency p99 | app APM + `pg_stat_statements` | `events_statements_summary_by_digest` |
| Cache hit ratio | `pg_stat_database.blks_hit / (blks_hit+blks_read)` — target >99% | `Innodb_buffer_pool_read_requests` vs `_reads` |
| Connection saturation | `numbackends` vs `max_connections` | `Threads_connected` vs `max_connections` |
| Replication lag | `replay_lag` | applier status + `Seconds_Behind_Source` |
| Locking | `pg_blocking_pids()` non-empty, `deadlocks` counter | `sys.innodb_lock_waits`, `Innodb_row_lock_time_avg` |
| Bloat / dead tuples | `n_dead_tup`, `last_autovacuum` | table fragmentation vs `data_free` |
| Transaction age | `age(datfrozenxid)`, longest `xact_start` | oldest `INNODB_TRX.trx_started` |
| Disk | free space, IOPS saturation | same |

Alert on trend and saturation, not just thresholds — connection count climbing steadily is a better signal than connection count crossing 80%.
