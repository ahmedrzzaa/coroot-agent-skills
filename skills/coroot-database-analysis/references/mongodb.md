# MongoDB in Coroot

Sources: https://docs.coroot.com/databases/mongodb, https://docs.coroot.com/inspections/mongodb, https://docs.coroot.com/metrics/cluster-agent, and github.com/coroot/coroot `model/check.go`, `auditor/mongodb.go`, coroot-cluster-agent `metrics/mongo/` (read 2026-10-27).

## Prerequisites for server-side data

- A user with `clusterMonitor` and `read` on `local`; an optional `corootCatalog` role enables index tracking.
- The agent must connect **to each mongod directly, not through mongos**.
- Top queries need the profiler: `operationProfiling.mode: slowOp`, or `all` with `rateLimit` on Percona. The agent reads `system.profile` (the inspection page mentions `$queryStats`, which is out of date).
- Configured by `coroot.com/mongodb-scrape` annotations (plus TLS and auth-source parameters), the UI, or `clusterAgent.databases`.

## Inspection checks

| Check | Default |
|---|---|
| Availability (includes "no primary") | any |
| Replication lag | 30s |
| Latency (`serverStatus.opLatencies`) | 0.1s |
| Oplog window | under 1h while the oplog is at least 95% full |
| Connections | 90% of `maxIncomingConnections` |
| Saturation (queued operations, last 3 points) | above 0 |
| Fragmentation | above 50% and at least 1 GiB |
| Backups | 24h |

The lag check distinguishes **blocked** apply (fsyncLock, prepared transaction, lock held) from **slow** apply.

## Metrics

| Metric | Type | Labels |
|---|---|---|
| `mongo_op_latency_seconds_total`, `mongo_op_latency_ops_total` | counter | `type` |
| `mongo_top_query_*_per_second` | gauge, already per-second | `db`, `collection`, `query` |
| `mongo_connections_current`, `mongo_connections_active`, `mongo_connections_max` | gauge | — |
| `mongo_connections_rejected_total` | counter | — |
| `mongo_connections_by_app` | gauge | `app` |
| `mongo_queued_operations`, `mongo_wt_tickets_available` | gauge | `type` |
| `mongo_wt_cache_used_bytes`, `mongo_wt_cache_dirty_bytes`, `mongo_wt_cache_max_bytes` | gauge | — |
| `mongo_wt_pages_evicted_by_app_threads_total`, `mongo_wt_cache_bytes_read_into_total` | counter | — |
| `mongo_rs_status` | gauge | `rs`, `role` |
| `mongo_rs_last_applied_timestamp_ms`, `mongo_oplog_window_seconds`, `mongo_repl_buffer_operations`, `mongo_fsync_locked` | gauge | — |
| `mongo_long_running_operations` | gauge | `db`, `collection`, `query`, `plan` |
| `mongo_operations_waiting_for_lock` | gauge | `db` |

All carry `address="<ip>:<port>"`. Coroot's queries also reference `mongo_rs_member_status` and `mongo_rs_member_replication_lag_seconds`, which are not in the docs; confirm with `list_metric_names`.

## PromQL

Average operation latency by type (formula from the docs):

```promql
sum by (address, type) (rate(mongo_op_latency_seconds_total[5m]))
/ sum by (address, type) (rate(mongo_op_latency_ops_total[5m]))
```

Connection usage and the heaviest clients:

```promql
mongo_connections_current / mongo_connections_max
topk(5, mongo_connections_by_app)
```

Documents scanned per document returned (index efficiency):

```promql
sum by (address) (rate(mongo_scanned_documents_total[5m]))
/ sum by (address) (rate(mongo_documents_returned_total[5m]))
```

Replication lag (Coroot's query):

```promql
timestamp(mongo_rs_last_applied_timestamp_ms) - mongo_rs_last_applied_timestamp_ms / 1000
```

## Interpretation notes

- Application threads evicting cache pages (`mongo_wt_pages_evicted_by_app_threads_total` rising) means the WiredTiger cache is under pressure and requests pay for eviction.
- Few available WiredTiger tickets plus queued operations indicates saturation; find the long-running operation and its `plan` (a `COLLSCAN` plan suggests a missing index; inferred).
