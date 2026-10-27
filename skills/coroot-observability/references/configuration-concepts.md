# Configuration concepts that change how to read Coroot data

Sources: https://docs.coroot.com/configuration/{application-categories,custom-applications,multi-cluster,multi-tenancy,rbac,authentication}, and github.com/coroot/coroot `model/application_category.go`, `model/application_id.go`, `model/kubernetes.go`, `constructor/`, `rbac/`, `api/mcp.go` (read 2026-10-27). "(inferred)" marks reasoning not stated in the sources.

## Application IDs and kinds

Format: `cluster_id:namespace:Kind:name`. Empty namespace or cluster becomes `_`. MCP rejects three-part IDs.

| Kind | How it arises |
|---|---|
| Deployment, StatefulSet, DaemonSet, CronJob, Job, Pod, StaticPods | Kubernetes owners (ReplicaSet with a hash suffix → Deployment; Job with a numeric suffix → CronJob; ownerless pods → StaticPods) |
| CustomApplication | Grouped by the `coroot.com/custom-application-name` annotation |
| Unknown | Non-Kubernetes containers or systemd units (namespace `_`, `.service` stripped; instance `<unit>@<node>`) |
| ExternalService | Destinations outside the cluster, e.g. `external:external:ExternalService:api.github.com:443` |
| DatabaseCluster | Grouped database components |
| RDS, ElasticacheCluster, CloudSQL, Memorystore, OCIDB, OCICache | Managed databases (namespace `_`) |
| DockerSwarmService, NomadJobGroup, ECSService | Other orchestrators |
| Workflow, SparkApplication, GitLabJob, InferenceService, DynamoGraphDeployment, Sandbox | Specialised workloads |

Custom applications defined by `instancePatterns` keep their kind and change only the name. An ExternalService is something Coroot sees only from the client side; it has no server-side health data.

## Application categories

| Category | Built-in examples (`<namespace>/<app>`) | Auxiliary |
|---|---|---|
| `application` | Default for everything unmatched | No |
| `control-plane` | `kube-system/*`, `istio*/*`, `argocd/*`, `flux-system/*`, `cert-manager/*`, `cilium/*`, `*/coredns`, `openshift*/*`, `_/systemd*` | Yes |
| `monitoring` | `monitoring/*`, `*/*prometheus*`, `*/*grafana*`, `coroot/*`, `loki/*`, `thanos/*`, `*/*datadog*` | Yes |
| Custom | User patterns or annotation | No |

Assignment order: annotation `coroot.com/application-category` (an unknown name creates the category), then user patterns (categories in alphabetical order), then built-in patterns, then `application`.

Effects:

- **SLIs:** requests from auxiliary apps to normal apps are excluded from the target's availability and latency SLIs.
- **SLO defaults:** latency threshold 500ms, relaxed to 5s for auxiliary apps; ExternalService SLOs are off.
- **Notifications:** each category routes incidents, deployments, and alerts separately. Deployment notifications are on by default only for `application`.
- **Alert rules** can target categories. `list_applications` returns a `category` field but cannot filter by it.

## Projects, clusters, and multi-cluster

- A project is one environment, region, or cluster; `cluster_id` in application IDs is the project ID.
- A multi-cluster project aggregates `memberProjects` (possibly remote Coroot instances) on demand. It ingests nothing itself. Application IDs keep the member's ID, so the same workload in two clusters is two applications. External services that match an app in another member become cross-cluster dependencies.
- In multi-tenant setups Coroot adds `{coroot_project_id="…"}` to queries automatically; do not add it yourself.
- **MCP with a multi-cluster project selected:** `query_metrics` and `list_metric_names` use the aggregate project's own (empty) Prometheus settings and do not fan out; project-wide `query_logs` fails with "ClickHouse is not configured". Select the member project for PromQL and project-wide logs. App-scoped `query_logs` routes correctly.

## RBAC and MCP

| Role | Can |
|---|---|
| Admin | Everything, including settings, integrations, users, roles |
| Editor | View everything; edit categories, custom applications, inspections, risks, dashboards, alerting rules, alerts |
| Viewer | View everything |
| Enterprise custom role | Scoped by project, application category, namespace, kind, name, node, dashboard |

| MCP tool | Permission |
|---|---|
| `list_applications`, `list_incidents` | Per-application view (**silently filtered**) |
| `get_application_status`, `get_incident_details` | Application view, otherwise "forbidden" |
| `list_alerts` | Alerts view, plus per-application filter |
| `resolve_alerts` | Alerts edit (Editor or Admin) |
| `list_nodes`, `get_node_details` | Node view |
| Trace tools | Traces view |
| `query_metrics`, `list_metric_names` | Metrics view |
| `query_logs` | Logs view, plus application view with `app_id` |

Service accounts use the same RBAC. Because filtering is silent, an empty or short list may mean the identity cannot see the data (inferred). When results look incomplete, ask the user which role the MCP connection uses.
