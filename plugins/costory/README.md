# Costory FinOps Agent plugin

This plugin connects Claude Code, Codex, or Cursor to the hosted [Costory FinOps MCP](https://docs.costory.io/features/mcp) (`https://app-api.costory.io/mcp`, OAuth).

Costory is the FinOps agent your team builds on. Claude plus a single provider's MCP works for one person and one cloud. It breaks when you add more providers, more people, and more tools to maintain.

- **Multiple providers:** AWS, GCP, Azure, LLM APIs and data clouds normalized and allocated to teams and features in one model, including allocation from telemetry (Kubernetes, Datadog, dbt).
- **Multiple users:** one hosted setup with OAuth, per-user access and enterprise SSO, instead of local credentials and prompts per person.
- **Maintained for you:** tools, skills and connectors kept up to date by Costory, so reorgs and new services don't break your agent.

Your agent finds savings, explains every change in spend and issues recommendations that your coding agent can implement. It works in Claude, Cursor and any MCP client, plus Slack. Customers include Alan, Deezer, Delivery Hero, Voodoo and Pennylane.

The plugin ships two things:

- **The MCP connection** (`.mcp.json`), so the Costory tools are available right after install. Your client opens a browser window for the OAuth login on first use.
- **Seven skills** that sequence those tools for common FinOps jobs: `cost-change-investigation`, `query`, `virtual-dimensions`, `dashboards`, `reports`, `bigquery`, and `recipes`.

## Requirements

A Costory workspace with billing data connected.

**Pricing:** Starter from $300/month, flat: unlimited users and data sources, and a fair use that 80% of customers never outgrow. Nothing is ever cut off, and agents connected through the MCP use no AI credits. 30-day free trial.

No IAM credentials, Docker, or local server are needed.

## Example prompts

- "Why did our AWS bill go up last week? Check deploys in the same window."
- "Split our shared Kubernetes cost by namespace and publish it as a Team dimension."
- "What is our total AI spend across OpenAI, Anthropic and Bedrock this month, by team?"
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."

## Support

support@costory.io · [Documentation](https://docs.costory.io/features/mcp) · [Privacy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
