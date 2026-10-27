# Coroot setup guides (summaries)

Full guides: https://docs.coroot.com/guides/ (read 2026-10-27). Follow the linked page for exact manifests; these summaries help choose and sequence the steps.

## ArgoCD (GitOps install of Coroot)

App-of-apps: `apps/coroot-operator.yaml` installs the `coroot-operator` chart from `https://coroot.github.io/helm-charts` (`CreateNamespace=true`); `apps/coroot.yaml` deploys the Coroot custom resource from `coroot/`. Option A: single instance with SQLite. Option B: HA with `replicas: 2`, `postgres`, `storeMetricsInClickhouse: true`, ClickHouse 2×2. Both use an auto-generated `coroot-api-key` Secret. Bootstrap with `kubectl apply -f bootstrap/coroot-root.yaml`; a CRD-ordering error resolves on ArgoCD retry. Store the EE license in Secret `coroot-license`.

## AWS EKS + RDS / ElastiCache

1. Create IAM policy `CorootMonitoringReadOnly` (permissions in [integrations.md](integrations.md)).
2. Create role `coroot-cluster-agent` trusted by `pods.eks.amazonaws.com`; associate it with `aws eks create-pod-identity-association` to service account `coroot-cluster-agent`. Restart the agent; `AWS_CONTAINER_CREDENTIALS_FULL_URI` confirms it. (IRSA alternative documented.)
3. Enable RDS Enhanced Monitoring (`--monitoring-interval 60`).
4. Add a credentials Secret and `clusterAgent.databases` entries (`rds:`, `elasticache:`, `sslmode: require`).

The agent log line `credentials=` shows which credential source is used.

## GKE + Cloud SQL / Memorystore

Requires GKE Standard with Workload Identity Federation. Grant the six viewer roles to `principal://.../subject/ns/${NAMESPACE}/sa/coroot-cluster-agent`; create a `coroot` database user with `gcloud sql users create`; add the Secret and `cloudsql:` / `memorystore:` entries. A 403 during the first minute is normal while identity propagates.

## OCI OKE + MySQL HeatWave / PostgreSQL / OCI Cache

Requires an enhanced OKE cluster. Create an IAM policy for the workload principal; create the MySQL user with `GRANT PROCESS, REPLICATION CLIENT, SELECT`; enable `pg_stat_statements` on PostgreSQL through an OCI configuration; add `ocidb:` / `ocicache:` entries. Service logs are optional.

## ClickHouse on S3

Dedicated bucket (not shared, no lifecycle policies) → credentials Secret (or IRSA/workload identity) → `clickhouse.s3` with `mode: tiered` or `s3only` → apply. Each replica gets its own S3 path; zero-copy replication is off. Verify with `system.disks` and `system.storage_policies`.

## JFrog Artifactory (private registry)

Create a remote Docker repository proxying `https://ghcr.io`; create a `kubernetes.io/dockerconfigjson` Secret `coroot-registry-auth`; install the operator with `--set registry.url=https://example.jfrog.io/coroot --set registry.pullSecret=coroot-registry-auth` (add `registry.tlsSkipVerify=true` only for self-signed certificates); apply a custom resource with `spec: {}`.

## Multi-cluster

Clusters A and B each run Coroot (NodePort 30001) with an extra `multi-cluster-api-key`. Cluster C stores those keys in Secret `remote-coroot-clusters`, defines `remoteCoroot` projects, and a `memberProjects` aggregate (for example `prod-global`). Data stays in each cluster; C queries them remotely.

## Shared database across projects

Requires Coroot 1.27.1+, Operator 1.10.6+, cluster agent 1.11.5+. Cluster 1 runs Coroot with projects `app1`, `app2`, `shared-db`; cluster 2 runs `agentsOnly`. The database host runs the node agent (install.sh with `API_KEY`). A second custom resource for `shared-db` sets `nodeAgent.enabled: false`, `clusterAgent.kubernetes.enabled: false`, and lists the MySQL host under `databases`. Multi-cluster projects `app1-with-db` and `app2-with-db` combine each application with the database.
