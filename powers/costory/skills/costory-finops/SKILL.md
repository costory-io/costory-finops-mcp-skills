---
name: costory-finops
description: "Use when the user asks about cloud, Kubernetes, SaaS or AI spend through Costory: why a bill moved, cost by team/service/namespace, unit economics, budgets, allocating shared or untagged cost, or building Costory dashboards, scheduled reports and cost alerts. Covers org selection, the tool order, preview-then-confirm rules for write tools, and when to load the detailed Costory skills with get_skill."
license: "Apache-2.0"
compatibility: "Requires the hosted Costory FinOps MCP (https://app-api.costory.io/mcp, OAuth in the browser) and a Costory workspace with billing data connected."
metadata:
  author: "Costory"
  version: "0.12.3"
---

# Costory FinOps Agent

## Overview

Costory is the cost-data layer for a FinOps agent. It ingests and normalizes billing from AWS, GCP, Azure,
Snowflake, Datadog, OpenAI and Anthropic, allocates it to teams, products, environments and features, and exposes it
through the hosted Costory MCP server (`costory` in this power). Use it to investigate, explain and allocate spend
with structured tool calls instead of raw billing lines.

This skill is the condensed playbook. The detailed workflows (payload formats, edge cases) are served by the MCP
itself: call `get_skill` with one of these `skillId` values before non-trivial work.

| `skillId` | Load it before |
|---|---|
| `query` | any non-trivial `query` call: scope vs split, period comparison, unit economics, budgets |
| `cost-change-investigation` | explaining what changed, when, and why |
| `virtual-dimensions` | `create_virtual_dimension_draft`, `update_virtual_dimension_draft`, `publish_virtual_dimension` |
| `dashboards` | `create_dashboard`, `update_dashboard` |
| `reports` | `create_report`, `preview_report_widget` (DIGEST), `run_report_now` |
| `bigquery` | BigQuery warehouse questions (on-demand vs slots, physical vs logical storage, labels) |
| `recipes` | the user states an outcome ("tag coverage", "cost per namespace", "EC2 spike alert") rather than a tool |

## Prerequisites

- A Costory workspace with billing data connected (a 14-day trial is available at https://www.costory.io).
- On first use, the MCP client opens a browser window for the OAuth login. No IAM keys, Docker or local server.
- If tools fail with an authentication error, ask the user to reconnect the `costory` MCP server and log in again.

## Step 1: Load context (always first)

Call `get_context` at the start of every conversation. It returns the dimensions, popular group-bys, dashboards
and the workspace currency. Never invent dimension or CEL field names; take them from `get_context` or `search`.

**Several organizations:** if `get_context` returns "Multiple organizations" with a list of slugs, do not pick one.
Ask the user which org to use (from that list, or call `list_organizations`), then pass that `slug` to
`get_context` and to every following tool call. If the user already named the org (e.g. "Use Costory org `acme`"),
use that slug directly.

## Step 2: Route the request

| The user wants | Do |
|---|---|
| Spend by service / team / account / namespace, a trend, a total | `query` (load `get_skill` `query`). Use `suggest_groupby` when the right split is unclear. |
| "Why did the bill jump?" (one-shot) | Load `cost-change-investigation` or `reports` (Explain workflow: DIGEST via `preview_report_widget`). Do not answer with a plain two-period `query` alone. |
| Usage or business volume next to cost | `suggest_usage_metrics` (with a specific `filterCel`) or `list_metrics`, then add those series to `query`. |
| Cost per unit (cost per user, per request) | `query` with a formula over a cost series and a metric series. |
| Split shared or untagged cost across teams | `virtual-dimensions` workflow (draft, preview, overlap check, publish). |
| A dashboard | `dashboards` workflow; run `suggest_groupby` first for open-ended overviews. |
| A recurring Slack, Teams or email report | `reports` Schedule workflow; ask the design questions before building. |
| A cost or budget alert | `preview_alert`, confirm, then `create_alert`. |
| Deploys or incidents that line up with a change | `list_events` (and `list_alerts`) over the same window; relevance needs matching content, not just matching dates. |

## Step 3: Query well

- **Scope vs split:** `filterCel` narrows what is counted (scope); `groupBy` breaks it down (split). Do not confuse them.
- Prefer `datePreset` (e.g. `LAST_MONTH`, `LAST_WEEK`, `MTD`, `TRAILING_30_DAYS`) over hand-computed `from`/`to`, and never send both. Do not invent preset tokens; use explicit dates if none fits.
- Cost data lags about two days; do not treat yesterday or today as complete billing days.
- Use CEL `== null` for missing labels, never the string `"null"`.
- Keep series `alias` labels at 50 characters or fewer; `name` is a short id (`a`, `b`, ...), not a label.
- Only cite numbers you observed in tool results.

## Step 4: Writes need preview and explicit confirmation

Several Costory tools write to the workspace. Always preview first and get an explicit "yes" before the write:

- `publish_virtual_dimension`: only after `preview_virtual_dimension_draft` (and `virtual_dimension_overlap_matrix` when rules may overlap) and an explicit request to go live. Get approval on the rules themselves, not just the strategy.
- `create_report` with delivery `NOW` or `SCHEDULED`: confirm the design, the channel type (Slack, Teams or email) and the destination first. Call `list_available_destinations` only after the user names the channel type. Use `datePreset` on scheduled reports.
- `create_alert`: show the `preview_alert` result first.
- `create_dashboard` / `update_dashboard`: put the shared period, scope and group-by in `dashboardContext`, not on every widget.
- `create_event` / `update_event`: confirm the annotation text and date.

Write tools act only inside the user's Costory workspace, and query scoping follows the user's workspace role.

## Example prompts

- "Why did our AWS bill jump last week? Which deploys line up with it?"
- "Break down Kubernetes cost by namespace and team for the payments cluster."
- "What is our total AI spend across OpenAI, Anthropic and Bedrock this month, by team and model?"
- "Split our shared Cloud SQL cost across teams by usage and publish it as a dimension."
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."
- "Send the top 5 cost movers to Slack every Monday."

## Troubleshooting

- **"Multiple organizations"** from `get_context`: see Step 1; ask which org and pass its `slug`.
- **`-32602` invalid option on `datePreset`**: the token is not a valid preset; use a listed preset or explicit `from`/`to`.
- **`-32602` `too_big` on `alias`**: shorten the series label to 50 characters or fewer.
- **Empty results**: check the `filterCel` values character for character against `search` results, and confirm the period has data (cost data lags about two days).
- **Auth errors**: reconnect the `costory` MCP server and complete the OAuth login in the browser.

## Support

Docs: https://docs.costory.io/features/mcp · Support: support@costory.io · Privacy: https://www.costory.io/privacy
