# CPU resource allocation and utilization

Mandatory whenever the user asks about CPU allotment, CPU limits, performance, saturation, or resource right-sizing.

## Distinguish allocation from consumption

- **CPU request:** A Kubernetes scheduling reservation used for placement and resource planning.
- **CPU limit:** A configured upper bound on CPU time available to a container, enforced through the applicable runtime and kernel mechanisms.
- **CPU usage:** CPU time actually consumed over a measurement interval.
- **CPU utilization percentage:** A ratio whose denominator must be identified, such as a core, CPU limit, request, or node capacity.
- **CPU throttling:** Restriction of CPU execution when a container reaches an applicable CPU quota.
- **Node CPU utilization:** Consumption across workloads and system processes on the node.

Never use these terms interchangeably. A request is not a guaranteed dedicated physical core. A CPU limit is not evidence that the workload actually uses that amount.

## Workflow

1. Identify the affected application and its node or container scope.
2. Inspect application and node health.
3. Discover available CPU metrics.
4. Retrieve usage over the incident window.
5. Retrieve limits, requests, and throttling metrics when available.
6. Compare usage and throttling with application latency, error rate, and workload volume.
7. Check whether other workloads are competing for node CPU.
8. Determine whether the evidence indicates under-allocation, saturation, contention, or an unrelated bottleneck.

## PromQL

Run these only when the metrics exist; confirm names and label sets first. Apply the same container selector to every series in a query.

CPU consumption in cores (Coroot node agent):

```promql
rate(container_resources_cpu_usage_seconds_total[5m])
```

CPU consumption as a percentage of the CPU limit (Coroot node agent):

```promql
100 * rate(container_resources_cpu_usage_seconds_total[5m]) / container_resources_cpu_limit_cores
```

If no limit metric is available, replace the denominator with a verified numeric limit in cores. Never send a placeholder as executable PromQL.

Throttled time per second (Coroot node agent):

```promql
rate(container_resources_cpu_throttled_seconds_total[5m])
```

Time spent waiting for a CPU (Coroot node agent):

```promql
rate(container_resources_cpu_delay_seconds_total[5m])
```

Throttling ratio, only where cAdvisor cgroup/CFS metrics are scraped:

```promql
rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])
```

Metric names, labels, and collection methods vary by environment. A high throttling ratio can indicate quota pressure, but it does not independently prove that throttling caused an application slowdown.

## CPU-specific Kubernetes checks

- Check whether the workload is frequently throttled.
- Check whether the node is CPU-saturated.
- Determine whether the container limit is restrictive relative to its workload.
- Consider whether raising a limit would merely transfer contention to the node.

## Findings to report

Where data is available:

- Configured CPU request.
- Configured CPU limit.
- Observed CPU usage in cores.
- Utilization relative to the verified limit.
- CPU throttling ratio and/or throttled time.
- Node CPU usage and capacity.
- Measurement window and workload volume.
- Relationship to latency and errors.

If requests or limits are unavailable, state explicitly that CPU allocation could not be verified.
