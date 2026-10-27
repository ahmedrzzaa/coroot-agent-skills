# Querying logs in Coroot

Sources: https://docs.coroot.com/mcp/overview/, https://docs.coroot.com/logs/querying, and `api/mcp.go` and `clickhouse/logs.go` in github.com/coroot/coroot (commit 15137ef, 2026-10-21).

## Official MCP `query_logs`

| Argument | Behavior |
|---|---|
| `app_id` | Full four-part ID `cluster_id:namespace:Kind:name`. Omit for a project-wide search. |
| `from`, `to` | Epoch milliseconds or relative values such as `now-1h`. Default: the server's default range (1h) up to `now`. |
| `limit` | Default 100, maximum 1000. Out-of-range values silently fall back to 100. |
| `max_body_length` | Default 1000 characters, maximum 10000; longer bodies are cut with "…". Out-of-range values fall back to 1000. Only in Coroot v1.26.5 and later. |
| `severity` | List, OR'd. Values: `unknown`, `trace`, `debug`, `info`, `warning`, `error`, `fatal`. |
| `search` | Full-text search of the log body; see the search rules below. |
| `log_pattern` | A pattern hash from `get_application_status`. With `app_id`, also matches similar patterns. Forces agent-collected logs. |
| `source` | `agent` (node-agent logs) or `otel` (logs sent by the app over OpenTelemetry). Usually omit it: with `app_id`, Coroot prefers the app's OTel logs and falls back to agent logs; without `app_id`, it searches agent logs. Passing `auto` with `app_id` may drop the app scope (read from code, not tested). |

Older servers (before v1.26.5) lack `max_body_length` and the response-size caps. Use the arguments the connected server advertises.

### Search rules

- The text is split on whitespace and ASCII punctuation, and all resulting tokens must match (AND).
- Each token matches a whole token in the body: as typed, lowercase, UPPERCASE, or Titlecase. There is no substring or prefix matching.
- Consequences:
  - `user_id` becomes `user` AND `id`, which is broad. Search for the ID's value instead.
  - `timeout` does not match `TimeoutException` or `timeouts`. Search for the exact word that appears in the log.
  - An exception class such as `java.net.ConnectException` splits into `java`, `net`, and `ConnectException`, which still works.
- Not available over MCP: attribute filters, exclusion (`not contains`), and filtering by `trace_id`. To find all logs of one trace, use the Coroot UI ("Show logs" in the trace view).

### Output

- Response fields: `{source, from, to, limit, returned, truncated, hint, entries[]}`, newest first.
- Each entry: `service_name`, `timestamp`, `severity`, `severity_text`, `body`, `trace_id`, `log_attributes`, `resource_attributes`, `cluster_id`.
- Responses are capped at about 50 KB. The newest entries are kept and `truncated` is set to `true`. When truncated, narrow the window, add `severity` or `search`, or lower `max_body_length`; do not report counts from a truncated page.
- Requires permission to view logs and a configured ClickHouse.

## Local wrapper tools

- `coroot_get_app_logs(app_id='ID', severity='Error')` — application logs.
- `coroot_get_overview_logs(severity='Error', limit=50)` — project-wide logs.

Follow the connected wrapper schema; do not assume the wrappers accept `search`, `log_pattern`, or time-range arguments.

## Log volume from metrics

Pattern counts are also exported as the counter `container_log_messages_total` (by severity level and pattern hash). It covers all messages, including those not shipped to ClickHouse during sampling, so prefer it for volume and trends. Confirm the label names with `list_metric_names` or a raw query first, then for example:

```promql
sum by (level) (increase(container_log_messages_total{container_id=~"VERIFIED_SELECTOR"}[5m]))
```

Replace `VERIFIED_SELECTOR` and the label names with values confirmed in the connected environment. Never send a placeholder as executable PromQL.

## Coroot UI query syntax (to guide the user)

- Filters on severity, message, and attributes such as `http.response.status`.
- Attribute operators: `=`, `~` (regex), `!=`, `!~`. Positive filters on the same attribute are OR'd; negative filters are AND'd.
- Message "contains" is a token search backed by ClickHouse full-text indexes.
- "Patterns" mode with "Show Messages" drill-down works even without ClickHouse, because pattern counts come from metrics.
- The UI shows 100 entries by default, up to 10,000.
