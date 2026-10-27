---
name: coroot-telemetry-analytics
description: Correct quantitative analysis of Coroot metrics with PromQL - request rate, error ratio, and latency percentiles from Coroot's eBPF request metrics, RED and USE methods, baseline and time-window comparison, simple anomaly detection, SLO error-budget and burn-rate maths, and statistical traps such as averaged percentiles and Simpson's paradox. Use when writing or interpreting query_metrics PromQL, quantifying how much worse a service got, comparing periods, checking an SLO or burn rate, or validating numbers before they go into a report.
allowed-tools: coroot_query_metrics, coroot_get_app_detail, coroot_get_applications, coroot_get_incidents, coroot_get_incident_detail, coroot_get_traces
metadata:
  author: aurora
  version: "3.0"
  category: observability
  documentation: "https://docs.coroot.com/metrics/node-agent"
  mcp_endpoint: "/mcp"
  connection_check: "chat.backend.agent.tools.coroot_tool:is_coroot_connected"
  rca_priority: "3"
---

# Coroot Telemetry Analytics

Turn Coroot metrics into numbers that are correct and comparable. Most analysis errors come from the wrong query shape (averaged percentiles, rate after sum), the wrong comparison (raw counts across different traffic), or reading too much into small samples.

## Ground rules

- Discover before querying: `list_metric_names` (regex filter) only returns metrics that reported in the last 5 minutes. Confirm label names with a small raw query before building aggregations.
- Every number needs a denominator and a window. State rates as per-second over a named window, ratios as bad ÷ total, and percentiles with their bucket resolution.
- Report request volume next to every ratio. One failure in ten requests is a 10% error rate.
- Separate observation from inference. A metric that moves with the symptom is a candidate cause, not a proven one.
- Never send placeholders as executable PromQL.

Tool calls use the official Coroot MCP names, with the local wrapper in parentheses.

## `query_metrics` behaviour (official MCP)

Values below are from the Coroot source (`api/mcp.go`, 2026-10-21). The local wrapper (`coroot_query_metrics`) has its own limits; follow its schema.

| Setting | Value |
|---|---|
| Time range | Default 1h, maximum 24h |
| Step | Default 30s (project refresh interval), minimum 15s; widened automatically to keep at most 120 points per series (24h gives a 720s step) |
| Series | Default limit 100, maximum 1000; beyond that `truncated: true` |
| `$RANGE` | Replaced with 3 × step, so `rate(x[$RANGE])` never undersamples |
| NaN / Inf | Returned as `null` |

Consequences:

- Aggregate with `sum by (...)` or `topk` instead of relying on truncated per-series output.
- Compare to a baseline with `offset 1d` inside one query rather than querying beyond 24h. On ClickHouse-backed metric storage the default retention is 7 days, so `offset 1w` may return nothing.

## Choose the method

| Situation | Method | Coroot signals |
|---|---|---|
| Request-driven service is slow or failing | **RED**: rate, errors, duration | `container_*_inbound_requests_total` (and `*_queries_total`), the `status` label, `*_duration_seconds_total_bucket` |
| Host or container resource suspected | **USE**: utilization, saturation, errors | CPU usage versus limit, CPU delay and throttling, RSS versus memory limit, OOM kills, TCP retransmits and failed connects |
| User-facing health summary | **Four golden signals**: latency, traffic, errors, saturation | RED plus saturation; keep successful and failed latency separate where possible |

For CPU, memory, and capacity interpretation, use `coroot-resource-analysis`.

## Workflow

1. Define the question as a number: "error ratio of checkout inbound HTTP over the last 30 minutes versus the same time yesterday".
2. Find the metric and labels ([references/coroot-metrics.md](references/coroot-metrics.md)); confirm with `list_metric_names` and a raw query.
3. Write the query using [references/promql-rules.md](references/promql-rules.md). Take rate before aggregating; aggregate the numerator and denominator separately, then divide.
4. Query the incident window and a same-length baseline ([references/baselines-and-anomalies.md](references/baselines-and-anomalies.md)).
5. Break the result down `by (app_id, actual_destination)` or by instance before trusting an aggregate.
6. For SLO questions, compute the error budget and burn rate ([references/slo-burn-rate.md](references/slo-burn-rate.md)) and compare with Coroot's incident rules.
7. Check the result against Coroot's own view (`get_application_status`, incidents). If they disagree, explain why (custom SLI, client-side fallback, different window) before reporting.

## Reporting numbers

- Give the value, unit, window, and baseline: "error ratio 6.2% (1,240 of 20,000 requests) from 14:00 to 14:30 UTC, versus 0.3% at the same time yesterday".
- For latency, name the percentile and the bucket limits: "p95 between 0.5s and 1s (histogram resolution)".
- List data gaps: truncated series, `null` points, missing metrics, range capped at 24h.

## References

- [references/coroot-metrics.md](references/coroot-metrics.md) — Coroot request, latency, and resource metric names, labels, `status` values, failure definition, histogram buckets.
- [references/promql-rules.md](references/promql-rules.md) — PromQL correctness rules and verified example queries.
- [references/baselines-and-anomalies.md](references/baselines-and-anomalies.md) — window comparison, seasonality, z-score anomaly detection, statistical traps.
- [references/slo-burn-rate.md](references/slo-burn-rate.md) — Coroot SLO definitions and incident rules, error-budget and burn-rate maths, reproduction queries.
