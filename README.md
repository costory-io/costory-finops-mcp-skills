# Costory Plugin: The Premier FinOps MCP & Automation MCP for Claude Code

[![FinOps MCP](https://img.shields.io/badge/MCP-FinOps-blue)](https://costory.io/features/mcp)
[![Automation MCP](https://img.shields.io/badge/MCP-Automation-success)](https://costory.io/features/mcp)

The official **FinOps MCP** (Model Context Protocol) and **Automation MCP** by [Costory](https://costory.io). Learn more about our deep integrations on the [Costory FinOps MCP feature page](https://costory.io/features/mcp).

You're likely tired of jumping between Claude and your cloud billing console just to answer basic cost questions. From what I've observed working with engineering teams, context switching kills momentum. That's why we built this Model Context Protocol (MCP) server. It connects your AI assistants directly to AWS, GCP, and Azure billing data.

If you need a FinOps MCP to query real-time infrastructure costs or an Automation MCP to autonomously investigate spending spikes and set up Slack alerts, the Costory plugin equips your AI with the exact financial context it needs.

## Why I recommend this FinOps MCP

- **FinOps Automation at your Fingertips**: Ask your AI to investigate cost spikes, compare historical periods, or configure budget alerts without writing scripts.
- **Context-Aware AI**: As an advanced Automation MCP, it connects LLMs to your business metrics (DAU, requests/day), correlating infrastructure costs directly with business unit economics.
- **Multi-Cloud Support**: I verified native support for AWS, GCP, and Azure cost data.

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
| `cost-investigator` | Autonomous deep-dive Automation MCP agent that drills down across dimensions to find root causes of cost changes |

## Real-World Examples

The true value of this FinOps MCP is natively bringing AWS costs to Claude and enabling seamless Automation FinOps workflows. Based on my testing, here is how it performs in practice:

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

## MCP Tools Reference

This plugin connects to the Costory MCP server, exposing a suite of FinOps and Automation MCP tools organized by workflow:

### Discovery & Context
- `get_context` : Fetch the active operational context
- `list_organizations` : List accessible organizations
- `search` : Discover dimension values, events, alerts, dashboards, and virtual dimensions. *From what I learned, if a developer asks about "Project Phoenix", you can use search to instantly find all associated AWS tags, dashboards, and historical cost events for that specific project.*

### Query & Data
- `query` : Core data tool for running unified queries for cloud costs, business metrics, usage metrics, budgets, and formulas
- `suggest_groupby` : Find the most impactful dimension to split costs by. *For example, I found a cost spike but had no idea what labels or tags were available to drill down. I used `suggest_groupby` to automatically find the correct axes (like team, service, or environment) to guide my investigation.*
- `list_metrics` : List available business metric datasources
- `suggest_usage_metrics` : Suggest infrastructure usage metric units relevant to a billing scope
- `get` : Fetch full resource data by ID (dashboards, budgets, or cost alerts)

### Dashboards
- `get_dashboard_widget_data` : Run a saved dashboard widget and return its data
- `get_dashboard_widget_image` : Retrieve a visual image snapshot of a widget
- `create_dashboard` : Create a new dashboard with specific widgets and context
- `update_dashboard` : Append, replace, or remove widgets on an existing dashboard
- `set_dashboard_tags` : Assign descriptive tags to dashboards
- `set_dashboard_team` : Assign a dashboard to a specific team

### Automation & Alerts
- `list_alerts` : View and manage existing active cost and budget alerts
- `preview_alert` : Test alerting conditions on historical data
- `create_alert` : Set up FinOps automation alerts for spending thresholds. *For instance, I noticed a database's cost slowly creeping up over time. I set a smart alert: "Warn me when the 7-day rolling cost for this database exceeds $1,000 so I can investigate before it gets out of hand."*

### Events
- `list_events` : Correlate cost changes with tracked infrastructure events
- `create_event` : Annotate cost changes and attach them to a context. *When I detected a large spike with the MCP, I learned from the data team it was a one-time mislabeled backfill. I attached an event to the cost to remember what happened and add this context to the FinOps memory.*

### Reports & Notifications
- `list_available_destinations` : Discover available Slack channels, Teams channels, and email destinations
- `create_report` : Autonomously generate and schedule rich cost reports. *When I was working on deprecating a legacy Terraform module, I created a weekly report that sent a chart showing the top cost decreases directly to my Slack so I could track the savings progress.*

### Organization Metadata
- `list_teams` : List teams the current user belongs to
- `list_tags` : View all available organization tags
- `delete_tag` : Remove a specific tag

### Documentation
- `search_documentation` : Query the internal FinOps knowledge base
- `get_documentation_page` : Fetch the full content of a documentation page

### Agent Helpers
- `suggest_actions` : Receive context-aware follow-up FinOps suggestions based on current investigation context
