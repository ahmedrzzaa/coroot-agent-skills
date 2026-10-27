# MySQL, MariaDB, and Galera in Coroot

Sources: https://docs.coroot.com/databases/mysql, https://docs.coroot.com/inspections/mysql, https://docs.coroot.com/metrics/cluster-agent, and github.com/coroot/coroot `model/check.go`, `auditor/mysql.go` (read 2026-10-27).

## Prerequisites for server-side data

- `GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.*` (a minimal-grants variant loses schema and size tracking).
- `performance_schema=ON`. **On RDS it is off unless Performance Insights is enabled**; set `performance_schema=1` and reboot.
- Configured by `coroot.com/mysql-scrape` annotations, the UI, or `clusterAgent.databases`. MariaDB skips lock-wait sampling.

## Inspection checks

| Check | Default |
|---|---|
| Availability | any unavailability |
| Replication status (IO or SQL thread stopped; shows `last_error`) | any |
| Replication lag (`Seconds_Behind_Source`, asynchronous replication) | 30s |
| Connections | 90% of max |
| Connection errors (aborted connects) | above 1/s |
| Galera flow control | 10% |
| Group Replication queue | 1000 transactions |
| Latency | 0.1s |
| Backups (Percona XtraDB Cluster) | 24h |

**The latency check is client-observed** (eBPF, average of the last 5 points). When it fires, Coroot ranks likely causes:

- the top query by time;
- row locks;
- a transaction open longer than 60s;
- on-disk temporary tables above 1/s;
- buffer pool `wait_free` above 0 or hit rate below 99%;
- redo log waits;
- history list longer than 1M;
- table lock waits above 1/s;
- Galera flow control above 10%;
- `threads_running` per CPU core above 1.

## Metrics

| Metric | Type | Labels |
|---|---|---|
| `mysql_top_query_{calls,time,lock_time,rows_examined,rows_sent}_per_second` | gauge, already per-second, top 20 | `schema`, `query` |
| `mysql_locked_queries` | gauge | `schema`, `query` |
| `mysql_lock_awaiting_queries` | gauge | `schema`, `blocking_query` |
| `mysql_replication_lag_seconds` | gauge | — |
| `mysql_replication_io_status`, `mysql_replication_sql_status` | gauge | `state`, `last_error` |
| `mysql_connections_current`, `mysql_connections_max` | gauge | — |
| `mysql_connections_aborted_total`, `mysql_connection_errors_max_connections_total` | counter | — |
| `mysql_innodb_buffer_pool_reads_total`, `mysql_innodb_buffer_pool_read_requests_total` | counter | — |
| `mysql_innodb_deadlocks_total` | counter | — |
| `mysql_innodb_history_list_length` | gauge | — |
| `mysql_innodb_transaction_seconds` | gauge | `query` |

All carry `address="<ip>:<port>"`.

## PromQL

```promql
topk(5, mysql_top_query_time_per_second)
```

Rows examined per row sent for the top queries (a full-scan signal; inferred):

```promql
mysql_top_query_rows_examined_per_second / mysql_top_query_rows_sent_per_second
```

Buffer pool hit rate (formula from the docs):

```promql
1 - sum by (address) (rate(mysql_innodb_buffer_pool_reads_total[5m]))
  / sum by (address) (rate(mysql_innodb_buffer_pool_read_requests_total[5m]))
```

Connection usage and refusals:

```promql
mysql_connections_current / mysql_connections_max
rate(mysql_connection_errors_max_connections_total[5m])
```

Blocking queries:

```promql
sum by (blocking_query) (mysql_lock_awaiting_queries)
```

## Interpretation notes

- Because the latency check is client-side, compare it with server-side time (`sum(mysql_top_query_time_per_second)`) before blaming the server (inferred).
- A growing history list together with a long `mysql_innodb_transaction_seconds` points at an open transaction preventing purge.
