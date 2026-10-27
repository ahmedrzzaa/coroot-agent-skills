# Postgres in Coroot

Sources: https://docs.coroot.com/databases/postgres, https://docs.coroot.com/inspections/postgres, https://docs.coroot.com/metrics/cluster-agent, and github.com/coroot/coroot `model/check.go`, `auditor/postgres.go`, `constructor/queries.go` (read 2026-10-27). "(inferred)" marks reasoning not stated in the sources.

## Prerequisites for server-side data

- Postgres 10+; a role with `pg_monitor`.
- `pg_stat_statements` in `shared_preload_libraries` and `CREATE EXTENSION` in the `postgres` database (the agent connects there).
- `track_io_timing=on`, otherwise per-query I/O time is 0.
- `CONNECT` on user databases for schema and size tracking.
- Configured by pod annotations `coroot.com/postgres-scrape: "true"` (plus port, credentials, sslmode, TLS secret), the UI (Postgres tab → Configure), or `clusterAgent.databases`. The docs include CloudNativePG, Percona, and Zalando examples.

## Inspection checks

| Check | Default |
|---|---|
| Availability (`pg_up`) | any unavailability |
| Latency (`pg_latency_seconds{summary="avg"}`) | 0.1s |
| Replication lag | 30s (byte lag converted to time) |
| Connections | 90% of `max_connections` (includes reserved superuser connections) |
| Checkpoints | 3 × `checkpoint_timeout` |
| WAL archiving | last attempt failed |
| Transaction ID wraparound | 50% |
| Bloat | 50% of database size |
| Autovacuum / stale statistics | 2 × trigger |
| Backups (CloudNativePG, Percona) | 24h |

The report charts latency (avg, p50, p95, p99), queries per second, queries by total and I/O time, active and idle-in-transaction connections by query, locked and blocking queries, replication lag split into shipping and apply, WAL size, replication slots, and top tables.

## Metrics

| Metric | Type | Labels |
|---|---|---|
| `pg_up`, `pg_probe_seconds` | gauge | — |
| `pg_latency_seconds` | gauge | `summary` (avg, p50, p95, p99) |
| `pg_top_query_calls_per_second`, `pg_top_query_time_per_second`, `pg_top_query_io_time_per_second` | gauge, already per-second, top 20 | `db`, `user`, `query` |
| `pg_connections` | gauge | `db`, `user`, `state`, `wait_event_type`, `query` |
| `pg_setting` | gauge | `name`, `unit` |
| `pg_lock_awaiting_queries` | gauge | `db`, `user`, `blocking_query` |
| `pg_transaction_seconds` | gauge | `db`, `user`, `query` |
| `pg_wal_current_lsn`, `pg_wal_receive_lsn`, `pg_wal_reply_lsn` | counter | — (note the spelling "reply") |
| `pg_wal_replay_paused`, `pg_wal_receiver_status` | gauge | `sender_host`, `sender_port` |
| `pg_replication_slot_retained_wal_bytes` | gauge | `slot`, `active`, `wal_status` |
| `pg_xid_age`, `pg_oldest_xmin_age` | gauge | `db` / holder |

All carry `address="<ip>:<port>"`.

## PromQL

Top 5 queries by time spent:

```promql
topk(5, pg_top_query_time_per_second{address="VERIFIED_ADDRESS"})
```

Average time per call of the top queries (inferred):

```promql
pg_top_query_time_per_second / pg_top_query_calls_per_second
```

Connection usage ratio (inferred):

```promql
sum by (address) (pg_connections) / on (address) max by (address) (pg_setting{name="max_connections"})
```

Who is blocking whom:

```promql
sum by (blocking_query) (pg_lock_awaiting_queries)
```

WAL throughput (Coroot's query) and replica apply lag in bytes (inferred):

```promql
rate(pg_wal_current_lsn[5m])
pg_wal_receive_lsn - pg_wal_reply_lsn
```

## Interpretation notes

- Idle-in-transaction connections hold locks and block vacuum; look for them in `pg_connections{state="idle in transaction"}` together with `pg_transaction_seconds`.
- High `pg_oldest_xmin_age` with a named holder (long transaction, replication slot, prepared transaction) explains bloat and autovacuum warnings.
- Postgres has no connection-error check like MySQL's; use client `status="failed"` and TCP failed connects.
