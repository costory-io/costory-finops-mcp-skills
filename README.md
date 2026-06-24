# Costory Plugin: The Premier FinOps MCP & Automation MCP for Claude Code

[![FinOps MCP](https://img.shields.io/badge/MCP-FinOps-blue)](https://costory.io/features/mcp)
[![Automation MCP](https://img.shields.io/badge/MCP-Automation-success)](https://costory.io/features/mcp)

The official **FinOps MCP** (Model Context Protocol) and **Automation MCP** by [Costory](https://costory.io). Learn more about our deep integrations on the [Costory FinOps MCP feature page](https://costory.io/features/mcp).

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

## 💡 Real-World Examples

The true power of this **FinOps MCP** lies in natively bringing **AWS costs to Claude** and enabling seamless **Automation FinOps** workflows. Here is how it looks in practice:

### 1. Analyze AWS Costs with Period Comparisons
Stop logging into billing consoles. Ask Claude to query exactly what you need:
> *"How did our AWS costs change vs last month?"*

**How the MCP responds:**
```json
{
  "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" }],
  "from": "2026-03-01",
  "to": "2026-03-31",
  "compare": { "from": "2026-02-01", "to": "2026-02-28" }
}
```

### 2. Automation FinOps: Advanced Smart Alerts
Set up complex anomaly detection natively within your chat using CEL expressions:
> *"Create a Slack alert if our 7-day rolling AWS cost exceeds $1,000 AND has spiked by more than $1,000 compared to the previous week."*

**How the MCP handles it (`create_alert`):**
```json
{
  "name": "7-Day Cost Spike Alert",
  "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD", "filterCel": "cos_provider in [\"AWS\"]" }],
  "condition": "rollingSum(a, 7, DAY) > 1000 && (rollingSum(a, 7, DAY) - timeShift(rollingSum(a, 7, DAY), 7, DAY)) > 1000",
  "dedup": { "kind": "CALENDAR", "calendarUnit": "WEEK" },
  "notificationChannel": "SLACK",
  "slackChannelId": "C01ABC"
}
```

### 3. Automated Cost Reports via Slack
Put your FinOps reporting on autopilot with scheduled Top/Flop movers:
> *"Send a weekly Slack digest showing our top 10 and bottom 5 cost movers by service, comparing this week to last week."*

**How the MCP handles it (`create_report`):**
```json
{
  "scheduledPeriod": "WEEKLY",
  "widget": {
    "type": "TOP_FLOP",
    "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD", "groupBy": "cos_service_name" }],
    "topN": 10,
    "flopN": 5
  },
  "destinations": [{ "destinationType": "SLACK", "channelId": "C01ABC" }]
}
```

## 🛠️ MCP Tools Reference

This plugin connects to the Costory MCP server, exposing a rich suite of up-to-date FinOps and Automation MCP tools organized by workflow:

### 🔍 Discovery & Context
- `get_context` — fetch the active operational context
- `list_organizations` — list accessible organizations
- `search` — discover dimension values, events, alerts, dashboards, and virtual dimensions

### 📊 Query & Data
- `query` — core data tool for running unified queries for cloud costs, business metrics, usage metrics, budgets, and formulas
- `suggest_groupby` — find the most impactful dimension to split costs by
- `list_metrics` — list available business metric datasources
- `suggest_usage_metrics` — suggests infrastructure usage metric units relevant to a billing scope
- `get` — fetch full resource data by ID (dashboards, budgets, or cost alerts)

### 📈 Dashboards
- `get_dashboard_widget_data` — run a saved dashboard widget and return its data
- `get_dashboard_widget_image` — retrieve a visual image snapshot of a widget
- `create_dashboard` — create a new dashboard with specific widgets and context
- `update_dashboard` — append, replace, or remove widgets on an existing dashboard
- `set_dashboard_tags` — assign descriptive tags to dashboards
- `set_dashboard_team` — assign a dashboard to a specific team

### 🚨 Automation & Alerts
- `list_alerts` — view and manage existing active cost and budget alerts
- `preview_alert` — test alerting conditions on historical data
- `create_alert` — set up FinOps automation alerts for spending thresholds

### 📅 Events
- `list_events` — correlate cost changes with tracked infrastructure events
- `create_event` — annotate cost changes and attach them to a context. *Example: You detect a large spike with your MCP and learn from the data team it was a one-time mislabeled backfill. Attach an event to the cost to remember what happened and add this context to your FinOps memory.*

### 📢 Reports & Notifications
- `list_available_destinations` — discover available Slack channels, Teams channels, and email destinations
- `create_report` — autonomously generate and schedule rich cost reports (daily, weekly, monthly)

### 🏢 Organization Metadata
- `list_teams` — list teams the current user belongs to
- `list_tags` — view all available organization tags
- `delete_tag` — remove a specific tag

### 📚 Documentation
- `search_documentation` — query the internal FinOps knowledge base
- `get_documentation_page` — fetch the full content of a documentation page

### 🧠 Agent Helpers
- `suggest_actions` — receive context-aware follow-up FinOps suggestions based on current investigation context
