# MySQL / MariaDB

Assume InnoDB. MyISAM has no transactions, no FKs, and table-level locks — if you find it, migrating away is usually the fix.

## InnoDB fundamentals that drive design

- **The primary key is the table.** Rows are stored in PK order in the clustered index. Consequences:
  - A random PK (UUIDv4) causes page splits and fragmentation on every insert. Use `BIGINT AUTO_INCREMENT`, or UUIDv7 / `UUID_TO_BIN(uuid, 1)` (swapped, time-ordered) stored as `BINARY(16)`.
  - Every secondary index stores the PK as its pointer, so a wide PK inflates every index on the table.
- **Secondary index lookups are two-step**: index → PK → clustered index. Covering indexes (all needed columns in the index) skip the second step; look for `Using index` in `EXPLAIN`.
- Row format `DYNAMIC`, `innodb_file_per_table=ON` (defaults in 8.0).

## Online DDL

MySQL 8.0 supports three algorithms; specify them explicitly so a surprise table copy fails loudly instead of running for an hour:

```sql
ALTER TABLE orders ADD COLUMN notes TEXT, ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_customer (customer_id), ALGORITHM=INPLACE, LOCK=NONE;
```

| Algorithm | Meaning |
|---|---|
| `INSTANT` | Metadata only (8.0.12+). Adding a column at the end, adding a virtual column, renaming a column, setting a default. |
| `INPLACE` | Rebuilds in place; DML usually allowed (`LOCK=NONE`). Adding/dropping indexes and FKs. |
| `COPY` | Full table copy, table effectively read-only. Type changes, PK changes, charset changes. |

Even `INSTANT`/`INPLACE` need a brief **metadata lock (MDL)** at start and end. A single long-running transaction holding an MDL will block the ALTER, and then every new query on that table blocks behind it. Set `SET SESSION lock_wait_timeout = 5;` and retry.

Check what's blocking:
```sql
SELECT * FROM performance_schema.metadata_locks WHERE lock_status = 'PENDING';
SELECT * FROM sys.schema_table_lock_waits;
SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;
```

MySQL has **no transactional DDL** — one DDL statement per migration file, and a written recovery path.

## gh-ost and pt-online-schema-change

For `COPY`-algorithm changes on a large table, or any ALTER whose replica lag would be unacceptable:

- **gh-ost** (preferred): reads the binlog rather than using triggers, so it adds no write-path overhead, is throttleable, and is pausable/resumable. Requires row-based binlog.
  ```bash
  gh-ost --host=... --database=shop --table=orders \
    --alter="MODIFY id BIGINT NOT NULL AUTO_INCREMENT" \
    --max-load=Threads_running=25 --critical-load=Threads_running=100 \
    --chunk-size=1000 --max-lag-millis=1500 \
    --allow-on-master --initially-drop-ghost-table --execute
  ```
  The cut-over is a brief table-rename lock. Run it with `--postpone-cut-over-flag-file` so you choose the moment.
- **pt-online-schema-change**: trigger-based, works where binlog access isn't available; the triggers add latency to every write and it interacts badly with existing triggers and some FK setups (`--alter-foreign-keys-method` is a real decision, not a flag to guess at).
- Managed platforms (PlanetScale, Vitess) do this for you via branch/deploy-request workflows — use the platform mechanism instead.

## Charset and collation

- `utf8mb4` always. MySQL's `utf8` is a 3-byte subset that cannot store emoji or many CJK characters — inserting one truncates or errors.
- Collation: `utf8mb4_0900_ai_ci` (8.0 default, accent- and case-insensitive) or `utf8mb4_bin` for exact matching. MariaDB: `utf8mb4_uca1400_ai_ci` or `utf8mb4_unicode_ci`.
- **Mismatched collations between joined columns silently disable index usage** on the join. This is a very common invisible performance bug — check `SHOW FULL COLUMNS` on both sides when a join won't use its index.
- Changing a table's charset is a `COPY` operation. Fix new tables and migrate old ones deliberately.
- Index key length: an index on `VARCHAR(255)` utf8mb4 is 1020 bytes, near the 3072-byte limit; prefix indexes (`INDEX (col(50))`) work but can't serve `ORDER BY` or covering scans.

## Types

- `DATETIME` (8 bytes, no timezone conversion, range to 9999) over `TIMESTAMP` (4 bytes, 2038 limit, converts using `time_zone`). Store UTC in `DATETIME`.
- `DECIMAL(19,4)` for money.
- `BOOLEAN` is an alias for `TINYINT(1)` — fine.
- `ENUM` is compact but altering the value list is a table change and ordering is positional; a lookup table is safer for anything that churns.
- `BINARY(16)` for UUIDs via `UUID_TO_BIN(u, 1)` / `BIN_TO_UUID(b, 1)` — the swap flag makes them time-ordered.
- Avoid `TEXT`/`BLOB` in tables you sort or filter heavily; off-page storage means extra I/O and no in-memory temp tables.
- Generated columns + an index on them are the MySQL answer to expression indexes:
  ```sql
  ALTER TABLE users ADD COLUMN email_lower VARCHAR(255)
    GENERATED ALWAYS AS (LOWER(email)) STORED, ADD INDEX (email_lower);
  ```

## EXPLAIN

```sql
EXPLAIN FORMAT=JSON SELECT ...;
EXPLAIN ANALYZE SELECT ...;   -- 8.0.18+, actually runs it
```

`type` column, best to worst: `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`. `ALL` on a big table is a full scan; `index` is a full index scan, only marginally better.

Red flags in `Extra`: `Using filesort` (sort not served by an index), `Using temporary` (materialized intermediate — common with `GROUP BY` on a non-indexed column), `Using join buffer (Block Nested Loop)` (no usable index on the join). `Using index` is good — it means a covering index.

Also check `rows` (estimated rows examined) and `filtered` (percentage surviving the filter). `rows` in the millions with `filtered: 1.00` means you're reading everything to keep almost nothing.

## Finding slow queries

```sql
-- performance_schema, by total time
SELECT digest_text, count_star,
       round(sum_timer_wait/1e12, 2) AS total_s,
       round(avg_timer_wait/1e9, 2)  AS avg_ms,
       sum_rows_examined, sum_rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;

-- sys schema shortcuts
SELECT * FROM sys.statements_with_full_table_scans LIMIT 20;
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.statements_with_temp_tables LIMIT 20;
```
Slow query log: `slow_query_log=ON`, `long_query_time=0.5`, `log_queries_not_using_indexes` (noisy — enable briefly). Analyze with `pt-query-digest`.

A `sum_rows_examined / sum_rows_sent` ratio far above 1 is the clearest single indicator of a missing index.

## Configuration that actually matters

- `innodb_buffer_pool_size` — 50–75% of RAM on a dedicated server. This is the single most impactful setting.
- `innodb_flush_log_at_trx_commit = 1` for durability (0/2 trade data loss for throughput — only with explicit sign-off).
- `sync_binlog = 1` when replication correctness matters.
- `innodb_flush_method = O_DIRECT` on Linux.
- `max_connections` — MySQL threads are cheaper than PG processes, but a pool (ProxySQL, or the app's pool) still beats thousands of connections.
- `transaction_isolation`: default `REPEATABLE READ`. `READ COMMITTED` reduces gap locking and deadlocks and is what most other databases do; consider it for high-contention workloads, but understand it changes read semantics within a transaction.

## Deadlocks and locking

```sql
SHOW ENGINE INNODB STATUS\G   -- LATEST DETECTED DEADLOCK section
SELECT * FROM performance_schema.data_locks;
SELECT * FROM sys.innodb_lock_waits;
```

- InnoDB takes **gap locks** under `REPEATABLE READ`; a range `UPDATE` or a `SELECT ... FOR UPDATE` locks gaps between index entries, so concurrent inserts block on rows that don't exist yet.
- Locks are taken on **index records**. An `UPDATE` with no usable index locks every row it scans — effectively the whole table. Indexing the `WHERE` clause of an `UPDATE` is a locking fix, not just a speed fix.
- Deadlocks are normal under concurrency: acquire locks in a consistent order everywhere, keep transactions short, and make the application retry on error 1213.

## Replication

- Prefer GTID-based replication and `binlog_format = ROW`.
- Monitor `SHOW REPLICA STATUS` → `Seconds_Behind_Source`, plus `performance_schema.replication_applier_status_by_worker`. `Seconds_Behind_Source` reads 0 when the replica is stalled, so alert on the applier status too.
- Enable parallel replication (`replica_parallel_workers`, `replica_parallel_type = LOGICAL_CLOCK`) before assuming a single-threaded applier is a hard limit.
- Big backfills and schema changes are the main lag sources — throttle on lag, always.
- Semi-sync replication if you cannot tolerate losing the last transactions on failover.
