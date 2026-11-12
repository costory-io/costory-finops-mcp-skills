# Costory FinOps Agent: Kiro power

FinOps harness MCP for platform teams. Bills normalized in BigQuery and allocated to teams. Build on it, or try it next to your tool. Read-only on cloud. Starter from $300/month, 30-day free trial.

How Costory works, in five ideas: https://docs.costory.io/get-started/concepts

Moving from another FinOps tool? Try it [next to the tool you have](https://www.costory.io/switch). Asked to build one? [Build on Costory](https://www.costory.io/build-vs-buy), not from scratch.

A [Kiro power](https://kiro.dev/docs/powers/) in the [Agent Plugins](https://agent-plugins.org/) format:

- `plugin.json`: manifest and activation keywords (finops, cloud cost, aws cost, kubernetes cost, ...)
- `mcp.json`: the hosted Costory FinOps MCP at `https://app-api.costory.io/mcp` (streamable HTTP, OAuth in the browser)
- `skills/costory-finops/SKILL.md`: condensed FinOps skill (org selection, tool order, preview-then-confirm for write tools); detailed workflows load from the MCP with `get_skill`

## Install

In Kiro: **Powers panel → Add Custom Power → Import power from GitHub**, then enter
`https://github.com/costory-io/costory-finops-mcp-skills/tree/main/powers/costory`.
Or clone the repo and use **Import power from a folder** on `powers/costory`.

Requires a Costory workspace with billing data connected.

**Pricing:** Starter from $300/month, flat: unlimited users and data sources, every skill and the MCP. Pay more only when you use more. Never a % of your bill. App usage (queries, reports, alerts) and Ask Costory have a fair use. You see it in the app, get a warning at 70%, and nothing is ever cut off. Starter fits 80% of our customers. Fair use per plan: Standard (Starter), 3x (Team), 6x (Business), 10x+ (Custom). Above the limit: slowed, never cut off. Agents connected through the MCP use no Ask Costory credits. 30-day free trial. Domain auto-join on every plan; enterprise SSO from Team ([identity and access](https://docs.costory.io/docs/sso-access-control)).

## Support

support@costory.io · [Concepts](https://docs.costory.io/get-started/concepts) · [Documentation](https://docs.costory.io/features/mcp) · [Privacy Policy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
