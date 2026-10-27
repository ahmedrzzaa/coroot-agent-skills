# Trace analysis patterns

## Critical path

The critical path is the chain of spans that determines end-to-end duration. Optimising work off the critical path does not reduce latency. Dapper was used to "identify unnecessary serial requests along the critical path", and engineers "routinely guess wrong about the root cause" without it (https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/). With concurrent children, only the longest child is on the critical path.

## Self time versus child time

- High `self` on a frame: time spent inside that process (CPU, locks, GC, serialization). Follow up with CPU and runtime evidence (`coroot-resource-analysis`) or the Coroot Profiling tab.
- A frame wide because of its children: time spent waiting on downstream calls. Descend into the widest child.
- Coroot splits overlapping child time equally between children, so concurrent children look smaller than their wall time.

## N+1 queries

Sequential, non-overlapping database spans with similar statements, typically more than about 5 spans totalling over 50ms (https://docs.sentry.io/product/issues/issue-details/performance-issues/n-one-queries/). In `get_trace`, look for repeated `db.statement` values under one parent with back-to-back timestamps. Fix: batch or join.

## Fan-out and the tail

When a request fans out to many backends, the slowest backend sets the latency. If each server is slow 1% of the time, a request touching 100 servers is slow 63% of the time (Dean and Barroso, "The Tail at Scale", CACM 2013). Look for many parallel child spans where one outlier dominates.

## Retries and timeouts

Retries multiply across layers (the product of attempts at each layer) and can turn a slow dependency into an overload; deadlines should propagate (https://sre.google/sre-book/addressing-cascading-failures/). OpenTelemetry creates one span per resend, tagged `http.request.resend_count` (https://opentelemetry.io/docs/specs/semconv/http/http-spans/). Look for repeated identical client spans separated by a fixed delay, and spans ending at round durations (a timeout).

## Error origin versus propagation

The deepest failing span is the origin; its ancestors only propagate the error. `traces_errors` already removes propagating ancestors. Note that OTel server spans leave 4xx Unset while client spans mark 4xx as Error (OTel HTTP semantic conventions), so a 4xx can appear as an error on the caller and not on the callee.

## Compare slow with normal

A slow trace on its own shows where time went, not what changed. Compare the slow band with a normal band (two `traces_outliers` calls) and look for frames whose share of the total grew, new frames, or attribute values that appear only in the slow band (for example one database host, one tenant, one route).

## Sampling bias

Head sampling decides before the outcome is known, so rare errors can be missed; tail sampling keeps errors and slow traces and therefore over-represents them (https://opentelemetry.io/docs/concepts/sampling/). Coroot's error and outlier tools also cap at the 200 most recent traces. Use metrics for rates and ratios; use traces for explanations and examples. Clock skew between hosts can make child spans appear to start before their parents.
