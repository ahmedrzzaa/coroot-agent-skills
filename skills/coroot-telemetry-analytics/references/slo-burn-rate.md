# SLOs, error budgets, and burn rates

## Coroot SLO definitions

Sources: https://docs.coroot.com/inspections/slo, https://docs.coroot.com/alerting/slo-monitoring, github.com/coroot/coroot (`model/check.go`, `constructor/sli.go`, `model/alert.go`, `watchers/incidents.go`, 2026-10-21).

| SLO | Default objective |
|---|---|
| Availability | 99% of requests succeed |
| Latency | 99% of requests faster than 500ms (5s for `control-plane` and `monitoring` apps) |
| LLM latency | 99% faster than 10s (time to first byte) |

- External services have SLOs disabled (objective 0).
- Annotations: `coroot.com/slo-availability-objective`, `coroot.com/slo-latency-objective`, `coroot.com/slo-latency-threshold`. Setting an objective to 0% disables that SLO. The latency threshold is rounded **up** to the next histogram bucket.
- Custom SLIs can be defined with PromQL in the UI. Coroot wraps them as `sum(rate(<query>[$RANGE]))` for availability and `sum by (le) (rate(<query>[$RANGE]))` for latency.

### Where the SLI data comes from

- Coroot uses the application's inbound (server-side) metrics when they exist.
- Otherwise it sums the outbound (client-side) metrics of its callers, excluding calls from auxiliary apps (monitoring, control-plane) into normal apps.
- **Latency SLI:** slow ÷ total, where total is the `+Inf` bucket and slow is total minus the bucket at or below the threshold.
- **Failed request:** status `failed`, 5xx, or a non-OK gRPC code. 4xx is not a failure.

### When Coroot opens an incident

- Burn rate = (bad ÷ total) ÷ (1 − objective/100).
- Two CRITICAL rules: burn rate above **14.4** over 1h **and** 5m, or above **6** over 6h **and** 15m.
- A window counts only when at least half of it has data.
- The incident resolves when no rule exceeds its threshold, or when alerting is disabled for the app.
- There is no lower-severity (ticket) rule.
- With the default 99% objective, this means a bad-request ratio above 14.4% over both 1h and 5m, or above 6% over both 6h and 15m.

## General SLO maths

Source: https://sre.google/workbook/alerting-on-slos/ and https://sre.google/workbook/implementing-slos/.

- **Error budget** = 100% − SLO. A 99% SLO allows 1% bad requests.
- **Burn rate** = how fast the budget is consumed relative to the SLO. A burn rate of 1 uses exactly the whole budget over the SLO period.
- **Budget consumed** = burn rate × alert window ÷ SLO period.
- The short window is usually 1/12 of the long window; it makes alerts reset quickly after recovery.

Google's recommended starting points for a 99.9% SLO over 30 days:

| Severity | Long window | Short window | Burn rate | Budget consumed |
|---|---|---|---|---|
| Page | 1h | 5m | 14.4 | 2% |
| Page | 6h | 30m | 6 | 5% |
| Ticket | 3d | 6h | 1 | 10% |

Coroot follows the first two rows but uses a 15m short window for the 6h rule and has no ticket rule. Coroot does not state its SLO period, so "% of budget consumed" for Coroot cannot be derived exactly.

## Reproducing Coroot's burn rate

Approximate 1h availability burn rate for an application with a 99% objective:

```promql
(
  sum(increase(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL",status=~"5.*|failed|grpc:.*",status!="grpc:OK"}[1h]))
  / sum(increase(container_http_inbound_requests_total{app_id="VERIFIED_APP_LABEL"}[1h]))
) / (1 - 0.99)
```

Repeat with `[5m]` for the short window. Both must exceed 14.4 for Coroot's first rule.

Expect differences from Coroot's own figure when the app uses a custom SLI, when Coroot falls back to client-side metrics, or because Coroot computes from its internal cache. When numbers disagree, report Coroot's incident data as authoritative and the reproduction as a cross-check.
