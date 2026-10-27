# Configuring Coroot

Sources: https://docs.coroot.com/configuration/ (configuration, prometheus, clickhouse, high-availability, coroot-node-agent, coroot-cluster-agent), read 2026-10-27.

## Coroot server

Precedence: flags, then environment variables, then the config file. `projects` can only be defined in the config file.

| Flag | Env | Default |
|---|---|---|
| `--listen` | `LISTEN` | `0.0.0.0:8080` |
| `--grpc-listen` | `GRPC_LISTEN` | `:4317` |
| `--https-listen`, `--tls-cert-file`, `--tls-key-file` | `HTTPS_LISTEN`, `TLS_*` | — |
| `--url-base-path` | `URL_BASE_PATH` | `/` |
| `--data-dir` | `DATA_DIR` | `/data` |
| `--cache-ttl` | `CACHE_TTL` | `30d` |
| `--traces-ttl`, `--logs-ttl`, `--profiles-ttl`, `--metrics-ttl` | `*_TTL` | `7d` |
| `--pg-connection-string` | `PG_CONNECTION_STRING` | SQLite when unset |
| `--auth-anonymous-role` | `AUTH_ANONYMOUS_ROLE` | — (Admin, Editor, or Viewer) |
| `--auth-bootstrap-admin-password` | `AUTH_BOOTSTRAP_ADMIN_PASSWORD` | — |
| `--license-key` | `LICENSE_KEY` | — |
| `--global-prometheus-url` (+ `-user`, `-password`, `-custom-headers`, `-remote-write-url`, `-tls-skip-verify`) | `GLOBAL_PROMETHEUS_*` | — |
| `--global-refresh-interval` | `GLOBAL_REFRESH_INTERVAL` | `15s` |
| `--global-prometheus-use-clickhouse` | `GLOBAL_PROMETHEUS_USE_CLICKHOUSE` | false |
| `--global-clickhouse-address` (+ `-user`, `-password`, `-initial-database`, `-tls-enabled`, `-tls-skip-verify`, `-tls-ca-file`) | `GLOBAL_CLICKHOUSE_*` | — |
| `--disable-clickhouse-space-manager` | `CLICKHOUSE_SPACE_MANAGER_DISABLED` | false (cleans up at 70% disk) |
| `--disable-builtin-alerts` | `DISABLE_BUILTIN_ALERTS` | false |

- TTLs apply only when tables are created. To change retention later, run `ALTER TABLE ... MODIFY TTL` in ClickHouse.
- The config file also holds `auth.serviceAccounts`, `projects[]` (apiKeys, notificationIntegrations, applicationCategories, alertingRules, inspectionOverrides, remoteCoroot, memberProjects), `sso`, `ai`, and `corootCloud`.

## Node agent

- Connection: `--collector-endpoint` (`COLLECTOR_ENDPOINT`), `--api-key` (`API_KEY`), `--scrape-interval` (15s), `--cgroupfs-root` (`/sys/fs/cgroup`).
- Data: `--traces-sampling` (1.0), `--disable-log-parsing`, `--disable-l7-tracing`, `--enable-java-tls`, `--enable-java-async-profiler`.
- Scope: `--container-allowlist`, `--container-denylist`, `--track-public-network`.
- Buffering: `--wal-dir` (`/tmp/coroot-node-agent`), `--max-spool-size` (500MB).
- Per-container opt-outs (container environment): `COROOT_EBPF_PROFILING=disabled`, `COROOT_LOG_MONITORING=disabled`, `COROOT_EBPF_TRACES=disabled`.

## Cluster agent

- `--coroot-url` (required), `--api-key`, `--config-file` (static YAML; `${VAR}` references are expanded).
- `--disable-kubernetes` turns off collection from its own cluster.
- `--collect-aws-logs`, `--collect-gcp-logs`, `--collect-oci-logs` (default true).
- `--track-database-changes`, `--track-database-sizes`, `--track-database-bloat`, `--max-tables-per-database` (1000), `--exclude-databases`.

## External Prometheus

- Set in Project Settings → Prometheus, with the global flags, or `externalPrometheus` in the custom resource.
- Enable the remote-write receiver (`--web.enable-remote-write-receiver`).
- VictoriaMetrics cluster: `GLOBAL_PROMETHEUS_REMOTE_WRITE_URL` → vminsert, `GLOBAL_PROMETHEUS_URL` → vmselect.
- Coroot keeps its own metric cache, so Prometheus retention can be a few hours.

## External ClickHouse

- Set in Project Settings → Clickhouse, `global_clickhouse`, or `externalClickhouse` in the custom resource.
- Coroot picks the ClickHouse cluster named `coroot`, else `default` (`SHOW CLUSTERS`).
- ClickHouse Cloud: port 9440 with TLS enabled; the space manager turns off automatically.

## High availability

- Run several Coroot instances with PostgreSQL as the configuration database (`postgres` and `replicas` in the custom resource).
- Leader election uses PostgreSQL advisory locks for the per-minute SLO and deployment checks. Each instance keeps its own metric cache.

## ClickHouse S3 tiering

- Custom resource `clickhouse.s3`: `endpoint`, `region`, `credentials.{accessKeyId,secretAccessKey}` (Secret references), `cacheSize` (10Gi), `mode` (`tiered` or `s3only`), `moveFactor` ("0.1").
- Credentials Secret: `kubectl create secret generic clickhouse-s3-creds --from-literal=access_key_id=... --from-literal=secret_access_key=... -n coroot` (have the user run it so keys never enter the conversation). Omit credentials with IRSA or workload identity.
- Use a dedicated bucket without lifecycle policies. The space manager is disabled with S3.
