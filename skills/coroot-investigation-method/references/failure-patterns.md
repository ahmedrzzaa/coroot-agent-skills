# Common failure patterns and the Coroot signals that confirm them

Each row is a wolf you may find at the end of the fencing. Metric names are Coroot node-agent and cluster-agent metrics (see `coroot-telemetry-analytics` and `coroot-database-analysis` for details); the mapping of pattern to signal combination is diagnostic reasoning (inferred), so confirm with at least two independent signals.

| Pattern | Typical symptom | Confirming signals in Coroot | Go deeper |
|---|---|---|---|
| **Connection-pool exhaustion / leak** | Tail latency or timeouts in one service; its queries are fast | Inbound latency high, outbound DB query latency normal; `container_net_tcp_active_connections` to the database flat at pool size; `pg_connections` stuck `idle` or `idle in transaction`; OTel pool-acquire span long | `coroot-database-analysis` (client vs server) |
| **Slow database query or lock** | Latency in every caller of one database | Client and server latency both high; `pg_top_query_time_per_second` / `mysql_top_query_time_per_second` led by one query; `*_lock_awaiting_queries` by `blocking_query` | `coroot-database-analysis` |
| **CPU throttling** | Latency rises with load; CPU looks "fine" on average | `rate(container_resources_cpu_throttled_seconds_total[5m])` up; usage near `container_resources_cpu_limit_cores` | `coroot-resource-analysis` |
| **Noisy neighbour / node saturation** | Several unrelated services slow on one node | Node CPU above 80%; `container_resources_cpu_delay_seconds_total` rising across containers on that node | `coroot-resource-analysis` |
| **Memory leak / OOM** | Periodic restarts, errors after restarts | `container_resources_memory_rss_bytes` climbing; `container_oom_kills_total`, `container_restarts_total` increasing | `coroot-resource-analysis` |
| **Bad deployment** | Errors or latency starting at a rollout | Change aligned with a new ReplicaSet or GitOps revision; new error log patterns only after it | `coroot-deployment-analysis` |
| **Network problem** | Timeouts across a path; failed connects | RTT (`container_net_latency_seconds`) above baseline; `container_net_tcp_retransmits_total`, `container_net_tcp_failed_connects_total` up | `coroot-resource-analysis` (network) |
| **DNS failure** | Sudden errors connecting to names | DNS inspection: server errors or NXDOMAIN on previously valid names; `container_dns_requests_total` by `status` | `coroot-observability` (inspections) |
| **Retry storm / cascading failure** | Dependency overloaded while user traffic is flat | Outbound request rate to a dependency rises without inbound rise; repeated identical client spans; spans ending at round timeout values | `coroot-trace-analysis` |
| **Traffic spike** | Everything slower at once | Inbound request rate well above the same time yesterday | `coroot-telemetry-analytics` |
| **Cache miss storm** | Database load jumps after a cache restart or eviction | Redis/Memcached hit ratio down, evictions up; database query rate up at the same time | `coroot-database-analysis` |

Reading rules:

- A low p50 with a high p99 points at something that affects a subset: one dependency, one instance, a queue, or a pool.
- Fast queries plus slow requests means the wait happens before the query (pool, lock in the application, thread pool).
- High latency with low CPU means waiting, not computing — look for locks, pools, I/O, or dependencies.
- An error that appears only at the edge (gateway 504) is a timeout from further in; follow the latency, not the error.
