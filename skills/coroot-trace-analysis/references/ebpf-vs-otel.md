# eBPF spans versus OpenTelemetry traces

Sources: https://docs.coroot.com/tracing/overview, https://docs.coroot.com/tracing/ebpf-based-tracing, the OpenTelemetry Go/Java/Python and overhead pages under https://docs.coroot.com/tracing/, github.com/coroot/coroot-node-agent `tracing/tracing.go`, and github.com/coroot/coroot `api/views/overview/traces.go`, `collector/collector.go` (read 2026-10-27).

## Comparison

| | OpenTelemetry traces | eBPF spans |
|---|---|---|
| Needs code or agent changes | Yes (SDK or auto-instrumentation agent) | No |
| Shape | End-to-end tree across services via `traceparent` | One client-side span per outbound call; each is its own trace |
| Protocols | Whatever the SDK instruments | HTTP, Postgres, MySQL, Redis, MongoDB, Memcached (docs); also ClickHouse, Zookeeper, HTTP/2 and gRPC (agent code) |
| `service` | `service.name` (`OTEL_SERVICE_NAME`) | Calling workload, `/k8s/<namespace>/<workload>` (container ID outside Kubernetes) |
| Span name | Instrumentation-defined | HTTP method, `query`, or command name |
| Errors | Server spans leave 4xx Unset; client spans mark 4xx as Error | HTTP status ≥ 400 is an error; gRPC status > 0 |
| Durations | Server and client sides | Client side only, including network and queueing |
| Sampling | SDK sampler (`OTEL_TRACES_SAMPLER`) | Node agent `--traces-sampling` (0–1, default 1.0) |
| In Coroot UI Traces page | Yes | No (UI shows OTel traces only) |
| In MCP trace tools | Yes | Only when the project has no OTel traces at all (v1.27.1+) |

## Do not conclude from eBPF-only data

- Which inbound endpoint triggered a slow call: eBPF spans have no parents.
- Server-side processing time: durations are measured by the client.
- The server's identity from `service`: that is the caller; read `net.peer.name` or `http.url`.
- That eBPF and OTel error rates are comparable: eBPF counts 4xx; Coroot SLOs do not.
- Where an error originated: every span is its own root.
- Any call tree from `traces_outliers`: its flamegraph is one level deep.
- That a service has no traffic because it has no spans: if the project has any OTel traces, eBPF spans are never shown. Use `get_application_status` dependencies and request metrics.

## Adding OpenTelemetry

Coroot receives OTLP over HTTP at `<coroot>/v1/traces` and OTLP gRPC on `:4317`. Send the project API key as `x-api-key` (without it, data goes to the only project or to the project named `default`).

| Language | Minimum setup from the Coroot docs |
|---|---|
| Go | `go get` the `otlptracehttp` exporter and `otelhttp` (or otelmux/otelecho/otelgin); create a TracerProvider with a batcher and resource; set the TraceContext and Baggage propagator; wrap handlers. Env: `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://coroot.coroot:8080/v1/traces` |
| Java | Run with `java -javaagent:opentelemetry-javaagent.jar`. Env: `OTEL_SERVICE_NAME`, traces endpoint, `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=http/protobuf`, `OTEL_LOGS_EXPORTER=none`, `OTEL_METRICS_EXPORTER=none` |
| Python | `pip install opentelemetry-distro opentelemetry-exporter-otlp`, `opentelemetry-bootstrap -a install`. Env: `OTEL_RESOURCE_ATTRIBUTES=service.name=…`, traces endpoint, protocol `http/protobuf`. Run `opentelemetry-instrument --traces_exporter otlp --metrics_exporter none <app>` |

Full SDK environment and Collector exporter config are in the `coroot-setup` skill.

## Overhead (Coroot benchmark, 1,000 requests/s, 2 spans per request)

| | CPU at 100% sampling | CPU at 0% sampling | Memory | Bytes per span |
|---|---|---|---|---|
| Go, manual SDK | +50% | +20% | 13 → 15 MB | ~350 |
| Go, compile-time | +35% | +13% | — | — |
| Java agent | +30% | +11% | 310 → 459 MB | ~400 |
| Python | +160% | +74% | 201 → 304 MB | ~320 |

Python p99 rose from 2.0ms to 13.3ms; Java lost about 3% of spans at 100% sampling (raise `OTEL_BSP_MAX_QUEUE_SIZE`). Sampling lowers the cost but does not remove it. Benchmark sampler: `OTEL_TRACES_SAMPLER=parentbased_traceidratio` with `OTEL_TRACES_SAMPLER_ARG`.

## Storage

Traces are stored in ClickHouse (`otel_traces`, partitioned by day) with a per-minute histogram view. Retention: `--traces-ttl`, default 7 days, applied when tables are created.
