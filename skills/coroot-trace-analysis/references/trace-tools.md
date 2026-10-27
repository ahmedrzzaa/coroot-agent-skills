# Coroot MCP trace tools

Source: github.com/coroot/coroot `api/mcp.go`, `api/views/overview/traces.go`, `clickhouse/traces.go`, `model/trace.go`, `utils/url.go` (main, read 2026-10-27). Older servers differ; trust the schema the connected server advertises.

## Common behaviour

- Requires `select_project` first and the `project.traces:view` permission.
- `from` / `to`: epoch milliseconds or relative values such as `now-1h`. Default: the server's default range (1h) up to now.
- `service` → exact match on `ServiceName`; `span` → exact match on `SpanName`. **Both apply to root spans only** (`ParentSpanId = ''`). In OTel mode, filtering by a downstream service returns nothing.
- Automatically excluded: root spans named like `GET /health`, `/healthz`, `/metrics`, `/debug/…`, `/actuator/…`; spans whose peer is a monitoring or control-plane pod; in eBPF mode, monitoring and control-plane services.
- Size limits: lists cut at 50 KB (keeping at least one item, with `truncated` and `hint`); any response cut at 80 KB with a `[truncated: …]` note that leaves invalid JSON.
- Without ClickHouse or without traces, tools return `null` or an empty list with no explanation.

## traces_summary

| Argument | Default |
|---|---|
| `service`, `span` | none (all root spans) |
| `from`, `to` | server default range |

Computes root-span statistics per cluster, service, and span from a latency histogram (buckets 0, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000 ms).

Output: `{overall, total, returned, truncated?, hint?, items[]}`; each item has `cluster`, `service_name`, `span_name`, `total` (requests per second), `failed` (error fraction 0–1), `duration_quantiles` (`[p50, p95, p99]` in **ms**, interpolated within buckets, **capped at 10000**). Sorted by `total`, highest first.

- **No trace IDs** are returned.
- `overall.duration_quantiles` is built incorrectly in current code (buckets overwritten, not summed); use per-endpoint percentiles. `overall.total` and `overall.failed` are fine.

## traces_errors

| Argument | Default |
|---|---|
| `service`, `span`, `from`, `to` | as above |

Loads the **200 most recent traces whose root span has an error status**, keeps only the deepest error spans (removing ancestors that merely propagated the error), and groups them by service, span name, and labels.

Output items: `cluster`, `service_name`, `span_name`, `labels`, `sample_trace_id`, `sample_error`, `count`. **`count` is the share of all origin errors (0–1).** Sorted by share.

- `sample_error` is taken from, in order: status message, `grpc.error_message`, the `exception.message` event, then `HTTP-<code>`.
- `labels` come from `net.peer.name`, `server.address`, `net.host.name`, `http.route`, `http.host.name`, `db.system`, `db.operation`, `messaging.*`.

## traces_outliers

| Argument | Default |
|---|---|
| `service`, `span`, `from`, `to` | as above |
| `dur_from` | `"1s"` |
| `dur_to` | `"inf"` |

Builds a flamegraph from the **200 most recent root spans in the duration band**. **No baseline is fetched, so the result is the slow band only, not a diff.**

- Output: `{type: "::nanoseconds", flamegraph: {name: "total", total, self, comp, children[], data: {trace_id}}, diff: false}`.
- Each node is named `"<service>: <span> <labels>"`; siblings with the same name are merged. Overlapping child time is split equally; `self = total − sum(children)`. `data.trace_id` is a sample for `get_trace`.
- Duration parsing: a plain number is seconds; Go durations such as `500ms` work (v1.27.1+); `inf`, `err`, or empty means no bound. `dur_to="err"` selects "in band OR error"; `dur_from="inf"` selects error traces only.
- To compare, call twice: slow band (`dur_from="1"`) and normal band (`dur_from="0"`, `dur_to="0.2"`), then compare each frame's fraction of the root total.

## get_trace

| Argument | Default |
|---|---|
| `trace_id` | required |
| `from`, `to` | do not limit the lookup |

Output: spans sorted by start time, cut at about 50 KB keeping the **earliest** spans (hint says so). Each span: `service`, `trace_id`, `id`, `parent_id`, `name`, `timestamp` (epoch ms), `duration` (ms), `status{error, message}`, `details{text, lang}` (URL or SQL), `attributes` (resource and span attributes merged, plus `Cluster`), `events[]`, `cluster`. There is no span-kind field. `status.message` becomes `HTTP-<code>` when `http.status_code` is present.

## Version history

| Release | Change |
|---|---|
| v1.20.0 | MCP server added; raw output, no size caps, `get_trace` unsorted |
| v1.26.1 | Configurable default time range |
| v1.26.5 | `{total, returned, items}` envelope, 50 KB / 80 KB caps, sorting |
| v1.27.1 | eBPF fallback; duration strings such as `"1s"` parse correctly (before, they parsed as 0, so default `traces_outliers` returned all traffic) |
