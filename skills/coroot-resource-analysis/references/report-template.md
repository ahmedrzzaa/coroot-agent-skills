# System / deployment analysis report

**Environment and deployment**
- Project, application, deployment, and time window.

**Resource allocation**
- CPU requests and limits.
- Memory requests and limits.
- Replica count and node capacity, when available.

**Actual resource consumption**
- CPU usage and throttling.
- Memory usage and peak.
- OOM events and restarts.
- Process/thread evidence, if available.

**Performance impact**
- Latency percentiles and error rates.
- Relevant traces, logs, and dependencies.
- Workload volume and time-window comparison.

**Diagnosis**
- Confirmed findings.
- Most likely bottleneck.
- Confidence and alternative explanations.

**Recommendations**
- Resource or configuration adjustments justified by evidence.
- Additional instrumentation required.
- Verification steps and rollback criteria, where applicable.

**Data gaps**
- Missing resource specifications.
- Missing thread-level telemetry.
- Missing or truncated metrics.
- Unverified assumptions.
