# Managed cloud databases in Coroot

Sources: https://docs.coroot.com/configuration/aws, https://docs.coroot.com/configuration/gcp, https://docs.coroot.com/configuration/oci, and the guides aws-eks-rds, gcp-gke-cloudsql, oci-oke-mysql, shared-database (read 2026-10-27).

Discovery runs in the cluster agent once a minute. Query-level internals still need database credentials; replicas reuse the primary's credentials.

## AWS RDS, Aurora, ElastiCache

- Targets declared as `rds:` and `elasticache:`; RDS Postgres 15+ needs `sslmode: require`.
- `aws_rds_*` CPU, I/O, and filesystem metrics exist **only with Enhanced Monitoring** enabled.
- RDS logs arrive as `aws_rds_log_messages_total{level,pattern_hash,sample}` under service `/aws/rds/<region>/<id>`. ElastiCache logs are not collected.
- MySQL on RDS needs `performance_schema=1` (Performance Insights) for top queries and locks.
- The `rdsadmin` database is excluded. Discovery problems appear as `aws_discovery_error`.
- Status: `aws_rds_status`.

## GCP Cloud SQL and Memorystore

- Configured as code only (`cloudsql:`, `memorystore:`).
- Metrics `gcp_cloudsql_*`, `gcp_memorystore_*`; status `gcp_cloudsql_status`. Logs under `/gcp/cloudsql/<project>/<instance>`.
- A MySQL user created with gcloud has host `''`, so grants must target `'coroot'@''`.
- `cloudsqladmin` is excluded. A 403 during the first minute after setup is normal.

## OCI MySQL HeatWave, PostgreSQL, OCI Cache

- Targets `ocidb:` and `ocicache:`. Status `oci_db_status`.
- Postgres `pg_stat_statements` requires `oci.admin_enabled_extensions` (changing it restarts the database).
- OCI Cache uses TLS `skip-verify`. The HeatWave error log is read from `performance_schema.error_log`. `oci_db_io_latency_seconds` exists for Postgres only.

## Uninstrumented and shared databases

- A database Coroot has no credentials for appears as an **external service** with client-side data only. Its server-side health is unknown, not healthy.
- A database shared by several projects can live in its own project and be joined to application projects through a multi-cluster project (guide: shared-database).
- MCP node IDs for managed databases look like `cluster:rds:db1`.
