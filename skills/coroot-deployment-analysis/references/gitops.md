# GitOps (ArgoCD and FluxCD) signals in Coroot

Sources: https://docs.coroot.com/gitops/overview, https://docs.coroot.com/metrics/cluster-agent, github.com/coroot/coroot `api/views/overview/argocd.go`, `fluxcd.go`, `watchers/deployments.go`, coroot-cluster-agent `metrics/ksm/` (read 2026-10-27). "(inferred)" marks reasoning not stated in the sources.

There is no MCP tool for GitOps state; use `query_metrics`. Data comes from the cluster agent (v1.5.0+ for Flux, v1.7.0+ for Argo) with the read permissions granted by a current Coroot Operator. It is absent if the agent runs with `--disable-kubernetes`.

## GitOps versus Coroot deployment tracking

| Signal | Shows | Covers |
|---|---|---|
| Coroot deployment tracking | What actually rolled out (new ReplicaSet = new version; Stuck after 3 minutes; comparison 30 minutes later) | Deployment, InferenceService, DynamoGraphDeployment only |
| GitOps metrics | What was intended and whether reconciliation succeeded | Any resource the GitOps tool manages |

Reading combinations (inferred):

- **Sync without a rollout:** a config-only change (ConfigMap, Secret, Service, HPA, CRD) or a StatefulSet, DaemonSet, or CronJob update, none of which create a ReplicaSet. Coroot's deployment view will not show it.
- **Rollout without a sync:** an out-of-band manual change; expect Argo OutOfSync, an Argo self-heal sync, or a Flux `Drifted` condition.
- **Failed sync** (Argo `Failed`/`SyncFailed`, or Flux applied revision ≠ attempted revision): the new commit may not have landed, or landed partly.

## ArgoCD metrics

All carry `uid`, `name`, `namespace`. Info metrics have value 1.

| Metric | Labels |
|---|---|
| `argocd_application_info` | `project`, `source_type`, `repo`, `path`, `chart`, `target_revision`, `dest_server`, `dest_name`, `dest_namespace`, `revision` |
| `argocd_application_sync_status` | `sync_status`: Synced, OutOfSync, Unknown |
| `argocd_application_health_status` | `health_status`: Healthy, Progressing, Degraded, Suspended, Missing, Unknown |
| `argocd_application_operation_status` | `operation_phase`: Running, Succeeded, Failed, Error, Terminating |
| `argocd_application_operation_finished_timestamp_seconds` | value = Unix time the last sync finished |
| `argocd_application_resource_info` | `resource_group`, `resource_kind`, `resource_namespace`, `resource_name` |
| `argocd_application_resource_sync_status`, `_health_status`, `_status` | resource labels plus status (`_status`: Synced, Pruned, SyncFailed) |

Coroot flags an Argo app when sync is not Synced, health is not Healthy/Progressing/Suspended, or the operation phase is not Succeeded/Running/Terminating.

## FluxCD metrics

| Metric | Labels |
|---|---|
| `fluxcd_{git,oci,helm}_repository_info` | `url`, `interval`, `suspended` |
| `fluxcd_helm_release_info` | `suspended`, `interval`, `target_namespace`, `source_kind/name/namespace`, `chart`, `version` |
| `fluxcd_helm_chart_info` | `chart`, `version`, `source_*`, `interval`, `suspended` |
| `fluxcd_kustomization_info` | `suspended`, `interval`, `path`, `source_*`, `target_namespace`, `last_applied_revision`, `last_attempted_revision` |
| `fluxcd_resourceset_info` | `last_applied_revision` |
| `fluxcd_*_status` | `type`, `reason`; value 1 = True, 0 = False or Unknown |
| `fluxcd_{kustomization,resourceset}_inventory_entry_info` | `entry_id` = `<ns>_<name>_<group>_<Kind>` |
| `fluxcd_{kustomization,resourceset}_dependency_info` | `depends_on_name`, `depends_on_namespace` |

Condition `type` values: `Ready`, `Healthy`, `Stalled`, `Reconciling` (for the last two, 1 is the bad or in-progress state); HelmRelease adds `Released`, `TestSuccess`, `Remediated`, `Drifted`. Common `reason` values: ReconciliationSucceeded/Failed, Progressing, ProgressingWithRetry, DependencyNotReady, ArtifactFailed, BuildFailed, HealthCheckFailed, PruneFailed, Install/Upgrade/Rollback/Test Succeeded or Failed, DriftDetected. Because Unknown becomes 0, Flux resources that are still reconciling look failed; read `reason`.

## PromQL

Problem Argo apps (same rules as the UI):

```promql
argocd_application_sync_status{sync_status="OutOfSync"}
  or argocd_application_health_status{health_status=~"Degraded|Missing"}
  or argocd_application_operation_status{operation_phase=~"Failed|Error"}
```

Unhealthy resources of one Argo app:

```promql
argocd_application_resource_health_status{name="VERIFIED_APP",health_status!="Healthy"}
argocd_application_resource_status{name="VERIFIED_APP",status="SyncFailed"}
```

Flux resources not ready, stalled, or suspended:

```promql
{__name__=~"fluxcd_.*_status",type="Ready"} == 0
{__name__=~"fluxcd_.*_status",type="Stalled"} == 1
{__name__=~"fluxcd_.*_info",suspended="true"}
```

Revision changed within the last hour:

```promql
count by (namespace, name) (count by (namespace, name, revision) (last_over_time(argocd_application_info[1h]))) > 1
```

Flux applied versus attempted revision (a mismatch means the last apply failed):

```promql
group by (namespace, name, last_applied_revision, last_attempted_revision) (fluxcd_kustomization_info)
```

Recent Argo syncs:

```promql
time() - argocd_application_operation_finished_timestamp_seconds < 3600
```

## Correlating with an incident

1. Take the incident start from `list_incidents`.
2. Query `group by (namespace, name, revision) (argocd_application_info)` from 60 minutes before to 15 minutes after. Each revision is a separate series; the change point is the sync time, accurate to the scrape interval (inferred).
3. Map the affected workload to its GitOps owner:
   - Argo: `argocd_application_resource_info{resource_kind="Deployment",resource_namespace="NS",resource_name="NAME"}`
   - Flux: `fluxcd_kustomization_inventory_entry_info{entry_id="NS_NAME_apps_Deployment"}`
4. An Argo app whose `dest_server` is another cluster will not map to this project's applications.

Replace `VERIFIED_APP`, `NS`, and `NAME` with values confirmed in the connected environment.
