# Baselines, anomaly detection, and statistical traps

## Comparing windows

- Compare windows of the same length, with the same rate window and step.
- Account for daily and weekly cycles: compare with the same time yesterday (`offset 1d`) or last week (`offset 1w`), not with the previous hour, unless the change is abrupt.
- A median of several past weeks resists one-off events such as holidays. GitLab uses offsets of 166h, 334h, and 502h (https://about.gitlab.com/blog/2019/07/23/anomaly-detection-using-prometheus/).
- Coroot `query_metrics` limits a query to 24h, but `offset` reaches further back inside one query. On ClickHouse-backed metric storage the default retention is 7 days, so `offset 1w` may be empty; prefer `offset 1d` there.
- Compare ratios, not raw counts: errors per request, not error totals. An SLI is good events divided by total events (https://sre.google/workbook/implementing-slos/).
- Watch sample size. At 10 requests per hour, one failure is a 10% error rate (https://sre.google/workbook/alerting-on-slos/). Report request volume with every ratio, and treat low-traffic ratios as weak evidence.

Current traffic relative to the same time yesterday:

```promql
sum(rate(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[$RANGE]))
/ sum(rate(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[$RANGE] offset 1d))
```

The `app_id` label value has the form `/k8s/<namespace>/<workload>`; confirm it with a raw query.

## Simple anomaly detection

A z-score measures how many standard deviations the current value is from its recent mean; beyond about ±3 is unusual:

```promql
(
  sum(rate(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[5m]))
  - avg_over_time(sum(rate(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[5m]))[1d:5m] offset 1h)
)
/ stddev_over_time(sum(rate(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[5m]))[1d:5m] offset 1h)
```

Limitations:

- It assumes a normal distribution; error rates, latencies, and queue lengths usually are not normal (GitLab article above). Treat z-scores as a screening tool, not proof.
- `offset 1h` keeps the current event out of its own baseline.
- Subqueries such as `[1d:5m]` are expensive; scope them to one application.
- A standard deviation of zero produces NaN or Inf, returned as `null`.
- For spiky or low-traffic series, a plain ratio to the `offset 1d` baseline is more robust.

## Statistical traps

- **Correlation versus causation.** Coincident changes and shared causes produce spurious correlations. Recent changes are good places to start, not conclusions (https://sre.google/sre-book/effective-troubleshooting/).
- **Simpson's paradox.** An effect can appear, vanish, or reverse when data is split into groups (https://plato.stanford.edu/entries/paradox-simpson/). Break results down by instance, caller, or destination before trusting a fleet-wide number.
- **Averages hide the tail.** With a 100ms mean, 1% of requests can take 5 seconds (https://sre.google/sre-book/monitoring-distributed-systems/). Use percentiles or the share under a threshold.
- **Mixed-outcome latency.** Coroot's latency histograms have no `status` label. A spike in fast failures can lower latency; check the error ratio at the same time.
- **Sampled traces are biased.** Head sampling misses errors; error-biased tail sampling over-represents them (https://opentelemetry.io/docs/concepts/sampling/). Take rates from metrics, not from counting traces. Coroot's eBPF spans are single client-side spans, not complete traces.
- **Truncated or capped data.** A truncated series list or a `null` gap is missing data, not a zero.
