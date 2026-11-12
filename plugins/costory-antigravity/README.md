# Costory FinOps Agent

FinOps harness MCP for platform teams. Bills normalized in BigQuery and allocated to teams. Build on it, or try it next to your tool. Read-only on cloud. Starter from $300/month, 30-day free trial.

How Costory works, in five ideas: https://docs.costory.io/get-started/concepts

Moving from another FinOps tool? Try it [next to the tool you have](https://www.costory.io/switch). Asked to build one? [Build on Costory](https://www.costory.io/build-vs-buy), not from scratch.

This plugin connects Antigravity to Costory's hosted FinOps MCP at `https://app-api.costory.io/mcp` (Streamable HTTP, browser OAuth; enterprise SSO on Costory Team and above). No IAM keys, Docker, or local MCP process.

## What you get

- The Costory MCP tools, available after you sign in with your Costory account in the browser.
- Five skills for common FinOps jobs: `query`, `cost-change-investigation`, `virtual-dimensions`, `recipes`, and `bigquery`.

Costory recommends optimizations; your coding agent or your team implements them. It never gets write access to your cloud accounts, never applies automated rightsizing, and never buys Reserved Instances or Savings Plans. Some tools write to your Costory workspace only (alerts, reports, dashboards, events, allocation rules, archives). Preview those first and confirm with your team before applying them. Costory does not promise a savings percentage. Recommendations come with an impact estimate from your own usage. It never charges a percentage of your bill.

## Requirements

A Costory workspace with billing data connected. Starter from $300/month, every skill and the MCP included. App usage (queries, reports, alerts) and Ask Costory have a fair use. You see it in the app, get a warning at 70%, and nothing is ever cut off. Starter fits 80% of our customers. Fair use per plan: Standard (Starter), 3x (Team), 6x (Business), 10x+ (Custom). Above the limit: slowed, never cut off. Agents connected through the MCP use no Ask Costory credits. 30-day free trial, no credit card. Domain auto-join on every plan; enterprise SSO from Team. See [plans](https://docs.costory.io/get-started/plans).

## Example prompts

- Why did our cloud bill change last week, and were there deploys in the same window?
- What is our allocated spend across cloud, AI and data this month, by team?
- Show the top cost drivers for the last 30 days and explain the largest increases.

## Support

support@costory.io · [Concepts](https://docs.costory.io/get-started/concepts) · [Documentation](https://docs.costory.io/features/mcp) · [Privacy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

Customers include Pennylane, Alan, Deezer, Delivery Hero and Voodoo.

License: Apache-2.0.
