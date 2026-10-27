# Memory allocation and pressure

Investigate memory separately from CPU. Low CPU usage does not mean a workload has adequate resources.

## Distinguish memory measurements

- **Memory request:** Kubernetes scheduling reservation.
- **Memory limit:** Configured maximum memory for the container, when enforced.
- **RSS:** Resident memory currently held in physical memory.
- **Page cache:** File-backed memory; Coroot reports it separately from RSS.
- **Working set:** A metric commonly used to approximate actively used container memory; exact definitions depend on the collector.
- **Heap usage:** Runtime-managed heap memory, which is not equivalent to total process RSS.
- **OOM kill:** A process termination associated with memory exhaustion.
- **Node memory pressure:** Host-level contention that can affect multiple workloads.

Do not compare these measurements as if they were identical.

## Workflow

1. Retrieve available container and node memory metrics.
2. Check memory requests and limits where accessible.
3. Compare current, average, and peak usage over the incident window.
4. Inspect OOM events, restarts, and eviction events.
5. Examine whether memory usage increases continuously or tracks workload volume.
6. Inspect application logs and traces for allocation failures, timeouts, or degraded performance.
7. Determine whether the problem is a container limit, host pressure, a potential leak, or an unrelated bottleneck.

## PromQL

Run these only when the metrics exist; confirm names and labels first.

| Signal | Coroot node agent | cAdvisor (if scraped) |
|---|---|---|
| RSS (gauge) | `container_resources_memory_rss_bytes` | `container_memory_rss` |
| Page cache (gauge) | `container_resources_memory_cache_bytes` | — |
| Memory limit (gauge) | `container_resources_memory_limit_bytes` | — |
| RSS as % of limit | `100 * container_resources_memory_rss_bytes / container_resources_memory_limit_bytes` (same container selector on both sides) | — |
| OOM kills in window | `increase(container_oom_kills_total[1h])` | — |
| Restarts in window | `increase(container_restarts_total[1h])` | — |
| Node total / free | `node_resources_memory_total_bytes`, `node_resources_memory_free_bytes` | — |

Use an appropriate range query or aggregation (for example `max_over_time(...[1h])` for peaks) for the target container and time window. Do not interpret a raw gauge as a rate.

## Memory-specific Kubernetes checks

- Check usage relative to the limit.
- Inspect OOM and eviction evidence.
- Evaluate node memory pressure.
- Do not recommend increasing memory limits without considering node capacity.

## Findings to report

Where available:

- Memory request and limit.
- Current and peak memory usage.
- Memory utilization relative to the verified limit.
- OOM events and restart counts.
- Node memory pressure.
- Observed growth over time.
- Relevant runtime or application evidence.

A memory leak is a hypothesis until supported by a sustained growth pattern and appropriate process or runtime evidence.
