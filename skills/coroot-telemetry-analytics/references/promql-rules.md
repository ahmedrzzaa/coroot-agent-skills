# PromQL correctness rules

Sources: https://prometheus.io/docs/prometheus/latest/querying/functions/, https://prometheus.io/docs/practices/histograms/, https://prometheus.io/docs/practices/rules/, https://prometheus.io/docs/prometheus/latest/querying/basics/, https://www.robustperception.io/what-range-should-i-use-with-rate/, https://www.robustperception.io/step-and-query_range/.

## Counters

- `rate()` — per-second average over the window; handles counter resets. Use it for counters only.
- `irate()` — uses only the last two points; for graphing volatile counters, not for analysis.
- `increase()` — `rate()` × window length. Extrapolated, so it can return non-integers; do not present it as an exact count.
- `resets()` — number of counter resets, which usually means process restarts.
- **Rate first, then aggregate:** `sum(rate(x[5m]))`, never `rate(sum(x)[5m])`, or counter resets are missed.

## Windows and step

- Use a rate window of at least 4 × the scrape interval. With Coroot's 15s scrape, use 1m or more.
- If the window is shorter than the query step, samples are skipped. In Coroot `query_metrics`, use `$RANGE` (3 × step) to avoid this.

## Ratios

- Aggregate the numerator and the denominator separately, then divide. Never average per-instance ratios.

## Histograms and percentiles

- `histogram_quantile(q, sum by (le, ...) (rate(x_bucket[w])))` — keep `le` in the aggregation and include the `+Inf` bucket.
- The result is linearly interpolated inside a bucket, so the error is up to the bucket width. With Coroot's buckets, a value between 0.5s and 1s can be off by up to 500ms. If the quantile falls in the top bucket, the result is the top finite bound, not the true value.
- **Never average percentiles** across instances or time. Aggregate buckets, then compute the quantile.
- For threshold questions ("how many requests were under 500ms?"), divide bucket rates directly; it is exact at bucket boundaries.

## Cardinality

- Keep result sets to hundreds of series. Every label combination is a new series.
- Use `sum by (...)` and `topk(k, ...)` to bound output.

## Verified example queries (Coroot metrics)

Server-side error ratio per workload, using Coroot's failure definition:

```promql
sum by (app_id) (rate(container_http_inbound_requests_total{status=~"5.*|failed|grpc:.*",status!="grpc:OK"}[$RANGE]))
/ sum by (app_id) (rate(container_http_inbound_requests_total[$RANGE]))
```

p95 latency per workload:

```promql
histogram_quantile(0.95, sum by (app_id, le) (rate(container_http_inbound_requests_duration_seconds_total_bucket[$RANGE])))
```

Share of requests faster than 500ms (Coroot's latency SLI shape):

```promql
sum by (app_id) (rate(container_http_inbound_requests_duration_seconds_total_bucket{le="0.5"}[$RANGE]))
/ sum by (app_id) (rate(container_http_inbound_requests_duration_seconds_total_bucket{le="+Inf"}[$RANGE]))
```

Mean latency (hides the tail; report alongside percentiles, not instead of them):

```promql
sum by (app_id) (rate(container_http_inbound_requests_duration_seconds_total_sum[$RANGE]))
/ sum by (app_id) (rate(container_http_inbound_requests_duration_seconds_total_count[$RANGE]))
```

Client-side Postgres failure ratio per caller and database instance:

```promql
sum by (app_id, actual_destination) (rate(container_postgres_queries_total{status="failed"}[$RANGE]))
/ sum by (app_id, actual_destination) (rate(container_postgres_queries_total[$RANGE]))
```

Node CPU utilisation % (Coroot's own query):

```promql
sum(rate(node_resources_cpu_usage_seconds_total{mode!="idle"}[$RANGE])) without(mode)
/ sum(rate(node_resources_cpu_usage_seconds_total[$RANGE])) without(mode) * 100
```

Top 10 callers by request rate:

```promql
topk(10, sum by (app_id) (rate(container_http_requests_total[$RANGE])))
```
