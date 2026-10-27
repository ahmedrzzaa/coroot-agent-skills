# Redis and Memcached in Coroot

Sources: https://docs.coroot.com/databases/redis, https://docs.coroot.com/databases/memcached, https://docs.coroot.com/inspections/redis, https://docs.coroot.com/inspections/memcached, and github.com/coroot/coroot `auditor/redis.go`, `auditor/auditor.go`, `constructor/memcached.go`, `constructor/queries.go` (read 2026-10-27).

## Redis

- Collected by an embedded redis_exporter (metrics only). Configured by `coroot.com/redis-scrape` annotations (port, password, secret; `tls` = `true` or `skip-verify`), the UI, or `clusterAgent.databases`. ElastiCache, Memorystore, and OCI Cache disable the `CONFIG` command.
- **Checks:** availability, and latency above **5ms**. Latency is server-side: total command time ÷ total calls, last value.
- **No memory, eviction, or replication check exists.** Query those metrics directly when relevant.
- Metrics Coroot uses: `redis_up`, `redis_instance_info{role}`, `redis_commands_total{cmd}` (counter), `redis_commands_duration_seconds_total{cmd}` (counter), `redis_db_keys{db}`, `redis_db_keys_expiring{db}`.
- Other redis_exporter metrics (present in the exporter, not used by Coroot; confirm with `list_metric_names`): `redis_memory_used_bytes`, `redis_memory_max_bytes`, `redis_evicted_keys_total`, `redis_keyspace_hits_total`, `redis_keyspace_misses_total`, `redis_connected_clients`, `redis_rejected_connections_total`, `redis_blocked_clients`, `redis_master_link_up`.

```promql
# Server-side average latency
sum by (address) (rate(redis_commands_duration_seconds_total[5m]))
/ sum by (address) (rate(redis_commands_total[5m]))

# Most expensive commands
topk(5, sum by (cmd) (rate(redis_commands_duration_seconds_total[5m])))

# Hit ratio and evictions (exporter metrics)
rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
rate(redis_evicted_keys_total[5m])
```

Notes: slow commands such as `KEYS`, large `HGETALL`, or `SMEMBERS` stand out in the per-command time; memory at `maxmemory` with rising evictions lowers the hit ratio and moves load to the backing database (inferred).

## Memcached

- Embedded memcached_exporter; configured by `coroot.com/memcached-scrape` annotations (no credentials).
- **Check: availability only.** The report shows commands (top 5), hit rate %, and evictions per second.
- **A failing Memcached report does not raise the application's overall status.** Read it explicitly.
- Coroot's hit rate sums hits and misses across every command, not only `get`.
- Metrics: `memcached_up`, `memcached_limit_bytes` (gauge), `memcached_items_evicted_total` (counter), `memcached_commands_total{command,status}` (counter). From the exporter: `memcached_current_bytes`, `memcached_current_connections`, `memcached_max_connections`, `memcached_connections_rejected_total`.

```promql
# get hit ratio
sum by (address) (rate(memcached_commands_total{command="get",status="hit"}[5m]))
/ sum by (address) (rate(memcached_commands_total{command="get",status=~"hit|miss"}[5m]))

# Evictions and fill level
rate(memcached_items_evicted_total[5m])
memcached_current_bytes / memcached_limit_bytes
```
