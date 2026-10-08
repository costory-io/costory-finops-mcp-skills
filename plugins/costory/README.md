# Costory FinOps Agent plugin

This plugin turns Claude Code, Codex, or Cursor into a FinOps agent backed by the hosted
[Costory FinOps MCP](https://docs.costory.io/features/mcp) (`https://app-api.costory.io/mcp`, OAuth).
Costory ingests and normalizes cloud, Kubernetes, SaaS, and AI spend (AWS, GCP, Azure, Snowflake,
Datadog, OpenAI, Anthropic), allocates it to teams and features, and exposes it as structured MCP tools.

The plugin ships two things:

- **The MCP connection** (`.mcp.json`), so the Costory tools are available right after install. Your client opens a browser window for the OAuth login on first use.
- **Seven skills** that sequence those tools for common FinOps jobs: `cost-change-investigation`, `query`, `virtual-dimensions`, `dashboards`, `reports`, `bigquery`, and `recipes`.

## Requirements

A Costory workspace with billing data connected. A 14-day trial is available at [costory.io](https://costory.io).
No IAM credentials, Docker, or local server are needed.

## Example prompts

- "Why did our AWS bill go up last week? Check deploys in the same window."
- "Split our shared Kubernetes cost by namespace and publish it as a Team dimension."
- "What is our total AI spend across OpenAI, Anthropic and Bedrock this month, by team?"
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."

## Support

support@costory.io · [Documentation](https://docs.costory.io/features/mcp) · [Privacy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
