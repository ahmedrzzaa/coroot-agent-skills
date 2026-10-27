# Coroot node-agent metrics

Source: https://docs.coroot.com/metrics/node-agent (verified 2026-10-27). Always confirm availability and labels in the connected environment with `list_metric_names` (official MCP) or a raw query before relying on a metric.

## Container CPU

| Metric | Meaning |
|---|---|
| `container_resources_cpu_usage_seconds_total` | Total CPU time consumed (counter; use `rate()` for cores). |
| `container_resources_cpu_throttled_seconds_total` | Total time throttled by the CPU quota (counter). |
| `container_resources_cpu_delay_seconds_total` | Time spent waiting for a CPU (counter). |
| `container_resources_cpu_limit_cores` | Configured CPU limit in cores (gauge). |

## Container memory and lifecycle

| Metric | Meaning |
|---|---|
| `container_resources_memory_rss_bytes` | Resident memory, excluding page cache (gauge). |
| `container_resources_memory_cache_bytes` | Page cache (gauge). |
| `container_oom_kills_total` | OOM kills (counter; use `increase()`). |
| `container_restarts_total` | Container restarts (counter; use `increase()`). |

## Network

| Metric | Meaning |
|---|---|
| `container_net_tcp_failed_connects_total` | Failed TCP connection attempts (counter). |
| `container_net_tcp_retransmits_total` | Retransmitted TCP segments (counter). |
| `container_net_latency_seconds` | Round-trip time to remote IPs (gauge). |

## Disk

| Metric | Meaning |
|---|---|
| `container_resources_disk_read_time_seconds_total` | Time spent on reads (counter). |
| `container_resources_disk_write_time_seconds_total` | Time spent on writes (counter). |

## Node

| Metric | Meaning |
|---|---|
| `node_resources_cpu_usage_seconds_total` | Node CPU time by mode (counter). |
| `node_resources_memory_total_bytes` | Total physical memory (gauge). |
| `node_resources_memory_free_bytes` | Unassigned memory (gauge). |

## Non-Coroot metrics

cAdvisor-style metrics such as `container_memory_rss`, `container_cpu_cfs_periods_total`, and `container_cpu_cfs_throttled_periods_total` exist only if that exporter is scraped into the same Prometheus. Do not assume they are present.
