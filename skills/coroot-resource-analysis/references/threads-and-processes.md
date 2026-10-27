# Thread and process-level diagnostics

Thread analysis is distinct from container-level CPU analysis. A container may have low average CPU consumption while a single critical thread is saturated. Conversely, a process can have many idle threads without requiring additional CPU allocation.

## What to investigate

When supported telemetry is available, inspect:

- Process CPU usage.
- Process thread count.
- Per-thread CPU usage.
- Runnable threads and scheduling delay.
- Thread-pool size and queue depth.
- Blocked or waiting threads.
- Lock contention.
- Garbage collection and runtime pauses.
- CPU affinity and available processors.
- Context switches and other scheduler indicators.
- Thread or process creation failures.

These details may require application profiling, runtime metrics, a supported process-inspection tool, or host-level diagnostics beyond the exposed Coroot tools.

Do not assume the MCP integration exposes `top -H`, `/proc`, thread dumps, per-thread scheduler statistics, or language-runtime internals.

`container_resources_cpu_delay_seconds_total` (Coroot node agent) measures time a container's tasks spent waiting for a CPU. It is a container-level signal for scheduling delay; it does not attribute delay to individual threads.

## Workflow

1. Establish whether the problem is process-wide or isolated to a particular operation.
2. Inspect container CPU usage and throttling (see [cpu.md](cpu.md)).
3. Determine whether process/thread metrics are available.
4. If available, compare thread count and per-thread CPU usage over the relevant interval.
5. Examine runtime or application instrumentation for blocked threads, lock contention, queue buildup, or garbage collection.
6. Correlate the findings with traces and request latency.
7. Identify whether the bottleneck is CPU execution, synchronization, blocking I/O, or another resource.

## Interpreting thread symptoms

- One saturated thread may indicate a serial execution bottleneck even when total CPU usage is modest.
- Many runnable threads may indicate CPU contention or excessive concurrency.
- Many waiting threads may indicate blocking I/O, lock contention, or an undersized downstream resource.
- High thread count alone does not prove a leak.
- High context-switch rates alone do not establish the root cause.
- Low CPU usage alongside high latency may indicate waiting, locking, network delays, or database latency rather than inadequate CPU allocation.

If per-thread telemetry is unavailable, report that limitation and use process-level metrics, traces, logs, and application instrumentation to narrow the investigation.
