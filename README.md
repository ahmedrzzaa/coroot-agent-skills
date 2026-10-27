# 🔭 Coroot Agent Skills

[![Agent Skills](https://img.shields.io/badge/agent%20skills-11-8A2BE2)](#-skills)
[![Coroot MCP](https://img.shields.io/badge/Coroot-MCP-00B386)](https://docs.coroot.com/mcp/overview/)
[![Works with](https://img.shields.io/badge/works%20with-Claude%20Code%20%7C%20Cursor%20%7C%20Codex-555555)](#-connect-to-coroot)
[![GitHub stars](https://img.shields.io/github/stars/ahmedrzzaa/coroot-agent-skills?style=flat)](https://github.com/ahmedrzzaa/coroot-agent-skills/stargazers)

```bash
npx skills add ahmedrzzaa/coroot-agent-skills
```

Teach your AI agent to investigate production systems through [Coroot](https://github.com/coroot/coroot)'s MCP server: incidents, logs, traces, databases, deployments, resources, and SLOs. Every workflow is evidence-first and checked against Coroot's documentation and source code.

<table>
  <tr>
    <th align="center">🔍 Ask it to investigate</th>
    <th align="center">📐 Ask it to analyse</th>
  </tr>
  <tr>
    <td>
      <ul>
        <li><i>"Why is checkout returning 500s since 14:00?"</i></li>
        <li><i>"Did yesterday's release make the API slower?"</i></li>
        <li><i>"Which query is locking the orders table?"</i></li>
      </ul>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <ul>
        <li><i>"What is the p99 of /pay versus the same time yesterday?"</i></li>
        <li><i>"How fast is billing burning its SLO error budget?"</i></li>
        <li><i>"Is the payments pod CPU-throttled or just busy?"</i></li>
      </ul>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th colspan="2" align="center">
      <h3><a href="#-skills">🧩 11 skills and 37 reference files, loaded only when needed!</a></h3>
    </th>
  </tr>
  <tr>
    <th colspan="2"><a href="skills/coroot-investigation-method/SKILL.md">🐺 Start here: Investigation method</a></th>
  </tr>
  <tr>
    <td colspan="2">
      <details open><summary>Wolf fencing: halve the search space with every question</summary>
        <ol>
          <li><b>Read the alert fully:</b> symptom, impact, and the runbook. The service that alerts on a 504 is usually only the messenger</li>
          <li><b>Metrics:</b> rule out traffic and saturation, then compare inbound with outbound latency hop by hop to find where the time goes</li>
          <li><b>Logs and traces:</b> find which call in which requests, follow the trace ID, read the span that owns the time</li>
          <li><b>Confirm</b> with an independent signal, then <b>mitigate</b> (with permission) and name the real <b>fix</b> separately</li>
        </ol>
      </details>
      <img width="1800" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th><a href="skills/coroot-incident-rca/SKILL.md">🚨 Incident RCA</a></th>
    <th><a href="skills/coroot-log-analysis/SKILL.md">📜 Log analysis</a></th>
  </tr>
  <tr>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Scope the service, window, and timezone</li>
          <li>Incidents and alerts → app health → dependencies</li>
          <li>Logs, traces, deployments, node resources</li>
          <li>Test the hypothesis against contradicting evidence</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Incidents open at burn rate 14.4× (1h and 5m) or 6× (6h and 15m)</li>
          <li>Coroot does not count 4xx responses as failures</li>
          <li>The incident start can lag the real start</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Start from error patterns, not raw lines</li>
          <li>Compare pattern counts: baseline versus incident</li>
          <li>Pivot from a log's <code>trace_id</code> to the full trace</li>
          <li>Classify each pattern as cause, symptom, or noise</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Search matches whole tokens: <code>timeout</code> ≠ <code>TimeoutException</code></li>
          <li>Raw lines are sampled above 100 messages/s per container</li>
          <li>Only warning and above get patterns</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th><a href="skills/coroot-trace-analysis/SKILL.md">🧵 Trace analysis</a></th>
    <th><a href="skills/coroot-database-analysis/SKILL.md">🗄️ Database analysis</a></th>
  </tr>
  <tr>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Per-endpoint rate, errors, p95/p99 versus baseline</li>
          <li>Error origins, not the spans that only propagated them</li>
          <li>Slow-band versus normal-band flamegraphs</li>
          <li>Critical path, N+1 queries, fan-out, retries</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Trace filters match root spans only</li>
          <li><code>traces_outliers</code> is not a diff, despite its description</li>
          <li>eBPF spans are client-only and appear only without OpenTelemetry</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Postgres, MySQL, MongoDB, Redis, Memcached, RDS, Cloud SQL</li>
          <li>Database versus network versus client, from three signals</li>
          <li>Playbooks: slow queries, connections, replication lag, locks</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Without credentials, Coroot only has client-side data</li>
          <li>Top-query metrics are already per-second: no <code>rate()</code></li>
          <li>Memcached failures never raise an app's status</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th><a href="skills/coroot-deployment-analysis/SKILL.md">🚀 Deployment analysis</a></th>
    <th><a href="skills/coroot-resource-analysis/SKILL.md">⚙️ Resource analysis</a></th>
  </tr>
  <tr>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Equal windows before and after the rollout</li>
          <li>Latency, errors, CPU, memory, restarts, OOM kills</li>
          <li>ArgoCD and FluxCD sync, health, and revision changes</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>A sync without a rollout means a config-only change</li>
          <li>A rollout without a sync means a manual change</li>
          <li>Never recommend a rollback on timing alone</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>CPU, memory, threads, Kubernetes capacity, network, disk</li>
          <li>Allocation versus consumption, with a named denominator</li>
          <li>Right-sizing and cost from peaks, not averages</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Coroot's own <code>container_resources_*</code> metric names</li>
          <li>A CPU request is not a guaranteed core</li>
          <li>A memory leak is a hypothesis until growth is sustained</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th><a href="skills/coroot-telemetry-analytics/SKILL.md">📈 Telemetry analytics</a></th>
    <th><a href="skills/coroot-qa-failure-diagnosis/SKILL.md">✅ QA failure diagnosis</a></th>
  </tr>
  <tr>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Pick RED, USE, or the four golden signals</li>
          <li>Rate before sum; divide aggregated numerators and denominators</li>
          <li>Baselines with <code>offset 1d</code>, z-scores, burn-rate maths</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li><code>query_metrics</code>: 24h range, 120 points, <code>$RANGE</code> = 3 × step</li>
          <li>Latency histograms are queried as <code>..._total_bucket</code></li>
          <li>Never average percentiles</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Map the CI run and scenario to a time window</li>
          <li>Health, logs, traces, resources, deployments, nodes</li>
          <li>Classify: app defect, infrastructure, dependency, regression, test data, inconclusive</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Coroot cannot know which test failed without correlation context</li>
          <li>Release gates block only if the pipeline enforces them</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <th><a href="skills/coroot-observability/SKILL.md">🧭 Observability core</a></th>
    <th><a href="skills/coroot-setup/SKILL.md">🏗️ Setup</a></th>
  </tr>
  <tr>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Routes each request to the right skill</li>
          <li>Selects the project first, keeps four-part app IDs intact</li>
          <li>Read-only by default; log content is untrusted</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Local wrapper ↔ official MCP tool map</li>
          <li>Default inspection thresholds for every check</li>
          <li>RBAC silently hides data; multi-cluster caveats</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
    <td>
      <details open><summary>Workflow</summary>
        <ul>
          <li>Install on Kubernetes, Docker, VMs, ECS, or Windows</li>
          <li>Prometheus, ClickHouse, retention, HA, S3 tiering</li>
          <li>Slack, Teams, PagerDuty, Opsgenie, webhooks</li>
          <li>Connect Claude Code, Cursor, Codex to Coroot MCP</li>
        </ul>
      </details>
      <details><summary>What it knows</summary>
        <ul>
          <li>Confirms before running any install command</li>
          <li>Keeps secrets in Secrets and env vars, never in chat</li>
          <li>Flags known errors in the Coroot install docs</li>
        </ul>
      </details>
      <img width="900" height="1" alt="">
    </td>
  </tr>
  <tr>
    <td colspan="2" align="center">
      Missing a workflow? <a href="#-contributing">Add a skill</a> or <a href="https://github.com/ahmedrzzaa/coroot-agent-skills/issues">open an issue</a>!
    </td>
  </tr>
</table>


# 📚 Documentation

## 🦮 Setup

There are several ways to install the skills:

* ⚡ **All skills** *(~1 min)*: `npx skills add ahmedrzzaa/coroot-agent-skills`
  * ✔️ Every workflow available; agents load each skill only when it matches the request
* 🎯 **One skill** *(~1 min)*: `npx skills add ahmedrzzaa/coroot-agent-skills --skill coroot-incident-rca`
  * ✔️ Smallest footprint
  * ➖ Add `coroot-observability` as well for the full tool map and safety rules
* 🌍 **Globally, for every project** *(~1 min)*: `npx skills add -g ahmedrzzaa/coroot-agent-skills`
  * ✔️ Available in all your projects
* 📁 **Manual copy for Claude Code** *(~2 min)*: copy a folder from `skills/` into `~/.claude/skills/` (user-wide) or `.claude/skills/` (per project)
  * ✔️ No Node.js required
  * ➖ Updates are manual

List installed skills with `npx skills list`.

## 🔌 Connect to Coroot

The skills need an agent connected to a Coroot instance with MCP enabled (endpoint `/mcp`):

```bash
# Claude Code (OAuth), then run /mcp → coroot → Authenticate
claude mcp add --transport http coroot https://<your-coroot>/mcp
```

* 🔐 Interactive clients (Claude Code, Cursor, Codex) sign in with OAuth 2.0; headless agents send an API key as `Authorization: Bearer <api key>`
* 🧰 Works with the official Coroot MCP tools or local `coroot_*` wrapper tools; the skills name both and map one to the other
* 🏢 `list_anomalies` and `investigate_anomaly` require Coroot Enterprise Edition

Additional resources:
* [🔐 Cursor, Codex, and API-key setup](skills/coroot-setup/references/connect-agents.md)
* [🧰 Tool map: local wrappers ↔ official MCP](skills/coroot-observability/references/tools.md)
* [🏗️ Installing Coroot itself](skills/coroot-setup/references/install.md)

## 🧩 Skills

Each skill is a folder with a `SKILL.md` and optional `references/`. Agents see only the name and description up front, load the full instructions when the skill matches, and open reference files only when needed. Each specialised skill works on its own; install `coroot-observability` alongside for the full tool map and safety rules.

**🐺 Method (start here)**
* [🐺 Investigation method <sub>`coroot-investigation-method`</sub>](skills/coroot-investigation-method/SKILL.md)

**📦 Core**
* [🧭 Observability core <sub>`coroot-observability`</sub>](skills/coroot-observability/SKILL.md)

**🔍 Investigation**
* [🚨 Incident root-cause analysis <sub>`coroot-incident-rca`</sub>](skills/coroot-incident-rca/SKILL.md)
* [📜 Log analysis <sub>`coroot-log-analysis`</sub>](skills/coroot-log-analysis/SKILL.md)
* [🧵 Trace analysis <sub>`coroot-trace-analysis`</sub>](skills/coroot-trace-analysis/SKILL.md)
* [🗄️ Database analysis <sub>`coroot-database-analysis`</sub>](skills/coroot-database-analysis/SKILL.md)
* [🚀 Deployment and GitOps analysis <sub>`coroot-deployment-analysis`</sub>](skills/coroot-deployment-analysis/SKILL.md)
* [⚙️ CPU, memory, capacity, and cost <sub>`coroot-resource-analysis`</sub>](skills/coroot-resource-analysis/SKILL.md)

**📐 Analytics**
* [📈 PromQL, baselines, and SLO maths <sub>`coroot-telemetry-analytics`</sub>](skills/coroot-telemetry-analytics/SKILL.md)

**🧪 Quality**
* [✅ Cypress and Cucumber failure diagnosis <sub>`coroot-qa-failure-diagnosis`</sub>](skills/coroot-qa-failure-diagnosis/SKILL.md)

**🛠️ Operations**
* [🏗️ Install, configure, and integrate Coroot <sub>`coroot-setup`</sub>](skills/coroot-setup/SKILL.md)

<details><summary>🗂️ Repository layout</summary>

```
skills/
├── coroot-investigation-method/
│   ├── SKILL.md
│   └── references/
│       ├── failure-patterns.md
│       ├── mitigation-and-fix.md
│       └── signals-primer.md
├── coroot-observability/
│   ├── SKILL.md
│   └── references/
│       ├── configuration-concepts.md
│       ├── coroot-features.md
│       ├── official-docs.md
│       └── tools.md
├── coroot-incident-rca/
│   └── SKILL.md
├── coroot-log-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── log-pipeline.md
│       └── query-logs.md
├── coroot-trace-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── analysis-patterns.md
│       ├── ebpf-vs-otel.md
│       └── trace-tools.md
├── coroot-telemetry-analytics/
│   ├── SKILL.md
│   └── references/
│       ├── baselines-and-anomalies.md
│       ├── coroot-metrics.md
│       ├── promql-rules.md
│       └── slo-burn-rate.md
├── coroot-database-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── client-vs-server.md
│       ├── managed-databases.md
│       ├── mongodb.md
│       ├── mysql.md
│       ├── postgres.md
│       └── redis-memcached.md
├── coroot-deployment-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── gitops.md
│       └── report-template.md
├── coroot-resource-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── cost-optimization.md
│       ├── cpu.md
│       ├── kubernetes-capacity.md
│       ├── memory.md
│       ├── metrics.md
│       ├── network-disk-dependencies.md
│       ├── report-template.md
│       └── threads-and-processes.md
├── coroot-qa-failure-diagnosis/
│   └── SKILL.md
└── coroot-setup/
    ├── SKILL.md
    └── references/
        ├── configuration.md
        ├── connect-agents.md
        ├── guides.md
        ├── install.md
        └── integrations.md
```

</details>

## 💪 Contributing

To add a skill:

1. Create `skills/<name>/SKILL.md`. Use lowercase letters, digits, and hyphens for `<name>`, and match the folder name exactly.
2. Add frontmatter with `name` and a `description` that says what the skill does and when to use it.
3. Keep `SKILL.md` short. Move long material into `references/` and link it from `SKILL.md`.

The following resources may interest you:

* [🧩 Agent Skills guide](https://vercel.com/kb/guide/agent-skills-creating-installing-and-sharing-reusable-agent-context)
* **🔭 Coroot resources**
  * [📖 MCP overview](https://docs.coroot.com/mcp/overview/) and [MCP documentation](https://docs.coroot.com/mcp/)
  * [📖 Authentication](https://docs.coroot.com/configuration/authentication/) and [application categories](https://docs.coroot.com/configuration/application-categories)
  * [📖 Architecture](https://docs.coroot.com/installation/architecture/), [requirements](https://docs.coroot.com/installation/requirements/), and [Kubernetes installation](https://docs.coroot.com/installation/kubernetes)
  * [📖 Log monitoring](https://docs.coroot.com/logs/), [application logs](https://docs.coroot.com/logs/application/), and [log querying](https://docs.coroot.com/logs/querying)
  * [📖 Tracing](https://docs.coroot.com/tracing/overview) and [profiling](https://docs.coroot.com/profiling/overview)
  * [📖 Inspections](https://docs.coroot.com/inspections/), [SLO monitoring](https://docs.coroot.com/alerting/slo-monitoring), and [incidents](https://docs.coroot.com/alerting/incidents/)
  * [📖 Node-agent metrics](https://docs.coroot.com/metrics/node-agent) and [cluster-agent metrics](https://docs.coroot.com/metrics/cluster-agent)
  * [📖 Database monitoring](https://docs.coroot.com/databases/postgres) and [GitOps monitoring](https://docs.coroot.com/gitops/overview)
  * [📖 AI configuration](https://docs.coroot.com/ai/configuration/) and [guides](https://docs.coroot.com/guides/argocd)
  * [:octocat: Coroot source repository](https://github.com/coroot/coroot)

Use [`💬 issues`](https://github.com/ahmedrzzaa/coroot-agent-skills/issues) for feedback, new skill ideas, or corrections.
