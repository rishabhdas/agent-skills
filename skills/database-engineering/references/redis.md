# Redis

Redis is single-threaded for command execution. **Every slow command blocks every other client.** That one fact drives most of the rules here.

## Choosing a data structure

| Need | Structure | Notes |
|---|---|---|
| Cached blob, counter, flag | String | `SET k v EX 300`, `INCR`, `SETNX` |
| Object with fields you update independently | Hash | Small hashes are encoded compactly (`hash-max-listpack-entries`); much cheaper than N strings |
| Queue / stream of work | **Stream** (`XADD`/`XREADGROUP`) | Consumer groups, acknowledgement, replay. Prefer over List for real work queues |
| Simple FIFO/LIFO, small | List | `LPUSH`/`BRPOP`. No ack semantics — a crashed consumer loses the item |
| Uniqueness, tags, relations | Set | `SADD`/`SISMEMBER`. `SINTERCARD` for bounded intersections |
| Leaderboard, priority queue, time index | Sorted Set | `ZADD`/`ZRANGEBYSCORE`. Also the right tool for rate limiting and delayed jobs |
| Approximate unique counts at scale | HyperLogLog | ~12KB for billions of items, 0.81% error. `PFADD`/`PFCOUNT` |
| Membership test, huge set, false positives OK | Bloom filter (RedisBloom) | |
| Geospatial | Geo (sorted set backed) | `GEOADD`/`GEOSEARCH` |
| Time series metrics | RedisTimeSeries | Downsampling and retention built in |

Reach for the specialized structure before building it out of strings — you'll use a fraction of the memory and a single round trip.

## Key design

- Namespace with colons, most general first: `app:users:{user_id}:sessions`, `app:cache:orders:{id}:v2`.
- **Put a version in cache keys.** Changing the serialized shape then means writing to `...:v3` and letting v2 expire — no flush, no stale-shape deserialization errors.
- Keys are bytes; keep them short but readable. Millions of 80-byte keys is real memory.
- No user input directly in a key without sanitizing — collisions and injection into key patterns are both real.
- **Every key gets a TTL unless you can explain why not.** Keys without TTLs are how Redis instances fill up at 3am. Audit: `redis-cli --scan --pattern 'x:*' | while read k; do redis-cli ttl "$k"; done | sort | uniq -c`
- In Cluster mode, use hash tags to co-locate keys a multi-key command must touch: `{user:1000}:profile` and `{user:1000}:sessions` land on the same slot.

## Commands that will hurt you

| Never in production | Use instead |
|---|---|
| `KEYS pattern` | `SCAN cursor MATCH pattern COUNT 100` (cursor-based, non-blocking) |
| `FLUSHALL` / `FLUSHDB` | Namespaced keys + TTL; delete by scan if truly needed |
| `SMEMBERS` / `HGETALL` / `LRANGE 0 -1` on a big key | `SSCAN`/`HSCAN`, or bound the range |
| `DEL bigkey` | `UNLINK bigkey` (frees memory in a background thread) |
| `SAVE` | `BGSAVE` |
| Unbounded Lua scripts | Keep scripts short — they block the server for their entire duration |
| `MONITOR` under load | It degrades throughput substantially; sample briefly if at all |

Find the damage:
```
redis-cli --bigkeys            # largest key per type
redis-cli --hotkeys            # requires maxmemory-policy allkeys-lfu
redis-cli --memkeys
redis-cli SLOWLOG GET 25       # slowlog-log-slower-than default 10000µs
redis-cli INFO commandstats
redis-cli --latency-history
```

## Memory and eviction

```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

| Policy | Use when |
|---|---|
| `noeviction` (default) | Redis is a datastore, not a cache — writes error when full. Correct for queues and sessions you cannot lose |
| `allkeys-lru` / `allkeys-lfu` | Pure cache. LFU is better when access frequency is skewed and stable |
| `volatile-lru` / `volatile-ttl` | Mixed use: only keys with a TTL are evictable. Requires discipline about setting TTLs |

**Never run a cache and a durable queue in the same instance with an `allkeys-*` policy** — the eviction will silently eat your queue. Separate instances (or at minimum separate policies and separate deployments).

Watch `used_memory_rss / used_memory` (fragmentation ratio) in `INFO memory`. Above ~1.5 with a large dataset, consider `activedefrag yes`. Also watch `evicted_keys` and `keyspace_misses` — a rising eviction rate means your working set no longer fits.

## Persistence

- **RDB**: point-in-time fork+snapshot. Compact, fast restart, loses everything since the last snapshot. The fork can double memory momentarily on write-heavy instances — size the host accordingly and set `vm.overcommit_memory = 1`.
- **AOF**: append-only log. `appendfsync everysec` is the standard trade-off (≤1s loss). `always` is durable and slow.
- **Both** for anything you'd be unhappy to lose. `aof-use-rdb-preamble yes` gives fast loading plus a recent tail.
- Pure cache → persistence off is a legitimate choice; just be explicit that a restart means a cold cache, and make sure your database can survive the resulting stampede.

## Caching patterns

**Cache-aside** (default):
```
v = redis.get(k)
if v is None:
    v = db.query(...)
    redis.set(k, serialize(v), ex=ttl_with_jitter)
return v
```

- **Add jitter to every TTL** (`ttl * random.uniform(0.8, 1.2)`). Identical TTLs set during a deploy expire simultaneously and stampede the database.
- **Stampede protection** for hot keys: a short `SET lock:k token NX EX 10` so only one process recomputes while others serve stale or wait. Or probabilistic early expiry (XFetch).
- **Invalidation**: delete on write, don't update in place — updating races with concurrent readers. Accept that cache-then-DB write ordering has a window; version keys if that window matters.
- **Negative caching**: cache "not found" with a short TTL, or a bloom filter, to stop repeated misses hammering the DB.
- Never treat the cache as the source of truth. Every read path must work, more slowly, with Redis down. Test that path.

## Distributed locks

`SET lock:resource <random-token> NX PX 30000`, and release only via a Lua script that checks the token matches before deleting — otherwise you delete someone else's lock after your own expired.

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end
```

Redlock across independent masters is contentious and does not give you a correctness guarantee under GC pauses or clock skew. **If correctness depends on the lock, use a fencing token checked by the resource, or a database transaction.** Redis locks are appropriate for reducing duplicate work, not for guarding money.

## Atomicity

- `MULTI`/`EXEC` queues commands and runs them without interleaving, but **there is no rollback** — a command that fails at runtime doesn't undo the others. Combine with `WATCH` for optimistic concurrency.
- **Lua scripts** (`EVALSHA`) are the practical way to do read-then-write atomically. Keep them under a millisecond; they block the server.
- Redis Functions (7.0+) are the persistent, named successor to `EVAL` scripts.
- Pipelining is not atomicity — it's a round-trip optimization. Use it aggressively anyway; batching 100 commands into one round trip is often a 10× throughput win.

## Cluster and high availability

- **Sentinel**: HA for a single primary + replicas. Automatic failover, no sharding.
- **Cluster**: 16384 hash slots across primaries. Multi-key commands only work within a slot (use hash tags), no cross-slot transactions, and clients must be cluster-aware.
- Replicas are asynchronous — a failover can lose recent writes. `WAIT numreplicas timeout` gives a weaker-than-it-looks acknowledgement, not a durability guarantee.
- Reading from replicas gives stale data; `READONLY` on a cluster connection is opt-in for a reason.
- Scale up before you shard. A single Redis node handles 100k+ ops/sec; most "we need a cluster" situations are actually a big-key or hot-key problem found by `--bigkeys`/`--hotkeys`.

## Operational checklist

- `maxmemory` and `maxmemory-policy` explicitly set — not defaults
- TTLs on all cache keys, with jitter
- `requirepass` / ACLs enabled, no public binding, TLS if crossing a network boundary
- Dangerous commands renamed or ACL-denied (`KEYS`, `FLUSHALL`, `CONFIG`, `DEBUG`)
- `SLOWLOG` monitored; alert on entries
- Client-side timeouts and retry-with-backoff, and a code path that survives Redis being unavailable
- Alerts on `evicted_keys`, `blocked_clients`, `connected_clients` near `maxclients`, memory fragmentation, replication link status
