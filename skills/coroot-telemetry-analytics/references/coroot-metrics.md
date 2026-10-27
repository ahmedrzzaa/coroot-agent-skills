# Coroot metrics for analysis

Sources: https://docs.coroot.com/metrics/node-agent, github.com/coroot/coroot-node-agent (`containers/metrics.go`, `containers/l7.go`, `ebpftracer/l7/l7.go`), and github.com/coroot/coroot (`constructor/queries.go`, `model/connection.go`, `model/sli.go`), read at 2026-10-21. Confirm presence and labels in the connected environment before use.

## Request metrics (eBPF, no instrumentation needed)

**Inbound (server side)** — what the application served:

- `container_<proto>_inbound_requests_total` or `container_<proto>_inbound_queries_total`, for example `container_http_inbound_requests_total`, `container_llm_inbound_requests_total`.
- Matching latency histogram `container_<proto>_inbound_requests_duration_seconds_total`.

**Outbound (client side)** — what the application called:

- Requests: `container_http_requests_total`, `container_dubbo_requests_total`, `container_dns_requests_total`, `container_kafka_requests_total`, `container_zookeeper_requests_total`, `container_foundationdb_requests_total`.
- Queries: `container_postgres_queries_total`, `container_mysql_queries_total`, `container_redis_queries_total`, `container_memcached_queries_total`, `container_mongo_queries_total`, `container_cassandra_queries_total`, `container_clickhouse_queries_total`.
- Messages (count only, no latency): `container_rabbitmq_messages_total`, `container_nats_messages_total` (label `method`).
- Each request or query counter has a matching `*_duration_seconds_total` latency histogram.

### The latency metric is a histogram

The docs call `*_duration_seconds_total` a counter, but the agent source builds a Prometheus histogram. Because the name already ends in `_total`, the series are:

- `..._duration_seconds_total_bucket` (with `le`)
- `..._duration_seconds_total_sum`
- `..._duration_seconds_total_count` (standard histogram series; Coroot's own queries use `_bucket` and `_sum`)

Buckets (seconds): `0.005 0.01 0.025 0.05 0.1 0.25 0.5 1 2.5 5 10`. LLM metrics use buckets from 0.1s to 600s.

The histogram has no `status` label, so its percentiles mix successful and failed requests. A fast-failing dependency can make latency look better.

## Labels

| Label | Meaning |
|---|---|
| `container_id` | Container path, for example `/k8s/<namespace>/<pod>/<container>` |
| `app_id` | `/k8s/<namespace>/<workload>` for Deployment, DaemonSet, and StatefulSet pods; empty for others and on older agents. **Not** the four-part MCP application ID. On outbound metrics it is the caller. |
| `machine_id`, `system_uuid` | Host identity |
| `destination` | Address the client connected to (for example a Service IP:port) |
| `actual_destination` | Real pod IP:port after NAT; use it to split by server instance |
| `status` | Request outcome (below) |
| `type` | HTTP only; for example `llm` |

## `status` values and what Coroot counts as failed

- HTTP: status class `2xx`, `3xx`, `4xx`, `5xx`, or `unknown`; gRPC: `grpc:<CODE>`.
- Databases: `ok`, `failed`, `unknown`.
- DNS: values such as `nxdomain` and `servfail`.

Coroot treats a request as failed when the status is `failed`, starts with `5`, or starts with `grpc:` and is not `grpc:OK`. **4xx responses are not failures** in Coroot's SLIs. Matching selector:

```promql
{status=~"5.*|failed|grpc:.*", status!="grpc:OK"}
```

## Resource and network metrics

| Area | Metrics |
|---|---|
| CPU | `container_resources_cpu_usage_seconds_total`, `container_resources_cpu_delay_seconds_total`, `container_resources_cpu_throttled_seconds_total`, `container_resources_cpu_limit_cores` |
| Memory | `container_resources_memory_rss_bytes`, `container_resources_memory_cache_bytes`, `container_resources_memory_limit_bytes` |
| Lifecycle | `container_oom_kills_total`, `container_restarts_total` |
| TCP | `container_net_tcp_successful_connects_total`, `container_net_tcp_failed_connects_total`, `container_net_tcp_retransmits_total`, `container_net_tcp_active_connections` |
| Network latency | `container_net_latency_seconds` (label `destination_ip`) |
| Node | `node_resources_cpu_usage_seconds_total` (label `mode`), `node_resources_memory_total_bytes`, `node_resources_memory_free_bytes` |
| Logs | `container_log_messages_total` (message counts by level and pattern) |

The node agent scrapes every 15 seconds by default.
