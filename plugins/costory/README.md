# Costory FinOps Agent plugin

This plugin connects Claude Code, Codex, or Cursor to the hosted [Costory FinOps MCP](https://docs.costory.io/features/mcp) (`https://app-api.costory.io/mcp`, OAuth).

Costory is the FinOps harness for platform teams. Every cloud, data and LLM bill is [normalized into one model on BigQuery](https://docs.costory.io/setup/billing) and allocated to your teams, with [Enrichments](https://docs.costory.io/features/tagging/enrichments) and virtual dimensions you keep in git. The agent explains every change in spend and issues recommendations that your coding agent or your team implements. It works in Claude, Cursor and any MCP client, plus Slack. Customers include Pennylane, Alan, Deezer, Delivery Hero and Voodoo.

How Costory works, in five ideas: https://docs.costory.io/get-started/concepts

Moving from another FinOps tool? Try it [next to the tool you have](https://www.costory.io/switch). Asked to build one? [Build on Costory](https://www.costory.io/build-vs-buy), not from scratch.

What it does not do: it never gets write access to your cloud accounts (MCP actions create Costory objects only: virtual dimensions, alerts, reports, dashboards and events), never changes your code or infrastructure, never applies automated rightsizing, never buys Reserved Instances or Savings Plans, does not promise a savings percentage (recommendations come with an impact estimate from your own usage), and never charges a percentage of your bill.

The plugin ships two things:

- **The MCP connection** (`.mcp.json`), so the Costory tools are available right after install. Your client opens a browser window for the OAuth login on first use.
- **Seven skills** that sequence those tools for common FinOps jobs: `cost-change-investigation`, `query`, `virtual-dimensions`, `dashboards`, `reports`, `bigquery`, and `recipes`.

## Requirements

A Costory workspace with billing data connected.

**Pricing:** Starter from $300/month, flat: unlimited users and data sources, every skill and the MCP. Pay more only when you use more. Never a % of your bill. App usage (queries, reports, alerts) and Ask Costory have a fair use. You see it in the app, get a warning at 70%, and nothing is ever cut off. Starter fits 80% of our customers. Fair use per plan: Standard (Starter), 3x (Team), 6x (Business), 10x+ (Custom). Above the limit: slowed, never cut off. Agents connected through the MCP use no Ask Costory credits. 30-day free trial. Domain auto-join on every plan; enterprise SSO from Team ([identity and access](https://docs.costory.io/docs/sso-access-control)).

No IAM credentials, Docker, or local server are needed.

## Example prompts

- "Why did our AWS bill go up last week? Check deploys in the same window."
- "Split our shared Kubernetes cost by namespace and publish it as a Team dimension."
- "What is our total AI spend across OpenAI, Anthropic and Bedrock this month, by team?"
- "Create an alert if daily BigQuery cost goes above $2,000 and post it to our #finops Slack channel."

## Support

support@costory.io · [Concepts](https://docs.costory.io/get-started/concepts) · [Documentation](https://docs.costory.io/features/mcp) · [Privacy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
