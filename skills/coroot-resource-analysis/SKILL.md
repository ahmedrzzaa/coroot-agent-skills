---
name: coroot-resource-analysis
description: CPU, memory, thread and process, Kubernetes capacity, network, disk, and cost analysis with Coroot telemetry - allocation versus consumption, CPU throttling, OOM kills and memory pressure, single-thread bottlenecks, requests and limits versus node allocatable capacity, dependency bottlenecks, and evidence-based right-sizing. Use when the user asks about CPU or memory allotment, limits, throttling, saturation, OOMs, thread contention, node capacity, slow disk or network, infrastructure cost, or right-sizing.
allowed-tools: coroot_get_applications, coroot_get_app_detail, coroot_get_app_logs, coroot_get_traces, coroot_get_service_map, coroot_query_metrics, coroot_get_deployments, coroot_get_nodes, coroot_get_node_detail, coroot_get_costs
metadata:
  author: aurora
  version: "3.0"
  category: observability
  documentation: "https://docs.coroot.com/metrics/node-agent"
  mcp_endpoint: "/mcp"
  connection_check: "chat.backend.agent.tools.coroot_tool:is_coroot_connected"
  rca_priority: "3"
---

# Coroot Resource Analysis

## Ground rules

- Ground every claim in retrieved data. A correlation between resource pressure and a symptom is not proof of cause.
- Select the project first and preserve application, container, and node IDs exactly as returned.
- Verify whether each metric represents a container, process, node, or whole cluster before interpreting it.
- Do not combine measurements from different environments or unrelated time windows.
- If requests, limits, or scheduling data are not returned, say so. Identify the missing evidence rather than estimating values.
- Do not resize, restart, or reconfigure anything without explicit authorization; never claim a change was made when only recommending it.

Tool calls use the local `coroot_*` wrappers, with the official Coroot MCP equivalent in parentheses.

## Pick the reference

Read only the files the question needs. Many questions span several (a latency regression often needs CPU, threads, and network).

| Question | Reference |
|---|---|
| CPU allotment, requests, limits, usage, throttling, saturation | [references/cpu.md](references/cpu.md) |
| Memory usage, limits, RSS versus working set versus heap, OOM, leaks | [references/memory.md](references/memory.md) |
| Thread counts, single-thread bottlenecks, lock contention, GC, scheduling delay | [references/threads-and-processes.md](references/threads-and-processes.md) |
| Requests/limits versus allocatable capacity, replicas, placement, evictions | [references/kubernetes-capacity.md](references/kubernetes-capacity.md) |
| TCP failures and retransmits, network latency, disk latency, DB/cache/DNS dependencies | [references/network-disk-dependencies.md](references/network-disk-dependencies.md) |
| Cost, idle capacity, right-sizing | [references/cost-optimization.md](references/cost-optimization.md) |
| Coroot node-agent metric names | [references/metrics.md](references/metrics.md) |
| Writing the final report | [references/report-template.md](references/report-template.md) |

## Allocation is not consumption

These are different quantities. Never use them interchangeably or derive one from another.

- **Request:** a Kubernetes scheduling reservation. Not a guaranteed dedicated core or memory block.
- **Limit:** a configured upper bound. Not evidence that the workload uses that amount.
- **Usage:** what was actually consumed over a measurement interval.
- **Utilization %:** a ratio. Always name the denominator: a core, the limit, the request, or node capacity.

## General workflow

1. Identify the affected application and its container and node scope.
2. Inspect application and node health: `coroot_get_app_detail(app_id='ID')` (`get_application_status`), `coroot_get_nodes()` (`list_nodes`), `coroot_get_node_detail(node_name='NODE')` (`get_node_details`).
3. Discover available metrics (`list_metric_names`) before writing PromQL.
4. Retrieve usage, limits/requests, and pressure signals with `coroot_query_metrics` (`query_metrics`) over the incident window and a comparable baseline.
5. Compare against latency, error rate, and workload volume (`coroot_get_traces` / `traces_summary`).
6. Check whether other workloads compete for the same node.
7. Classify the bottleneck: under-allocation, saturation, contention, waiting (locks, I/O, dependencies), or an unrelated cause.
8. Report with explicit data gaps using [references/report-template.md](references/report-template.md).
