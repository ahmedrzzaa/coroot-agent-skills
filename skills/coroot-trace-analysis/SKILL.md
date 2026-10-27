---
name: coroot-trace-analysis
description: Investigates request latency and errors with Coroot distributed traces - triage with traces_summary, error-origin analysis with traces_errors, slow-band flamegraphs with traces_outliers, and span-by-span reading with get_trace - while accounting for root-span-only filters, eBPF client-only spans versus OpenTelemetry end-to-end traces, sampling, version differences, and response truncation. Use when the user asks which endpoint is slow or failing, why p95 or p99 latency rose, where an error originates, what a specific trace shows, or whether to add OpenTelemetry instrumentation.
allowed-tools: coroot_get_traces, coroot_get_app_detail, coroot_get_applications, coroot_get_app_logs, coroot_query_metrics
metadata:
  author: aurora
  version: "3.0"
  category: observability
  documentation: "https://docs.coroot.com/tracing/overview"
  mcp_endpoint: "/mcp"
  connection_check: "chat.backend.agent.tools.coroot_tool:is_coroot_connected"
  rca_priority: "3"
---

# Coroot Trace Analysis

Traces show where time and errors go inside a request. Coroot's four MCP trace tools behave differently from what their names suggest; read the caveats before drawing conclusions.

## Know which traces you have

- **OpenTelemetry traces** (apps instrumented with an OTel SDK): end-to-end trees across services, with attributes, events, and exceptions.
- **eBPF spans** (no instrumentation): one client-side span per outbound call, each its own trace. `service` is the **caller** (`/k8s/<namespace>/<workload>`); the callee is in `net.peer.name` or `http.url`. Durations include network and queueing. eBPF marks HTTP 4xx as errors.
- The MCP tools use eBPF spans **only when the project has no OpenTelemetry traces at all** (Coroot v1.27.1+). If any service sends OTel traces, uninstrumented services are invisible to the trace tools; use `get_application_status` dependencies for them.

Details: [references/ebpf-vs-otel.md](references/ebpf-vs-otel.md).

## Tool caveats (read before use)

- `service` and `span` filters are **exact matches on root spans only**. Filter by the entry service (the one that received the request), not a downstream one. `service` is the OTel `service.name`, not the Coroot application ID.
- `traces_summary` returns per-endpoint numbers and **no trace IDs**. Its `overall` percentiles are unreliable; use per-endpoint values. Durations are milliseconds, capped at 10000.
- `traces_errors` looks at the **200 most recent** failing traces; `count` is a **share (0–1)** of origin errors, not a count.
- `traces_outliers` returns the flamegraph of the **200 most recent** traces in the duration band. **It is not a diff against normal traffic**, whatever its description says. To compare, call it twice (slow band and fast band). On servers older than v1.27.1, duration strings such as `"1s"` parse as 0; pass numbers of seconds (`"1"`, `"0.5"`).
- `get_trace` keeps the **earliest** spans when the trace exceeds about 50 KB; later spans may be missing.
- Responses over 80 KB are cut mid-text (invalid JSON plus a `[truncated: …]` note). Narrow the time range or filters instead of parsing a cut response.
- An empty result can mean no ClickHouse, no traces, or a missing permission (`project.traces:view`), not "no problem".

Full argument tables: [references/trace-tools.md](references/trace-tools.md).

## Workflow

1. **Scope.** Entry service, endpoint, incident window, and a normal baseline window.
2. **Triage:** `traces_summary(service, from, to)` → endpoints by rate, error fraction, p50/p95/p99. Repeat for the baseline window; compare per endpoint.
3. **Errors:** `traces_errors(service, span, from, to)` → origin errors (the deepest failing spans), grouped by service, span, and labels such as `db.system` or `http.route`. Ancestors that only propagated the error are removed. Take `sample_trace_id`.
4. **Latency:** `traces_outliers(service, span, dur_from, dur_to)` for the slow band, then again for a normal band (for example `dur_from="0"`, `dur_to="0.2"`). Compare each frame's share of the root total; look for high `self` time (work in the process) versus wide children (downstream calls).
5. **Read examples:** `get_trace(trace_id)` from `traces_errors` samples or flamegraph `data.trace_id`. Find the critical path, repeated sequential calls (N+1), fan-out, retries, and the deepest error.
6. **Cross-check with metrics and logs.** Rates and error ratios come from metrics, not trace counts (traces are sampled and capped); see `coroot-telemetry-analytics`. Pivot from trace IDs to logs in the Coroot UI ("Show logs"), or search logs around the span timestamps with `coroot-log-analysis`.
7. **Recommend instrumentation** when only eBPF spans exist and the question needs causality across services ([references/ebpf-vs-otel.md](references/ebpf-vs-otel.md) has setup and overhead).

Analysis patterns: [references/analysis-patterns.md](references/analysis-patterns.md).

## Report

- Window, entry service, endpoints, and whether data is OTel or eBPF-only.
- Per endpoint: rate, error fraction, p95/p99 now versus baseline.
- Error origin: service, span, labels, sample error message (sanitized), share.
- Latency: frames that grew between fast and slow bands, critical-path span, pattern (N+1, fan-out, retries, self time).
- Example trace IDs for the user to open in the Coroot UI.
- Caveats: sampling, 200-trace cap, truncation, server version, eBPF limitations.
