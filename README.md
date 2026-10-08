# Costory FinOps MCP: build a FinOps agent on Claude, Cursor, or any LLM

[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-io.costory%2Ffinops-blue)](https://registry.modelcontextprotocol.io/v0.1/servers?search=io.costory/finops)
[![Claude connector](https://img.shields.io/badge/Claude-connector-d97757)](https://claude.ai/directory/connectors/costory)
[![Cursor Directory](https://img.shields.io/badge/cursor.directory-costory-black)](https://cursor.directory/plugins/costory)
[![Costory MCP connector](https://glama.ai/mcp/connectors/io.costory.app-api/costory/badges/score.svg)](https://glama.ai/mcp/connectors/io.costory.app-api/costory)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-green.svg)](./LICENSE)

**Costory is the cost-data layer for a FinOps agent.** It ingests and normalizes cloud, Kubernetes, SaaS, and AI
spend (AWS, GCP, Azure, Snowflake, Datadog, OpenAI, Anthropic), allocates it to teams, products, and features, and
exposes it through a hosted [Model Context Protocol](https://modelcontextprotocol.io) server. Claude, Cursor,
VS Code, Codex, Gemini CLI, Dust, or your own agent then investigates, explains, and optimizes spend with
structured tool calls instead of raw billing lines.

This repository is the open part: the **agent skills and plugin packaging** (Claude Code, Codex, Cursor) that teach
an agent how to run FinOps workflows on top of the Costory MCP tools.

- **Endpoint:** `https://app-api.costory.io/mcp` (streamable HTTP)
- **Auth:** OAuth 2.1 in the browser. No IAM credentials, no Docker, no local server
- **Docs:** [docs.costory.io/features/mcp](https://docs.costory.io/features/mcp)
- **Registry name:** `io.costory/finops` on the [official MCP Registry](https://registry.modelcontextprotocol.io)

### Why a cost layer for a FinOps agent?

A provider billing MCP returns line items for one cloud. That works for a single account. On a real stack
(several clouds, shared Kubernetes clusters, many teams) the agent also needs to know who owns which spend, how
shared and untagged cost is split, and what changed when the bill moved. Costory maintains that context, so the
agent starts from allocated, correlated data.

### What engineers ask it

- "Why did our AWS bill jump last week? Which deploys line up with it?"
- "Break down Kubernetes cost by namespace and team for the payments cluster."
- "What is our total AI spend across OpenAI, Anthropic, and Bedrock this month, by team and model?"
- "Split our shared Cloud SQL cost across teams by usage and publish it as a dimension."
- "What is our cloud cost per active user, this month vs last month?"
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."
- "Send the top 5 cost movers to Slack every Monday."

## Connect the MCP

You need a Costory workspace with [billing data connected](https://docs.costory.io/get-started/welcome). A 14-day trial is available.

**Claude Desktop / Claude Code / Cursor / VS Code:** add a custom connector pointing at `https://app-api.costory.io/mcp`, then complete the OAuth login in the browser window that opens. Per-client walkthroughs with screenshots are in the [MCP docs](https://docs.costory.io/features/mcp).

This repo ships an [`.mcp.json`](./.mcp.json) you can copy:

```json
{
  "mcpServers": {
    "costory": {
      "type": "http",
      "url": "https://app-api.costory.io/mcp",
      "oauth": { "callbackPort": 8080 }
    }
  }
}
```

For clients without native remote-MCP support, proxy it with `mcp-remote`:

```json
{
  "mcpServers": {
    "costory": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://app-api.costory.io/mcp"]
    }
  }
}
```

### First prompt

Try "Use Costory to show my top cost drivers last month". The agent calls `get_context`, then `query`.

**If your account has access to several organizations**, `get_context` without a slug returns "Multiple organizations" with the list of slugs. The agent should call `list_organizations` (or read that list), ask which org to use, and pass that `slug` to `get_context` and the following tools. You can also name it upfront: "Use Costory org `acme` and show my top cost drivers last month".

## MCP tools reference

The server exposes tools in five groups. Names and payloads are versioned; the
[API reference](https://docs.costory.io/api-reference/overview) is the source of truth.

| Group | Tools | What they do |
|---|---|---|
| Orientation | `get_context`, `search`, `get`, `suggest_groupby`, `suggest_usage_metrics`, `suggest_actions` | Discover dimensions, dashboards, metrics, and the right way to slice a question |
| Query | `query`, `list_metrics`, `list_virtual_dimensions` | Cost, usage, metric, formula, and budget queries with period-over-period comparison |
| Allocation | `create_virtual_dimension_draft`, `update_virtual_dimension_draft`, `preview_virtual_dimension_draft`, `publish_virtual_dimension`, `virtual_dimension_overlap_matrix` | Define custom cost axes with ordered CEL rules, preview, then publish |
| Reporting | `create_report`, `update_report`, `preview_report_widget`, `run_report_now`, `create_dashboard`, `update_dashboard` | Scheduled Slack, Teams, and email reports plus dashboards built from chat |
| Alerting and events | `create_alert`, `preview_alert`, `list_alerts`, `create_event`, `update_event` | Cost and budget alerts, and event annotations for correlation |

Write tools act only inside your Costory workspace. Query scoping follows the calling user's workspace role.

## FinOps skills

Skills are the workflow layer: each one encodes how to sequence the tools above for a class of question, so the assistant does not have to rediscover it.

| MCP `skillId` | Use when |
|---|---|
| `bigquery` | BigQuery warehouse caveats (on-demand vs slots, physical vs logical, labels) plus the [BigQuery] dashboard template |
| `cost-change-investigation` | Explain a cost change with contribution, timing, usage, metric, event, alert, and terminology evidence |
| `query` | Cost, usage, metric, formula, and budget investigation. Explorer period-over-period only; hands off "what changed" to `reports` Explain |
| `virtual-dimensions` | Create, edit, preview, and publish custom cost axes with ordered CEL rules |
| `dashboards` | Create or extend dashboards with context-first widget inheritance and overview generation |
| `reports` | Scheduled Slack, Teams, and email reports, and preview-first DIGEST to explain last month's cost |
| `recipes` | Ready-made tracking designs matched to an outcome, then handed off to the skills above to build |

Recipes currently cover BigQuery warehouse routing, GCP spend-based CUDs, budget-vs-actual dashboards, EC2 spike alerts, prod-vs-R&D splits, untagged coverage, marketplace spend, provider credits, namespace cost, compute drilldowns, and period-change explanation. See [`plugins/costory/skills/recipes/`](./plugins/costory/skills/recipes/).

## Cursor Marketplace packaging

This repo also ships a Cursor plugin layout next to the Claude Code marketplace:

- `.cursor-plugin/marketplace.json` — multi-plugin marketplace manifest
- `plugins/costory/.cursor-plugin/plugin.json` — Cursor plugin manifest
- `plugins/costory/mcp.json` — hosted MCP at `https://app-api.costory.io/mcp` (OAuth in the client)
- `plugins/costory/assets/logo.png` — plugin logo

Listed on [cursor.directory/plugins/costory](https://cursor.directory/plugins/costory).

## Install as a plugin (MCP connection + skills)

The plugin wires the MCP server and installs the skills in one step.

```bash
# Claude Code
claude plugin marketplace add costory-io/costory-finops-mcp-skills
claude plugin install costory@costory

# Codex
codex plugin marketplace add costory-io/costory-finops-mcp-skills
codex plugin add costory@costory
```

## Layout

```
.mcp.json                              ← ready-to-copy MCP client config
skills.json                            ← MCP skillId -> SKILL.md path
.claude-plugin/marketplace.json
server.json                            ← official MCP Registry entry (io.costory/finops)
gemini-extension.json + GEMINI.md      ← Gemini CLI extension (MCP connection + context)
plugins/costory/
  .claude-plugin/plugin.json
  .mcp.json                            ← MCP server wired by the Claude Code / Codex plugin
  README.md
  LICENSE                              ← Apache-2.0 (copy of the root LICENSE)
  skills/
    bigquery/SKILL.md
    cost-change-investigation/SKILL.md
    query/SKILL.md
    virtual-dimensions/SKILL.md
    dashboards/SKILL.md
    reports/SKILL.md
    recipes/SKILL.md  + recipe library
```

## Serving skills over MCP (`get_skill`)

[`skills.json`](./skills.json) maps each MCP `skillId` to a `SKILL.md` path. When wiring costory-app, load the file from this repo (or a pinned release), strip optional YAML frontmatter, and return the markdown body.

```json
{
  "skillId": "dashboards",
  "path": "plugins/costory/skills/dashboards/SKILL.md"
}
```

## Authoring

See [AGENTS.md](./AGENTS.md) for layout rules, version bumps, and validation. Use [SKILL_TEMPLATE.md](./SKILL_TEMPLATE.md) when adding a skill.

## License

Apache-2.0, see [LICENSE](./LICENSE).
