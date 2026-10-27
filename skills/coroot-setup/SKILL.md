---
name: coroot-setup
description: Installs, configures, and connects Coroot - Kubernetes (Operator/Helm), OpenShift, Docker, Docker Swarm, ECS, Ubuntu/RHEL, and Windows agents; external Prometheus and ClickHouse, retention, high availability, S3 tiering; AWS/GCP/OCI managed-database integrations; Slack, Teams, PagerDuty, Opsgenie, and webhook notifications; OpenTelemetry instrumentation, custom metrics, dashboards; and connecting Claude Code, Cursor, Codex, or headless agents to the Coroot MCP server. Use when the user wants to install, upgrade, configure, or integrate Coroot rather than investigate a live problem.
metadata:
  author: aurora
  version: "3.0"
  category: observability
  documentation: "https://docs.coroot.com/installation/architecture/"
---

# Coroot Setup

Helps install and configure Coroot and its integrations. For investigating a running system, use the other Coroot skills.

## Safety

- Installing, upgrading, or uninstalling changes infrastructure. Show the exact commands and target (cluster context, namespace, host) and get explicit confirmation before running them. Never run uninstall commands without being asked.
- Check the current state first: `kubectl config current-context`, existing `coroot` namespace or Helm releases, running containers or services.
- Keep secrets out of commands, chat, and committed files. Use Kubernetes Secrets (`*Secret` fields in the Coroot custom resource), environment variables, or secret managers. Never ask the user to paste a license key, API key, or password into the conversation.
- Commands below are copied from the Coroot docs (read 2026-10-27). Docs change; when a command fails, check the linked page before improvising. Known documentation errors are listed in [references/install.md](references/install.md).

## Workflow

1. **Gather facts.** Platform (Kubernetes flavour, Docker, VMs, ECS, Windows), Community or Enterprise edition, Linux kernel version, existing Prometheus or ClickHouse to reuse, managed databases to monitor, and where notifications should go.
2. **Check requirements.** Kernel 4.16+ for the node agent; Docker-in-Docker (MiniKube), WSL1, and GKE Autopilot cannot run it. See [references/install.md](references/install.md).
3. **Install** with the platform's commands from [references/install.md](references/install.md).
4. **Verify.** Pods or containers running, UI reachable on port 8080, nodes and applications appearing in the UI within a few minutes.
5. **Configure storage and retention** if needed ([references/configuration.md](references/configuration.md)). TTLs only apply when tables are created.
6. **Connect data sources and notifications** ([references/integrations.md](references/integrations.md)): managed databases, cloud integrations, OpenTelemetry, custom metrics, alert destinations.
7. **Connect AI agents** to the MCP server and, optionally, configure AI RCA ([references/connect-agents.md](references/connect-agents.md)).

## References

| Task | Reference |
|---|---|
| Install, upgrade, uninstall on any platform; requirements; overhead; known doc errors | [references/install.md](references/install.md) |
| Server, node-agent, and cluster-agent settings; external Prometheus/ClickHouse; HA; S3 tiering; TTLs | [references/configuration.md](references/configuration.md) |
| AWS/GCP/OCI, notifications and webhook templates, OpenTelemetry, custom metrics, dashboards | [references/integrations.md](references/integrations.md) |
| Claude Code, Cursor, Codex, service accounts and API keys, AI RCA, Coroot Cloud | [references/connect-agents.md](references/connect-agents.md) |
| ArgoCD, EKS+RDS, GKE+Cloud SQL, OKE+MySQL, ClickHouse S3, JFrog, multi-cluster, shared database | [references/guides.md](references/guides.md) |
