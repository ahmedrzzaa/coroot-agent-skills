# Network, disk, and dependency bottlenecks

For suspected infrastructure bottlenecks, investigate the available evidence for:

- TCP connection failures and retransmissions.
- Network latency and request failures.
- Disk latency and throughput.
- Database latency and connection failures.
- Cache latency and connection errors.
- DNS failures, when exposed.
- Dependency-level trace durations.
- Resource contention on shared nodes.

Do not assume every signal is available through eBPF in every environment.

## Where to look

- Dependency map: `coroot_get_service_map()`. With the official MCP, `get_application_status` lists upstream dependencies with connectivity, RTT, and latency, plus downstream clients.
- Dependency-level durations: `coroot_get_traces` (`traces_summary`, `traces_outliers`, `get_trace`); see the `coroot-trace-analysis` skill.
- Database-specific diagnosis (client versus server versus network): see the `coroot-database-analysis` skill.
- Node-level network and disk: `coroot_get_node_detail(node_name='NODE')` (`get_node_details`).

## Coroot node-agent metrics

Confirm names and labels with `list_metric_names` before querying.

| Signal | Metric | Typical use |
|---|---|---|
| Failed TCP connects | `container_net_tcp_failed_connects_total` | `rate()` / `increase()` |
| TCP retransmits | `container_net_tcp_retransmits_total` | `rate()` |
| Round-trip time to remote IPs | `container_net_latency_seconds` | gauge |
| Disk read time | `container_resources_disk_read_time_seconds_total` | `rate()` |
| Disk write time | `container_resources_disk_write_time_seconds_total` | `rate()` |

## Interpretation

A slow database operation might be caused by the database, a network path, connection-pool saturation, lock contention, or upstream application behavior. Use correlated telemetry to distinguish these possibilities. A dependency relationship identifies a potential failure path; it does not prove causality.
