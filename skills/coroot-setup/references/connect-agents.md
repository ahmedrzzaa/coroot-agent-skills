# Connecting AI agents and configuring AI RCA

Sources: https://docs.coroot.com/mcp/overview/, https://docs.coroot.com/configuration/authentication/, https://docs.coroot.com/ai/configuration/, https://docs.coroot.com/ai/coroot-cloud (read 2026-10-27).

## MCP endpoint

`https://<your-coroot>/mcp`, streamable HTTP. Interactive clients sign in with OAuth 2.0; headless agents use an API key.

## Interactive clients (OAuth)

- **Claude Code:** `claude mcp add --transport http coroot https://<your-coroot>/mcp`, then run `/mcp` → coroot → Authenticate.
- **Cursor:** add to `~/.cursor/mcp.json`, then Cursor Settings → MCP → Connect:

```json
{"mcpServers": {"coroot": {"url": "https://<your-coroot>/mcp"}}}
```

- **Codex:** `codex mcp add coroot --url https://<your-coroot>/mcp`, then `/mcp` → Authenticate.

## Headless agents (API key)

Create a **service account** (a passwordless user) with the least role that works, usually Viewer. Editor is needed only for `resolve_alerts`.

- **UI:** Project settings → Organization → Add user → tick Service account → key icon → create API key (shown once).
- **Config file:** `auth.serviceAccounts[{name, role, apiKeys[{key: ${AGENT_API_KEY}, description}]}]` — reference an environment variable, never a literal key.
- **Operator custom resource:** `spec.serviceAccounts[].apiKeys[].keySecret` (the Secret is generated if missing).

Connect with the key from an environment variable:

```bash
claude mcp add --transport http coroot https://<your-coroot>/mcp \
  --header "Authorization: Bearer ${COROOT_API_KEY}"
```

Rotate keys by creating the new key before deleting the old one. The user should set the variable themselves; do not ask them to paste the key into the conversation.

After connecting: the Community Edition exposes 17 tools (`list_projects` and `select_project` first); `list_anomalies` and `investigate_anomaly` are Enterprise only. Responses are capped at roughly 50–80 KB.

## AI root-cause analysis (Enterprise)

- Project Settings → AI (global setting, needs `settings.edit`).
- Providers: Anthropic (recommended by Coroot; needs egress to `api.anthropic.com:443`), OpenAI (`api.openai.com:443`), or an OpenAI-compatible API (key, base URL, model).
- Config file: `ai.provider: anthropic | openai | openai_compatible` with `anthropic.apiKey`, `openai.apiKey`, or `openaiCompatible.{apiKey,baseUrl,model}`. In the custom resource the field is `baseURL`, and every key has an `apiKeySecret` variant — use it.

## Coroot Cloud (AI RCA for Community Edition)

- Project Settings → Coroot Cloud → sign up or sign in; the API key is configured automatically.
- 10 free investigations per month; incidents can be investigated automatically.
- As code: `corootCloud.apiKey`, `corootCloud.rca.disableIncidentsAutoInvestigation`.
- RCA output appears in webhook payloads as `rca_summary` and `rca_remediations`.
