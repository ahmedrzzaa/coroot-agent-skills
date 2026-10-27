# How Coroot collects and processes logs

Read this when logs look incomplete, severities look wrong, or patterns are missing. Sources: https://docs.coroot.com/logs/overview, https://docs.coroot.com/configuration/coroot-node-agent, https://docs.coroot.com/configuration/clickhouse, https://docs.coroot.com/configuration/configuration, https://docs.coroot.com/alerting/alerts, https://docs.coroot.com/inspections/logs, and the coroot, coroot-node-agent, and logparser repositories on GitHub (read 2026-10-21). Items marked "from code" are not stated in the docs.

## Collection

- The node agent discovers logs from every container on the node automatically and ships them over OTLP/HTTP.
- Sources: container stdout/stderr (Docker json-file, containerd CRI, CRI-O), journald, and `/var/log/*`. Each record carries `log.source`: `journald`, `stdout/stderr`, or the file path.
- Applications can also send logs over OpenTelemetry (OTLP), directly or through an OTel Collector. These are "otel" logs; node-agent logs are "agent" logs.
- The Windows agent collects the Windows Event Log and container logs.
- Opt-outs: `COROOT_LOG_MONITORING=disabled` on a container; `--disable-log-parsing` or `--disable-json-log-parsing` on the agent. An opted-out container has no logs in Coroot.

## Storage and retention

- Logs are stored in ClickHouse.
- `--logs-ttl` defaults to 7 days. It applies when the table is created and does not change existing tables.
- A space manager drops the oldest partitions (about one day each) once disk usage exceeds 70%. Actual retention can therefore be shorter than the TTL. Check before concluding that older logs never existed.

## Severity detection

- **JSON logs:** message and severity come from well-known fields; other fields become searchable attributes.
- **Plain-text logs:** the agent scans the first 255 characters and first 7 fields for glog prefixes or level words (`err`, `warn`, `fatal`, and similar).
- **journald:** priorities 0–2 become critical, 3 error, 4 warning, 5–6 info, 7 debug.
- **Normalised levels:** `unknown`, `trace`, `debug`, `info`, `warning`, `error`, `fatal`; "critical" maps to `fatal`.
- An unstructured log with no level keyword is `unknown`. Real errors can hide there, so include `unknown` when an error is suspected but not found.

## Patterns

- Built on the node from the message text: quoted strings, bracketed text, numbers, hex values, and UUIDs are stripped. Patterns differing by at most one word merge. Each pattern has an MD5 `pattern_hash`.
- **Only warning and above get patterns.** Info, debug, and unknown messages have no pattern.
- Caps:
  - 256 patterns per container per level. Further patterns are counted as "unclassified pattern (pattern limit reached)".
  - Pattern extraction handles at most 100 messages per second per container. The rest are counted as "event was sampled".
  - From code: messages over that rate are counted but not shipped to ClickHouse. During error storms, raw lines are a sample while the metric counts stay complete.
- Multi-line messages such as stack traces are joined, up to 64 KB.

## Linking logs to traces and applications

- Logs sent with an OpenTelemetry SDK carry trace context and link to their trace.
- From code: the agent also extracts trace IDs from JSON fields such as `trace_id`, `traceid`, `trace.id`, or `@tr`.
- Agent logs map to applications through their containers. OTel logs map through the configured or detected OTel service name. A misnamed OTel service can make an app's OTel logs look empty.

## Inspections and alerts

- **Logs inspection:** the Errors check flags an application when ERROR or CRITICAL messages exceed the threshold (default 0).
- **Built-in "Log errors" alert** (`new-log-patterns`): fires on new error or fatal patterns once a pattern reaches 10 occurrences. "Keep firing for" (5 minutes) also sets the counting window. At most 20 alerts per application.
- Noisy patterns can be suppressed. In Enterprise Edition, an LLM can judge new patterns and suppress noise automatically.
- `investigate_anomaly` (Enterprise) includes log spikes and patterns among candidate causes.
