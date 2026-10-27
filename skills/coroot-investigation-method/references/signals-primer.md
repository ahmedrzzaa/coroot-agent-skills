# Observability signals primer

Short explanations to give during an investigation, with how Coroot collects each signal. Coroot specifics come from https://docs.coroot.com (read 2026-10-27).

## Metrics

A metric is a named number with labels, sampled over time: `metric_name{label="value"} 42`. A scraper (Prometheus, or Coroot's agents) collects them at an interval such as 15 seconds into a time-series database.

| Type | Behaviour | Read it as | Example |
|---|---|---|---|
| **Counter** | Only goes up (resets on restart) | A rate: `rate(x[5m])` per second | Requests served, errors |
| **Gauge** | Goes up and down | The current value | CPU in use, memory, open connections |
| **Histogram** | Counts observations into buckets (0–10ms, 10–30ms, …) | Percentiles: `histogram_quantile(0.99, …)` | Request duration |

**Percentiles:**

- **p50 (median):** half of requests are faster.
- **p95:** early warning for slow requests.
- **p99:** the experience of the slowest 1% — "99% of requests finish faster than this".
- A low p50 with a high p99 means most requests are fine and a minority are very slow: look for something that affects only some requests (one dependency, one instance, a queue).
- Never average percentiles; compute them from summed buckets.

**In Coroot:** request rate, errors, and latency histograms for HTTP, gRPC, and database protocols come from **eBPF** on every container — no client library needed. Applications can add their own Prometheus metrics with scrape annotations. Metrics are aggregates: they never show an individual request.

## Logs

Lines the code writes about what it is doing, with a severity (debug, info, warning, error, fatal). **Structured logs** (JSON) can be filtered by fields; plain text can only be searched.

**In Coroot:** the node agent collects container stdout/stderr, journald, and log files automatically, detects severity, and groups warning-and-above messages into **patterns** (same message with different numbers or IDs). Applications can also send logs over OpenTelemetry. A log only exists if a developer wrote the statement — missing logs are not evidence of nothing happening.

## Trace IDs and propagation

When a request enters the system (often at the gateway), it gets a unique **trace ID**, passed to every downstream service in a request header (W3C `traceparent`). Each service keeps and forwards it, so one ID follows the request everywhere.

- Logs only carry the trace ID if the logger includes it. Services that forget break the log-to-log path; the trace itself still works.
- **In Coroot:** logs sent with OpenTelemetry trace context, or JSON logs with fields such as `trace_id`, link to traces; `query_logs` entries include `trace_id` for `get_trace`.

## Traces and spans

A **trace** is the timeline of one request. It is made of **spans**; each span is one unit of work (an incoming request, an outgoing call, a query, waiting for a connection) with a start, duration, parent, status, and **attributes** (key-value context such as `db.system`, pool sizes, URLs).

- The widest span on the critical path is where the time went.
- **Self time** is work inside the span; the rest is waiting on its children.
- **In Coroot:** OpenTelemetry-instrumented services produce full traces. Without instrumentation, eBPF produces client-side spans for outgoing calls only, each as its own trace — useful for "which calls are slow", not for "which request caused them".

## HTTP status codes at gateways

| Code | Meaning | Where to look |
|---|---|---|
| 500 | The service itself failed | That service's logs and traces |
| 502 Bad Gateway | The upstream returned an invalid response or refused the connection | Upstream crashes, restarts, failed connects |
| 503 Service Unavailable | No healthy upstream, or the service is shedding load | Instance availability, readiness, saturation |
| 504 Gateway Timeout | The upstream did not answer in time | Upstream latency, downstream of the upstream |
| 499 (nginx) | The client gave up first | Client timeouts shorter than server latency |

Coroot counts 5xx (and failed or non-OK gRPC) as failures; 4xx are not failures in its SLOs.
