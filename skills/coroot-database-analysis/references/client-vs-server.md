# Is it the database, the network, or the client?

Sources: https://docs.coroot.com/metrics/node-agent, github.com/coroot/coroot `model/connection.go`, `model/check.go`, `auditor/network.go`, `constructor/queries.go`, `api/mcp.go` (read 2026-10-27). The decision rules are reasoning built from these signals (inferred).

## Signals

| Side | Signal | Metric |
|---|---|---|
| Client | Query rate and failures | `container_{postgres,mysql,mongo,redis,memcached}_queries_total{destination,actual_destination,status}` — failures are `status="failed"` |
| Client | Latency histogram | `container_<db>_queries_duration_seconds_total_bucket` / `_sum` (with `le`) |
| Server (eBPF, on the database container) | Inbound queries | `container_<db>_inbound_queries_*` (no destination labels) |
| Server (cluster agent) | Server-measured latency | `pg_latency_seconds`, Mongo op latency, Redis command time, `*_top_query_time_per_second` |
| Network | Round-trip time | `container_net_latency_seconds{destination_ip}` (ICMP) |
| Network | TCP failures and retransmits | `container_net_tcp_failed_connects_total{destination}`, `container_net_tcp_retransmits_total` |
| Network | TCP connect time | `rate(container_net_tcp_connection_time_seconds_total[5m]) / rate(container_net_tcp_successful_connects_total[5m])` |
| Client | Open connections | `container_net_tcp_active_connections` |

`get_application_status` reports, per upstream dependency: `connectivity`, `rtt_seconds`, `rps`, `errors_per_sec`, and latency avg/p50/p95/p99. Network checks: RTT above 10ms in-cluster, 100ms cross-cluster, 200ms external; connectivity lost when RTT samples stop; any failed TCP connect. If ICMP to a managed database is blocked, RTT may simply be absent rather than flagged.

## Decision rules

| Client latency | Server latency | Network | Likely location |
|---|---|---|---|
| High | High | Normal | **Database** (queries, locks, I/O, saturation) |
| High | Normal | RTT or retransmits up | **Network** path |
| High | Normal | Normal | **Client**: connection-pool saturation (active connections flat at pool size), client CPU throttling or GC pauses, slow TCP connects |
| Failures or failed connects | — | — | Database down, connection limit reached, or authentication/TLS failure; check MySQL aborted connects and MongoDB rejected connections |

MySQL's built-in latency check is itself client-side; use `sum(mysql_top_query_time_per_second)` as the server-side comparison.

## PromQL

Client-side average latency per database instance:

```promql
sum by (actual_destination) (rate(container_postgres_queries_duration_seconds_total_sum[5m]))
/ sum by (actual_destination) (rate(container_postgres_queries_total[5m]))
```

Client-side p99 per database instance:

```promql
histogram_quantile(0.99, sum by (le, actual_destination) (rate(container_mysql_queries_duration_seconds_total_bucket[5m])))
```

Client-side failure ratio:

```promql
sum by (actual_destination) (rate(container_redis_queries_total{status="failed"}[5m]))
/ sum by (actual_destination) (rate(container_redis_queries_total[5m]))
```

Split by caller with `app_id` or `container_id` to see whether one client or all clients are affected. If only one client is slow while others are fine, the problem is in that client or its network path, not in the database.
