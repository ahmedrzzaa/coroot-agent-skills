# Coroot tools

Two tool sets may be connected: the local `coroot_*` wrappers and the official Coroot MCP server. Use whichever is present and follow its actual schema.

## Local wrapper tools

| Tool | Purpose | Arguments used in these skills |
|---|---|---|
| `coroot_get_incidents` | Discover incidents and available RCA findings. | `lookback_hours` |
| `coroot_get_incident_detail` | Investigate a specific incident. | `incident_key` |
| `coroot_get_applications` | Identify application health and status. | — |
| `coroot_get_app_detail` | Inspect application health checks and diagnostics. | `app_id` |
| `coroot_get_app_logs` | Search logs for a specific application. | `app_id`, `severity` |
| `coroot_get_overview_logs` | Search broader project or cluster log data. | `severity`, `limit` |
| `coroot_get_traces` | Investigate traces, errors, latency, and individual requests where supported. | Follow the connected schema. |
| `coroot_get_service_map` | Discover service dependencies and communication paths. | — |
| `coroot_query_metrics` | Query available metrics through PromQL. | Follow the connected schema. |
| `coroot_get_deployments` | Correlate deployments with incidents and regressions. | `lookback_hours` |
| `coroot_get_nodes` | Identify unhealthy or resource-constrained nodes. | — |
| `coroot_get_node_detail` | Investigate a particular node. | `node_name` |
| `coroot_get_costs` | Examine available infrastructure cost information. | `lookback_hours` |
| `coroot_get_risks` | Review available security and availability risks. | Follow the connected schema. |

Do not invoke undocumented wrapper arguments.

## Official Coroot MCP tools

Served at `/mcp` on the Coroot instance. Verified against https://docs.coroot.com/mcp/overview/ (2026-10-27).

### Community Edition

| Tool | Purpose | Key arguments |
|---|---|---|
| `list_projects` | Discover projects (clusters) the user can access. | — |
| `select_project` | Set the active project for the session. | project id |
| `list_applications` | Triage which apps to look at. | `namespace`, `search`, `min_status` |
| `list_alerts` | See currently firing or recently resolved alerts. | — |
| `list_incidents` | Browse the SLO incident timeline. | — |
| `list_nodes` | Fleet-wide host overview (CPU%, memory%, network, status). | `search` |
| `get_application_status` | One application's health: per-inspection issues, log-pattern samples, upstream dependencies with connectivity/RTT/latency, downstream clients. | application id |
| `get_incident_details` | Full context on one SLO incident. | incident id |
| `get_node_details` | One host: audit report plus CPU/memory/network sparklines. | node id |
| `traces_summary` | Per-endpoint rps, error rate, p50/p95/p99. Pass `service` + `span` to focus on one endpoint. | `service`, `span` |
| `traces_errors` | Top failure reasons grouped by endpoint, with `sample_trace_id` and `sample_error`. | — |
| `traces_outliers` | Flamegraph diff of slow traces in `[dur_from..dur_to]` versus the rest (default `dur_from=1s`). | `dur_from`, `dur_to` |
| `get_trace` | Full span tree for one request, with attributes and events. | `trace_id` |
| `query_metrics` | Run a custom PromQL query. | PromQL expression |
| `list_metric_names` | Discover which metrics exist. | regex filter |
| `query_logs` | App-scoped or project-wide logs with severity, search, and time range; newest first. | search query, `max_body_length` |
| `resolve_alerts` | Manually resolve alerts after fixes. **Write action — requires explicit user authorization.** | alert ids |

### Enterprise Edition only

| Tool | Purpose |
|---|---|
| `list_anomalies` | Surface SLO violations and sub-SLO error or latency spikes across the fleet. |
| `investigate_anomaly` | Find the root cause of a problem in one app. |

Use Enterprise tools only if they are exposed and authorized. Do not assume a Community Edition connection or the local wrappers include them.

### Trace drill-down order

1. `traces_summary` — what is slow or failing.
2. `traces_errors` — why requests fail.
3. `traces_outliers` — why the tail (p95/p99) is slow.
4. `get_trace` — one request end to end, using a `trace_id` from the previous steps.

## Wrapper to official MCP mapping

| Wrapper | Official MCP equivalent |
|---|---|
| `coroot_get_incidents` | `list_incidents` |
| `coroot_get_incident_detail` | `get_incident_details` |
| `coroot_get_applications` | `list_applications` |
| `coroot_get_app_detail` | `get_application_status` |
| `coroot_get_app_logs` | `query_logs` scoped to an application |
| `coroot_get_overview_logs` | `query_logs` project-wide |
| `coroot_get_traces` | `traces_summary`, `traces_errors`, `traces_outliers`, `get_trace` |
| `coroot_get_service_map` | No direct tool; `get_application_status` returns upstream dependencies and downstream clients |
| `coroot_query_metrics` | `query_metrics` (discover names with `list_metric_names`) |
| `coroot_get_deployments` | Not in the documented tool list |
| `coroot_get_nodes` | `list_nodes` |
| `coroot_get_node_detail` | `get_node_details` |
| `coroot_get_costs` | Not in the documented tool list |
| `coroot_get_risks` | Not in the documented tool list |
| — | `list_projects`, `select_project`, `list_alerts`, `resolve_alerts` |

## Status codes

The local integration defines:

- `0` — UNKNOWN
- `1` — OK
- `2` — INFO
- `3` — WARNING
- `4` — CRITICAL

Apply this mapping only to fields that the wrapper documents as using these values.
