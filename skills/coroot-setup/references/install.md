# Installing Coroot

Source pages are under https://docs.coroot.com/installation/ (read from `docs/docs/installation/` in github.com/coroot/coroot on 2026-10-27).

## Requirements

- Linux kernel **4.16+**. RHEL/CentOS 7.6+ (kernel 3.10 with eBPF backport) works with limitations: no L7 metrics or eBPF tracing, no `container_net_latency_seconds`, no Python GIL or Node.js event-loop metrics, no eBPF profiling.
- eBPF profiling needs CO-RE: Ubuntu 20.10+, Debian 11+, RHEL 8.2+.
- Supported: Kubernetes (self-managed, EKS including basic Fargate, GKE, AKS, OKE), OpenShift, K3s, MicroK8s, Docker Swarm, ECS on EC2, systemd.
- Not supported: Docker-in-Docker (for example MiniKube), WSL1, GKE Autopilot (no privileged node agent), ECS Fargate.

## Measured overhead

| Test | Effect on workload | Agent cost |
|---|---|---|
| Node agent, Go server at 10,000 RPS (m5.2xlarge) | Latency change within measurement error | 200m CPU |
| MySQL 8.4, 10k tables, 10k QPS | No measurable latency change; mysqld +0.07 cores | Cluster agent ~0.01 cores, under 120MB |
| Postgres 18, 100 databases, 16k QPS | Below noise | ~0.05 cores, under 310MB |
| MongoDB 8.0, 10k collections | Below noise | ~0.015 cores, under 60MB |

Source: https://docs.coroot.com/installation/performance-impact

## Platform matrix

| Platform | Editions | UI access |
|---|---|---|
| Kubernetes (Operator + Helm) | CE chart `coroot/coroot-ce`; EE chart `coroot/coroot-ee` | `kubectl port-forward -n coroot service/coroot-coroot 8080:8080`, then http://localhost:8080 |
| OpenShift | **EE only**, OperatorHub | Configured Ingress host |
| Docker Compose | CE and EE | http://NODE_IP:8080 |
| Docker Swarm | CE and EE | http://NODE_IP:8080 |
| ECS on EC2 | Node agent only; Coroot runs elsewhere | — |
| Ubuntu/Debian, RHEL/CentOS | CE and EE | http://NODE_IP:8080 |
| Windows | Agent only; no Windows server | — |

Enterprise Edition needs a license key (free trial at coroot.com/account; pricing from $1 per CPU core per month). Keep it in a Secret or environment variable.

## Kubernetes (Coroot Operator)

```bash
helm repo add coroot https://coroot.github.io/helm-charts
helm repo update coroot
helm install -n coroot --create-namespace coroot-operator coroot/coroot-operator
helm install -n coroot coroot coroot/coroot-ce \
  --set "clickhouse.shards=2,clickhouse.replicas=2"
```

Enterprise: install `coroot/coroot-ee` and set `licenseKey` (prefer `enterpriseEdition.licenseKeySecret` in the custom resource over a plain `--set`).

- Upgrade: the operator upgrades components whose image is not pinned. Upgrade the operator with `helm upgrade -n coroot coroot-operator coroot/coroot-operator`.
- Uninstall: `helm uninstall coroot -n coroot`, then `helm uninstall coroot-operator -n coroot`.
- Pod Security admission blocking the agent (for example Talos): `kubectl label ns coroot pod-security.kubernetes.io/enforce=privileged`.
- Private registry: `--set registry.url=https://registry.example.com/coroot --set registry.pullSecret=coroot-registry-auth --set registry.tlsSkipVerify=false`.
- Useful custom-resource fields: `service` (port 8080, `grpcPort` 4317), `ingress` (className, host, path, tls), `storage.size` (default 10Gi), `agentsOnly.corootURL` with `apiKey`/`apiKeySecret`, `projects[]`, `serviceAccounts[]`, `enterpriseEdition.licenseKey`/`licenseKeySecret`.

Source: https://docs.coroot.com/installation/kubernetes, https://docs.coroot.com/installation/k8s-operator

## OpenShift (Enterprise)

Requires OpenShift 4.x, cluster-admin, an EE license, and access to the Red Hat catalog and coroot.com. Operators → OperatorHub → search "Coroot" → Install (Automatic approval recommended) → choose a project → Coroot tab → **Create Coroot** → set license, storage, ingress → check Workloads → Pods.

## Docker Compose

```bash
apt update
apt install docker-compose-v2
curl -fsS https://raw.githubusercontent.com/coroot/coroot/main/deploy/docker-compose.yaml | \
  docker compose -f - up -d
```

Runs Coroot, cluster agent, ClickHouse 24.3, node agent, and Prometheus 2.45. Upgrade: the same pipe with `docker compose -f - pull`, then `up -d`. Enterprise: set `LICENSE_KEY`.

## Docker Swarm

```bash
docker swarm init
curl -fsS https://raw.githubusercontent.com/coroot/coroot/main/deploy/docker-swarm-stack.yaml | \
  docker stack deploy -c - coroot
```

Swarm does not support privileged services, so run the node agent on every node:

```bash
docker run --detach --name coroot-node-agent \
  --pull=always \
  --privileged --pid host \
  -v /sys/kernel/tracing:/sys/kernel/tracing:rw \
  -v /sys/kernel/debug:/sys/kernel/debug:rw \
  -v /sys/fs/cgroup:/host/sys/fs/cgroup:ro \
  ghcr.io/coroot/coroot-node-agent \
  --cgroupfs-root=/host/sys/fs/cgroup \
  --collector-endpoint=http://NODE_IP:8080
```

Uninstall: `docker stack rm coroot`.

## ECS on EC2 (node agent)

```bash
curl -fsS https://raw.githubusercontent.com/coroot/coroot/main/deploy/ecs/coroot-node-agent.json | \
  sed -e 's|COROOT_URL|http://coroot.example.com:8080|' -e 's|API_KEY|YOUR_API_KEY|' > coroot-node-agent.json
aws ecs register-task-definition --cli-input-json file://coroot-node-agent.json
aws ecs create-service --cluster CLUSTER_NAME --service-name coroot-node-agent \
  --task-definition coroot-node-agent --launch-type EC2 --scheduling-strategy DAEMON
```

Do not commit the generated JSON; it contains the API key.

## Ubuntu/Debian and RHEL/CentOS

1. Install ClickHouse from the clickhouse.com repository.
2. Install Prometheus 2.25+ with the remote-write receiver enabled.
3. Install Coroot and the node agent:

```bash
curl -sfL https://raw.githubusercontent.com/coroot/coroot/main/deploy/install.sh | \
  BOOTSTRAP_PROMETHEUS_URL="http://127.0.0.1:9090" \
  BOOTSTRAP_REFRESH_INTERVAL=15s \
  BOOTSTRAP_CLICKHOUSE_ADDRESS=127.0.0.1:9000 \
  sh -
curl -sfL https://raw.githubusercontent.com/coroot/coroot-node-agent/main/install.sh | \
  COLLECTOR_ENDPOINT=http://127.0.0.1:8080 \
  SCRAPE_INTERVAL=15s \
  sh -
```

Upgrade: re-run the scripts. Uninstall: `/usr/local/bin/coroot-uninstall.sh`.

## Windows agent

Windows Server 2016+, x64 or arm64, administrator rights, outbound TCP to Coroot. Elevated PowerShell:

```powershell
$env:COROOT_COLLECTOR_ENDPOINT = 'http://COROOT_URL:8080'
$env:COROOT_API_KEY = '<API_KEY>'
iwr -useb https://raw.githubusercontent.com/coroot/coroot-node-agent/main/install.ps1 | iex
```

MSI alternative: `msiexec /i coroot-windows-agent-amd64.msi /qn COLLECTOR_ENDPOINT=http://COROOT_URL:8080 API_KEY=<API_KEY> SCRAPE_INTERVAL=15s`. Service name: `coroot-windows-agent`. Collects the Windows Event Log and container logs.

## Known documentation errors (as of 2026-10-27)

- **RHEL Prometheus versions:** the Community tab downloads v2.53.5 but the `tar`/`cp` lines reference 2.51.2. Use one version consistently.
- **RHEL Enterprise tab** skips the node-agent install step; use the node-agent command above.
- **Remote-write flag:** Ubuntu/RHEL pages use `--enable-feature=remote-write-receiver`; Prometheus 2.33+ uses `--web.enable-remote-write-receiver`, and Prometheus 3.x accepts only the latter.
- **Docker uninstall** `curl ... | docker compose rm -f -s -v` lacks `-f -` to read the compose file from stdin.
- **Slack settings path** differs between pages ("Settings → Notifications" versus "Project Settings → Integrations").
