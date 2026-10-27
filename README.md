# Coroot Agent Skills

Agent skills for investigating live production systems with [Coroot](https://github.com/coroot/coroot) through its MCP server: incidents, deployments, CPU and memory allocation, threads, Kubernetes capacity, network and disk bottlenecks, cost, and failed QA runs.

Each skill is a folder with a `SKILL.md` and optional `references/`. Agents load only the skill's name and description up front. The full instructions load when the skill activates, and reference files load only when needed.

## Skills

| Skill | Use it for |
|---|---|
| [`coroot-observability`](skills/coroot-observability) | Core rules, tool map (local wrappers and official MCP), query limits, status codes, safety. Entry point that routes to the others. |
| [`coroot-incident-rca`](skills/coroot-incident-rca) | Outages, SLO violations, error and latency spikes, incident reports. |
| [`coroot-deployment-analysis`](skills/coroot-deployment-analysis) | Before/after comparison of a deployment or release; rollback decisions. |
| [`coroot-resource-analysis`](skills/coroot-resource-analysis) | CPU, memory, throttling, OOM, threads, Kubernetes capacity, network, disk, cost, right-sizing. |
| [`coroot-qa-failure-diagnosis`](skills/coroot-qa-failure-diagnosis) | Failed Cypress/Cucumber tests and CI runs; observability-based release gates. |

Each specialised skill works on its own. Install `coroot-observability` alongside them for the full tool map and safety rules.

## Install

```bash
# All skills
npx skills add ahmedrzzaa/coroot-agent-skills

# One skill
npx skills add ahmedrzzaa/coroot-agent-skills --skill coroot-incident-rca

# Globally, for every project
npx skills add -g ahmedrzzaa/coroot-agent-skills

# Show installed skills
npx skills list
```

Manual install for Claude Code: copy a folder from `skills/` into `~/.claude/skills/` (user-wide) or `.claude/skills/` (per project).

## Requirements

- A Coroot instance with MCP enabled. The endpoint is `/mcp` on the instance.
- Authentication: OAuth 2.0 for interactive clients (Claude Code, Cursor, Codex), or an API key sent as `Authorization: Bearer <api key>` for headless agents.
- Either the local `coroot_*` wrapper tools or the official Coroot MCP tools. The skills name both and map one to the other; see [`tools.md`](skills/coroot-observability/references/tools.md).
- `list_anomalies` and `investigate_anomaly` require Coroot Enterprise Edition.

## Layout

```
skills/
├── coroot-observability/
│   ├── SKILL.md
│   └── references/
│       ├── tools.md
│       └── official-docs.md
├── coroot-incident-rca/
│   └── SKILL.md
├── coroot-deployment-analysis/
│   ├── SKILL.md
│   └── references/
│       └── report-template.md
├── coroot-resource-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── cpu.md
│       ├── memory.md
│       ├── threads-and-processes.md
│       ├── kubernetes-capacity.md
│       ├── network-disk-dependencies.md
│       ├── cost-optimization.md
│       ├── metrics.md
│       └── report-template.md
└── coroot-qa-failure-diagnosis/
    └── SKILL.md
```

## Adding a skill

1. Create `skills/<name>/SKILL.md`. Use lowercase letters, digits, and hyphens for `<name>`, and match the folder name exactly.
2. Add frontmatter with `name` and a `description` that says what the skill does and when to use it.
3. Keep `SKILL.md` short. Move long material into `references/` and link it from `SKILL.md`.

## Coroot documentation

- MCP overview: https://docs.coroot.com/mcp/overview/
- MCP documentation: https://docs.coroot.com/mcp/
- Authentication: https://docs.coroot.com/configuration/authentication/
- Architecture: https://docs.coroot.com/installation/architecture/
- Installation requirements: https://docs.coroot.com/installation/requirements/
- Application logs: https://docs.coroot.com/logs/application/
- Log monitoring: https://docs.coroot.com/logs/
- Incident investigation: https://docs.coroot.com/alerting/incidents/
- AI configuration: https://docs.coroot.com/ai/configuration/
- Node-agent metrics: https://docs.coroot.com/metrics/node-agent
- Source repository: https://github.com/coroot/coroot
