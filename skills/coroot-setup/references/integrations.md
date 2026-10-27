# Coroot integrations

Sources: https://docs.coroot.com/configuration/{aws,gcp,oci}, https://docs.coroot.com/alerting/{slack,teams,pagerduty,opsgenie,webhook}, https://docs.coroot.com/metrics/custom-metrics, https://docs.coroot.com/dashboards/overview, and github.com/coroot/coroot source where noted (read 2026-10-27).

## Managed databases and cloud integrations

All three run in the cluster agent and poll once a minute. Query-level internals (queries, locks, replication) also need credentials for a monitoring database user, set in the custom resource `clusterAgent.databases` or the UI.

| | AWS | GCP | OCI |
|---|---|---|---|
| Discovers | RDS (incl. Aurora), ElastiCache | Cloud SQL (Postgres, MySQL, SQL Server), Memorystore (Redis, Memcached, Valkey) | MySQL HeatWave, PostgreSQL DB systems, OCI Cache |
| Configure via | UI (Project Settings → Cloud integrations) or custom resource `clusterAgent.aws` | Config as code only | Config as code only |
| Permissions | `rds:DescribeDBInstances`, `rds:DescribeDBLogFiles`, `rds:DownloadDBLogFilePortion`, `rds:ListTagsForResource`, `elasticache:DescribeCacheClusters`, `elasticache:ListTagsForResource`, `logs:GetLogEvents` on `RDSOSMetrics` | `roles/cloudsql.viewer`, `redis.viewer`, `memcache.viewer`, `memorystore.viewer`, `monitoring.viewer`, `logging.viewer` | Read `mysql-family`, `postgres-db-systems`, `redis-family`, `metrics`, `log-content`; inspect `log-groups` |
| Credentials | EKS Pod Identity / IRSA, EC2 instance profile, or static key | Application default credentials (GKE Workload Identity) or `credentialsSecret` | OKE Workload Identity, instance principal, or `apiKeySecret` |
| Collects | Inventory, Enhanced Monitoring OS metrics (granularity 60s or finer), RDS logs, costs | Inventory, Cloud Monitoring metrics, Cloud SQL logs (about $60/month for 10 instances) | Inventory, OCI Monitoring metrics, service logs, MySQL error log |
| Filters | `rdsTagFilters`, `elasticacheTagFilters` | `cloudsqlLabelFilters`, `memorystoreLabelFilters` | `dbTagFilters`, `cacheTagFilters` |

Prefer workload identity over static keys.

## Notifications

| Integration | Provider side | Coroot side |
|---|---|---|
| Slack | In Coroot click **Create Slack app** (sends a manifest), install to the workspace, copy the **Bot User OAuth Token** | Paste the token, set a default public channel, send a test |
| Microsoft Teams | Channel → … → Workflows → "Post to a channel when a webhook request is received" → copy the URL | Project Settings → Integrations: add channels (one URL each) and a default |
| PagerDuty | Service → Integrations → **Events API V2** → copy the Integration Key | Paste the key |
| Opsgenie | Team → Integrations → **API** with "Create and Update Access" → copy the API key | Paste the key (EU instance option available) |
| Webhook | — | URL, basic auth, headers, custom fields, Go templates per event type |

Routing is per application category (`applicationCategories[].notificationSettings`), with separate settings for incidents, deployments, and alerts.

### Webhook template data (Go `text/template`)

| Event | Fields |
|---|---|
| Incident | `.Status` (OK, WARNING, CRITICAL), `.Application.{Namespace,Kind,Name}`, `.Reports[].{Name,Check,Message}`, `.URL`, `.RCASummary`, `.RCARemediations` |
| Deployment | `.Status` (Deployed, Cancelled, Stuck), `.Application`, `.Version`, `.Summary[]`, `.URL` |
| Alert | `.Status`, `.ProjectName`, `.Application`, `.RuleName`, `.Severity`, `.Summary`, `.Details[].{Name,Value}`, `.Duration`, `.ResolvedBy`, `.URL` |

- `{{ json . }}` emits JSON with snake_case keys (`rca_summary`, `rule_name`).
- Custom fields are capitalised in templates (`environment` → `{{ .Environment }}`); built-in fields win on name clashes.
- The webhook page includes sample templates for Telegram, Mattermost, and Discord.
- As code: `projects[].notificationIntegrations` (`baseURL` required).

## OpenTelemetry (traces and logs beyond eBPF)

- Coroot accepts OTLP over HTTP at `/v1/traces`, `/v1/logs`, `/v1/metrics`, `/v1/profiles` on the HTTP port, and OTLP gRPC on `:4317`.
- Authentication header: `x-api-key: <project API key>` (from source; the tracing docs' examples omit it). A missing or wrong key returns "project not found".
- SDK environment, as shown by Coroot's in-app dialog:

```bash
OTEL_SERVICE_NAME="<service>" \
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="<coroot_url>/v1/traces" \
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT="<coroot_url>/v1/logs" \
OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf" \
OTEL_METRICS_EXPORTER="none" \
OTEL_EXPORTER_OTLP_HEADERS="x-api-key=<api_key>"
```

- Existing OpenTelemetry Collector: exporter `otlphttp/coroot` with `endpoint: "<coroot_url>"`, `encoding: proto`, `headers: {"x-api-key": "<api_key>"}`, added to both the traces and logs pipelines.
- Language guides: https://docs.coroot.com/tracing/opentelemetry-go, -java, -python. Logs emitted with trace context link to their traces.

## Custom metrics

The cluster agent scrapes pods annotated (Kubernetes service discovery only):

```yaml
coroot.com/scrape-metrics: 'true'
coroot.com/metrics-port: '8080'
coroot.com/metrics-path: '/metrics'   # optional
coroot.com/metrics-scheme: 'http'     # optional
```

`pod` and `namespace` labels are added. Go pprof profiling uses `coroot.com/profile-scrape` and `coroot.com/profile-port`.

## Custom dashboards

1. Dashboards → **Add dashboard** → name → Save.
2. **Add panel**: name, group, PromQL query, legend with `{{ label_name }}` → Apply.
3. Drag to resize or move; Save. Groups can be collapsed and reordered.

Only time-series charts are available. In Community Edition, Admins and Editors edit and Viewers read; Enterprise adds per-dashboard permissions.
