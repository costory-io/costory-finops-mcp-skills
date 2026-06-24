# Costory Plugin: The Premier FinOps MCP & Automation MCP for Claude Code

[![FinOps MCP](https://img.shields.io/badge/MCP-FinOps-blue)](https://costory.io)
[![Automation MCP](https://img.shields.io/badge/MCP-Automation-success)](https://costory.io)

The official **FinOps MCP** (Model Context Protocol) and **Automation MCP** by [Costory](https://costory.io). 

Bring intelligent cloud cost management, automated insights, and FinOps automation directly into Claude Code. This Model Context Protocol (MCP) server seamlessly connects your AI assistants to AWS, GCP, and Azure billing data, transforming how teams manage cloud spend.

Whether you are looking for a **FinOps MCP** to query real-time infrastructure costs, or an **Automation MCP** to autonomously investigate spending spikes and set up Slack alerts, the Costory plugin equips your AI with deep financial context.

## 🚀 Why Use This FinOps MCP?

- **FinOps Automation at your Fingertips**: Ask your AI to investigate cost spikes, compare historical periods, or configure budget alerts without writing a single line of script.
- **Context-Aware AI**: As an advanced Automation MCP, it connects LLMs to your business metrics (DAU, requests/day), correlating infrastructure costs directly with business unit economics.
- **Multi-Cloud Support**: Native support for AWS, GCP, and Azure cost data.

## Installation

Install the Costory FinOps MCP plugin via the Claude Code marketplace:

```bash
/plugin marketplace add costory-io/costory-plugin
/plugin install costory@costory-marketplace
```

## Setup

After installing, run `/mcp` in Claude Code and follow the browser OAuth flow to log in with your Costory account. This connects your local automation MCP environment to your cloud infrastructure.

## FinOps Skills

Easily trigger built-in FinOps automation skills:

| Skill | Description |
|-------|-------------|
| `/costory:analyze-costs` | Analyze cloud costs by service, team, environment, or any dimension |
| `/costory:investigate-change` | Investigate cost spikes and compare periods |
| `/costory:query-metric` | Query custom business metrics (DAU, requests/day, etc.) |
| `/costory:setup-alert` | Create cost alerts with Slack notifications |
| `/costory:send-report` | Send cost reports to Slack channels |

## Autonomous FinOps Agents

| Agent | Description |
|-------|-------------|
| `cost-investigator` | Autonomous deep-dive Automation MCP agent — drills down across dimensions to find root causes of cost changes |

## MCP Tools Reference

This plugin connects to the Costory MCP server, exposing the following FinOps and Automation MCP tools to Claude and other LLMs:

- `search` — discover dimensions, dashboards, views, events, metrics
- `query_costs` — query cost data grouped by dimensions
- `query_metric` — query custom business metrics
- `list_metrics` — list available metric datasources
- `get_cost_diff` — compare costs between periods
- `get` — fetch dashboard/view/explorer configs
- `suggest_groupby` — find the best dimension to split by
- `list_events` — correlate cost changes with events
- `create_event` — annotate cost changes
- `create_saved_view` — persist queries as reusable views
- `create_alert` — set up FinOps automation alerts
- `list_alerts` — view existing alerts
- `list_slack_channels` — discover Slack channels for reporting
- `send_to_slack` — send rich cost reports to Slack
- `suggest_actions` — get context-aware follow-up FinOps suggestions
- `list_organizations` — list accessible organizations
