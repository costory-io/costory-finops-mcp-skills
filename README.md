# Costory FinOps MCP: agent skills and plugin

The **Costory FinOps MCP** is a hosted [Model Context Protocol](https://modelcontextprotocol.io) server that lets Claude, Cursor, VS Code, Codex, or Dust answer questions about your cloud and AI spend.

Feeding raw AWS or GCP billing lines into a prompt does not work. Costory acts as a **context layer**: it normalizes billing from AWS, GCP, Azure, Snowflake, Datadog, OpenAI, and Anthropic into one schema, allocates shared and untagged cost from real usage metrics, and correlates spend with deploy events and incidents. The assistant then calls structured tools against data that is already allocated and explained.

This repository holds the **agent skills** that sit on top of those tools: the workflows that turn "why did prod cost jump last week" into the right sequence of tool calls.

- **Full MCP documentation:** [docs.costory.io/features/mcp](https://docs.costory.io/features/mcp)
- **Endpoint:** `https://app-api.costory.io/mcp`
- **Auth:** OAuth, no IAM credentials, no Docker, no local server

## Connect the MCP

You need a Costory workspace with [billing data connected](https://docs.costory.io/get-started/welcome). A 15-day trial is available.

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
| `query` | Cost, usage, metric, formula, and budget investigation. Explorer period-over-period only; hands off "what changed" to `reports` Explain |
| `virtual-dimensions` | Create, edit, preview, and publish custom cost axes with ordered CEL rules |
| `dashboards` | Create or extend dashboards with context-first widget inheritance and overview generation |
| `reports` | Scheduled Slack, Teams, and email reports, and preview-first DIGEST to explain last month's cost |
| `recipes` | Ready-made tracking designs matched to an outcome, then handed off to the skills above to build |

Recipes currently cover budget-vs-actual dashboards, EC2 spike alerts, prod-vs-R&D splits, untagged coverage, marketplace spend, provider credits, namespace cost, compute drilldowns, and period-change explanation. See [`plugins/costory/skills/recipes/`](./plugins/costory/skills/recipes/).

## Install as a plugin

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
plugins/costory/
  .claude-plugin/plugin.json
  skills/
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
