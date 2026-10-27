# Coroot features: what exists, defaults, and what MCP can reach

Use this to interpret what Coroot reports and to know which evidence is reachable through MCP. Defaults come from docs.coroot.com and, where the docs give none, from `model/check.go` in github.com/coroot/coroot (commit 15137ef, 2026-10-21). Defaults are project-wide and can be overridden per application, so treat them as typical values and confirm the actual threshold from the inspection output.

## Architecture: where each signal comes from

| Component | Provides |
|---|---|
| `coroot-node-agent` (eBPF DaemonSet) | Container metrics, TCP connections, L7 request metrics, logs, eBPF spans, CPU profiles. Pull or push (Remote Write, OTLP). |
| `coroot-cluster-agent` | Postgres, MySQL, Redis, Memcached, MongoDB metrics; Go pprof scraping; AWS RDS and ElastiCache. |
| Prometheus-compatible store (Prometheus, VictoriaMetrics, Thanos, Mimir) | Metrics, with a local Coroot cache. |
| ClickHouse | Logs, traces, profiles, and optionally metrics. |

Requirements: Linux kernel 4.16+ (RHEL/CentOS 7.6+ with limitations); eBPF profiling needs CO-RE (Ubuntu 20.10+, Debian 11+, RHEL 8.2+). Not supported: Docker-in-Docker setups such as MiniKube, WSL1. Missing signals on a host often trace back to these requirements.

Sources: https://docs.coroot.com/installation/architecture/, https://docs.coroot.com/installation/requirements/

## Inspections and default thresholds

Inspections: SLO, Instances, CPU, GPU, Memory, Storage, Network, Logs, JVM, .NET, Python (GIL), Postgres, MongoDB, MySQL, Memcached, Redis, Deployment Tracking (https://docs.coroot.com/inspections/).

| Area | Default trigger |
|---|---|
| SLO | Availability 99%; latency 99% of requests under 500ms (5s for `control-plane` and `monitoring` apps); LLM latency 99% under 10s to first byte |
| CPU | Node CPU above 80%; container CPU above 80% of its limit; throttling and CPU delay are inspected |
| Memory | Any OOM kill; leak above 10%/hour (needs growth of at least 5% of limit or 50 MB, over at least 1/3 of the window); memory pressure stall above 0.02 s/s |
| Storage | Disk I/O load above 5 s/s; disk space above 80% |
| Network | RTT above 10ms in-cluster, 100ms cross-cluster, 200ms external; any unavailable upstream; any failed TCP connect |
| Instances | Available instances below 75% of desired; more than 1 restart |
| Deployment | Rollout in progress longer than 180s (Stuck) |
| Logs | Any ERROR or CRITICAL message |
| Runtimes | JVM safepoint above 0.05s; Python GIL wait above 0.05s; Node.js event loop blocked above 0.7s; JVM/.NET unavailable |
| DNS | p95 latency above 0.1s; server errors (excluding NXDOMAIN); NXDOMAIN on previously valid names |
| Postgres | Unavailable; avg query latency above 0.1s; replication lag above 30s; connections above 90% of `max_connections`; checkpoints above 3x `checkpoint_timeout`; WAL archiving failing; XID wraparound above 50%; bloat above 50%; no backup within 24h |
| MySQL | Unavailable; replication thread stopped; lag above 30s; connections above 90%; aborted connects above 1/s; latency above 0.1s; no backup within 24h |
| MongoDB | Unavailable; replication lag above 30s; latency above 0.1s; oplog window under 1h; connections above 90%; queued operations; fragmentation above 50% |
| Redis / Memcached | Redis latency above 5ms; unavailable |

A failed check means the threshold was crossed, not that it caused the user's symptom.

## SLOs

- A request fails when its status is `failed`, HTTP 5xx, or a non-OK gRPC status. LLM errors count against availability.
- Configure in the UI, with custom Prometheus metrics, or with annotations `coroot.com/slo-availability-objective`, `coroot.com/slo-latency-objective`, `coroot.com/slo-latency-threshold`. Annotation-defined SLOs cannot be edited in the UI. An objective of 0% disables the SLO. Latency thresholds must match a predefined histogram bucket.

Source: https://docs.coroot.com/inspections/slo

## Incidents versus alerts

These are separate objects.

- **Incidents** are SLO violations, opened by multi-window burn-rate rules: burn rate 14.4 over 1h and 5m, or burn rate 6 over 6h and 15m. Both the long and the short window must exceed the rate, and at least half of each window must have valid data. An incident resolves when the burn rate returns to OK. Coroot runs RCA on incidents (Enterprise). Source: https://docs.coroot.com/alerting/slo-monitoring
- **Alerts** come from about 30 built-in rules plus custom rules. Rule sources: an inspection check, log patterns, Kubernetes events, or PromQL. Severity is Warning or Critical, with `For` and `Keep firing for` durations. Built-in rules can be disabled but not deleted. Notifications go to Slack, Microsoft Teams, PagerDuty, Opsgenie, or webhooks, routed per application category. Source: https://docs.coroot.com/alerting/alerts

`resolve_alerts` is the only MCP write tool. It sends notifications downstream and needs the Editor role or higher. The tool description says an alert whose condition still holds re-fires on the next evaluation. A reading of the source suggests otherwise: the alert stays silently updated and only re-fires after the condition clears and returns (inferred, not tested). Never resolve an alert to make a dashboard look healthy.

## Distributed tracing

- **eBPF spans:** node-agent captures outbound HTTP, Postgres, MySQL, Redis, MongoDB, and Memcached requests. Each is a single client-side span, not an end-to-end trace, with a service name like `/k8s/<namespace>/<workload>`. The MCP trace tools fall back to these spans when a project has no OpenTelemetry traces. Do not read a missing downstream span as "the downstream was not called".
- **OpenTelemetry SDK traces** (OTLP over HTTP) give full end-to-end trees.
- Traces are stored in ClickHouse. The UI offers a heatmap, error-origin analysis, attribute comparison against a baseline, and a latency flamegraph diff. In MCP these are `traces_summary`, `traces_errors`, `traces_outliers`, and `get_trace`.

Sources: https://docs.coroot.com/tracing/ebpf-based-tracing, https://docs.coroot.com/tracing/overview

## Continuous profiling (not exposed through MCP)

- eBPF CPU profiling of every process on the node. Java needs `-XX:+PreserveFramePointer`; Node.js needs perf-map flags.
- Go: heap profiles from node-agent; CPU, memory, block, and mutex pprof scraped by cluster-agent from pods annotated `coroot.com/profile-scrape` and `coroot.com/profile-port`.
- Java: async-profiler (CPU, allocation, lock) when `ENABLE_JAVA_ASYNC_PROFILER=true`, HotSpot JDK 8+.

No MCP tool returns profiles. When a CPU or lock hotspot needs a flamegraph, point the user to the application's Profiling tab in the Coroot UI.

Sources: https://docs.coroot.com/profiling/overview, https://docs.coroot.com/profiling/go-profiling, https://docs.coroot.com/profiling/java-profiling

## Service map semantics

Dependencies are discovered from eBPF TCP connect, listen, and retransmit events, with no code changes.

- **RTT** (`container_net_latency_seconds`) is network round-trip time to the peer, measured with ICMP.
- **Latency** in dependency output is L7 request latency for the protocol, which is different from RTT and from TCP connect time.
- **Connectivity problem:** RTT data existed but recent samples are missing.
- **Failed connections:** the latest failed TCP connect count is above zero. It counts TCP-level failures only, not HTTP errors.

Either of the last two marks the connection CRITICAL.

Source: https://docs.coroot.com/metrics/node-agent/

## Deployment tracking (not exposed through MCP)

Detected automatically from Kubernetes Deployments via kube-state-metrics; each new ReplicaSet is a version. States: Started, InProgress, Stuck (over 3 minutes), Cancelled, Deployed. Thirty minutes after a rollout Coroot compares the new version with the previous one on SLOs, log errors, restarts, CPU, memory, leaks, and monthly CPU/memory cost. Source: https://docs.coroot.com/inspections/deployment-tracking

## Costs and risks (not exposed through MCP)

- **Costs:** AWS, GCP, and Azure list prices; other environments use adjustable GCP C4 pricing. Covers CPU, memory, and traffic; ignores reserved-instance discounts. Shows per-app usage, allocation, overprovisioning, and node idle cost. Source: https://docs.coroot.com/costs/overview
- **Risks:** availability (single instance, all instances on one node, single AZ, spot-only, unreplicated databases) and security (publicly exposed databases). Risks can be dismissed with a reason. Source: https://docs.coroot.com/risks/overview

## AI root-cause analysis and editions

- AI RCA is Enterprise Edition. Its ML analysis walks the dependency graph using metrics, logs, traces, and profiles; only findings, not raw telemetry, go to the LLM. Community Edition users get a small monthly allowance through Coroot Cloud.
- MCP `list_anomalies` and `investigate_anomaly` are Enterprise only; the open-source server registers 17 tools without them. `get_incident_details` returns any RCA already stored on the incident.

Sources: https://docs.coroot.com/ai/overview, https://docs.coroot.com/ai/configuration/

## MCP access and limits

- Endpoint `/mcp` (streamable HTTP). OAuth 2.0 for interactive clients; `Authorization: Bearer <api key>` for headless agents. API keys belong to service accounts and follow the same RBAC as users (roles: Admin, Editor, Viewer).
- Every tool call is authorized server-side. Responses are capped at about 50 KB for lists and 80 KB otherwise, so narrow filters and time ranges instead of assuming completeness.
- `list_alerts`: `state` = `firing` (default), `resolved`, or `any`; optional `app_id`; `limit` up to 1000.

Sources: https://docs.coroot.com/mcp/overview/, https://docs.coroot.com/configuration/authentication/
