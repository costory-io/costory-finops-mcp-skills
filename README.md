# Costory FinOps MCP: the FinOps harness for platform teams, in Claude, Cursor or any MCP client

[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-io.costory%2Ffinops-blue)](https://registry.modelcontextprotocol.io/v0.1/servers?search=io.costory/finops)
[![Claude connector](https://img.shields.io/badge/Claude-connector-d97757)](https://claude.ai/directory/connectors/costory)
[![Cursor Directory](https://img.shields.io/badge/cursor.directory-costory-black)](https://cursor.directory/plugins/costory)
[![Costory MCP connector](https://glama.ai/mcp/connectors/io.costory.app-api/costory/badges/score.svg)](https://glama.ai/mcp/connectors/io.costory.app-api/costory)
[![Costory Finops Mcp Skills on AI Agents Listing](https://aiagentslisting.com/costory-finops-mcp-skills/badge.svg?claim=b4cf4c826fe986336f9907b6e40e0a21)](https://aiagentslisting.com/mcp/costory-finops-mcp-skills)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-green.svg)](./LICENSE)

Costory is built by data engineers. Every bill (AWS, Google Cloud, Azure, Oracle Cloud, Snowflake, Databricks, Datadog, OpenAI, Anthropic and more) is normalized into one model on BigQuery and joined with your context: telemetry, CI, dbt, product events and business metrics. One model for every provider is what keeps the price flat. See [supported providers](https://docs.costory.io/setup/billing).

One person curious about cost? Ask Claude and DIY. A whole org that needs the same number? Use Costory.

### Why platform teams use it

1. **Allocation that holds in a large org.** [Enrichments](https://docs.costory.io/features/tagging/enrichments) are YAML lookup files you keep in git: one stable key, such as a component, fills team, cost center or product columns. Entries are dated, so a reorg never rewrites last quarter, and [shared costs](https://docs.costory.io/features/tagging/shared-cost-allocation) can be split by real usage. Reviewed in a PR, published through the API or Terraform.
2. **Everything included, at a flat price.** Unlimited users and data sources, every skill and the MCP in every [plan](https://docs.costory.io/get-started/plans). Starter from $300/month. Pay more only when you use more. Never a % of your bill. App usage (queries, reports, alerts) and Ask Costory have a fair use. You see it in the app, get a warning at 70%, and nothing is ever cut off. Starter fits 80% of our customers. Fair use per plan: Standard (Starter), 3x (Team), 6x (Business), 10x+ (Custom). Above the limit: slowed, never cut off. Agents connected through the MCP use no Ask Costory credits. 30-day free trial, no credit card.
3. **The FinOps harness.** An agent with skills, your company context and loops to monitor, investigate, report and recommend. The [hosted MCP server](https://docs.costory.io/features/mcp) and the skills in this repo let Claude, Cursor or any MCP client explain a cost change, build a report or create an alert. It also answers in Slack. Loops are [Scouts](https://www.costory.io/finops-harness): assign one to your reports and alerts, give it instructions in plain text, and it adds the cause, the owner and the next step to every delivery. Costory runs it.
4. **Open by default.** [Public API](https://docs.costory.io/api-reference/overview), [export of allocated data to BigQuery](https://docs.costory.io/setup/destinations/bigquery) and other destinations, skills published here as markdown. No lock-in.

### Two ways teams come to Costory

Moving from another FinOps tool? Costory reads billing exports directly from each provider, read-only, so it runs alongside your current tool for the 30-day trial. Enrichment rules are YAML in git: reviewed in a PR, managed with Terraform. Starter from $300/month, never a % of your bill. See [moving from another tool](https://www.costory.io/switch).

Asked to build one? One engineer with Claude and a billing API gets a useful dashboard in an afternoon. That part is free, and we recommend it. A tool the whole team can trust takes weeks, then maintenance forever. We built that part. You keep control: YAML, formulas, the API, Terraform, and export to your own warehouse. See [build vs buy](https://www.costory.io/build-vs-buy).

### What Costory does not do

- It reads billing and usage exports and never gets write access to your cloud accounts. MCP actions create Costory objects only: virtual dimensions, alerts, reports, dashboards and events.
- It recommends optimizations, for example rightsize an instance group or partition a table. Your coding agent or your team implements them. It never changes your code or infrastructure, never applies automated rightsizing, and never buys Reserved Instances or Savings Plans.
- It does not promise a savings percentage. Recommendations come with an impact estimate from your own usage.
- It never charges a percentage of your bill.

Domain auto-join on every plan; enterprise SSO from Team. See [identity and access](https://docs.costory.io/docs/sso-access-control).

Customers include Pennylane, Alan, Deezer, Delivery Hero and Voodoo.

This repository is the open part: the **agent skills and plugin packaging** (Claude Code, Codex, Cursor) that teach
an agent how to run FinOps workflows on top of the Costory MCP tools.

- **Endpoint:** `https://app-api.costory.io/mcp` (streamable HTTP)
- **Auth:** OAuth 2.1 in the browser. No IAM credentials, no Docker, no local server
- **Docs:** [docs.costory.io/features/mcp](https://docs.costory.io/features/mcp)
- **Registry name:** `io.costory/finops` on the [official MCP Registry](https://registry.modelcontextprotocol.io)

### What engineers ask it

- "Why did our AWS bill jump last week? Which deploys line up with it?"
- "Break down Kubernetes cost by namespace and team for the payments cluster."
- "What is our total AI spend across OpenAI, Anthropic, and Bedrock this month, by team and model?"
- "Split our shared Cloud SQL cost across teams by usage and publish it as a dimension."
- "What is our cloud cost per active user, this month vs last month?"
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."
- "Send the top 5 cost movers to Slack every Monday."

## Connect the MCP

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Costory_MCP-0098FF?style=flat-square)](https://insiders.vscode.dev/redirect/mcp/install?name=costory&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fapp-api.costory.io%2Fmcp%22%7D)
[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Install_Costory_MCP-24bfa5?style=flat-square)](https://insiders.vscode.dev/redirect/mcp/install?name=costory&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fapp-api.costory.io%2Fmcp%22%7D&quality=insiders)
[![Add to Cursor](https://img.shields.io/badge/Cursor-Add_Costory_MCP-000000?style=flat-square&logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=costory&config=eyJ1cmwiOiJodHRwczovL2FwcC1hcGkuY29zdG9yeS5pby9tY3AifQ%3D%3D)

One click adds `https://app-api.costory.io/mcp` to VS Code or Cursor; the OAuth login opens in the browser on first use.
Cursor deeplink, if you prefer to paste it: `cursor://anysphere.cursor-deeplink/mcp/install?name=costory&config=eyJ1cmwiOiJodHRwczovL2FwcC1hcGkuY29zdG9yeS5pby9tY3AifQ%3D%3D`

You need a Costory workspace with [billing data connected](https://docs.costory.io/get-started/welcome). Starter from $300/month, flat; 30-day free trial.

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

Claude Desktop (developer config, `claude_desktop_config.json`), see [alternative JSON config](https://docs.costory.io/features/mcp#alternative-json-config):

```json
{
  "mcpServers": {
    "costory": {
      "url": "https://app-api.costory.io/mcp"
    }
  }
}
```

VS Code (`.vscode/mcp.json`):

```json
{
  "servers": {
    "costory": {
      "type": "http",
      "url": "https://app-api.costory.io/mcp"
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

**Skills only, any agent ([skills.sh](https://skills.sh)):** installs the seven skills into Claude Code, Cursor,
Codex and other agents that read Agent Skills. Connect the MCP separately (see above).

```bash
npx skills add costory-io/costory-finops-mcp-skills
```

**Kiro:** this repo ships a [Kiro power](./powers/costory/) (MCP connection + FinOps skill). In Kiro, open the
Powers panel → **Add Custom Power** → **Import power from GitHub** and enter
`https://github.com/costory-io/costory-finops-mcp-skills/tree/main/powers/costory`.

## Layout

```
.mcp.json                              ← ready-to-copy MCP client config
skills.json                            ← MCP skillId -> SKILL.md path
.claude-plugin/marketplace.json
server.json                            ← official MCP Registry entry (io.costory/finops)
gemini-extension.json + GEMINI.md      ← Gemini CLI extension (MCP connection + context)
powers/costory/                        ← Kiro power (plugin.json + mcp.json + skills/costory-finops)
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

## Support

support@costory.io · [Documentation](https://docs.costory.io/features/mcp) · [Privacy Policy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

## License

Apache-2.0, see [LICENSE](./LICENSE).
