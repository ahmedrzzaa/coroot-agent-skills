# Kubernetes resource allocation and node capacity

When Kubernetes metrics or configuration data are available, evaluate resource allocation at both container and node levels.

## Investigate

- CPU and memory requests/limits.
- Pod replica count.
- Container restarts and readiness.
- Node capacity and allocatable resources.
- Requested resources versus allocatable capacity.
- Current workload usage.
- CPU throttling and memory pressure.
- Uneven pod placement or concentrated workload.
- Scheduling failures, evictions, and OOM events.

Do not calculate remaining allocatable capacity from usage alone. Requests, limits, actual usage, and allocatable resources represent different quantities.

## Request utilization

For a resource with a known request:

`Request utilization = Observed usage / Configured request × 100`

This is a comparison ratio, not a scheduling guarantee. Values above 100% are possible because actual usage can exceed a request.

## CPU

- Check whether the workload is frequently throttled.
- Check whether the node is CPU-saturated.
- Determine whether the container limit is restrictive relative to its workload.
- Consider whether raising a limit would merely transfer contention to the node.

## Memory

- Check usage relative to the limit.
- Inspect OOM and eviction evidence.
- Evaluate node memory pressure.
- Do not recommend increasing memory limits without considering node capacity.

## Missing data

If the connected Coroot tool does not return Kubernetes requests, limits, or scheduling data, identify the missing evidence rather than estimating those values.
