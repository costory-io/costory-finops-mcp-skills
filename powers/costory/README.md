# Costory FinOps Agent: Kiro power

FinOps agent MCP: one allocated view of cloud, AI and data costs across providers, shared by your whole team with SSO. Starter $300/mo, 30-day free trial.

A [Kiro power](https://kiro.dev/docs/powers/) in the [Agent Plugins](https://agent-plugins.org/) format:

- `plugin.json`: manifest and activation keywords (finops, cloud cost, aws cost, kubernetes cost, ...)
- `mcp.json`: the hosted Costory FinOps MCP at `https://app-api.costory.io/mcp` (streamable HTTP, OAuth in the browser)
- `skills/costory-finops/SKILL.md`: condensed FinOps skill (org selection, tool order, preview-then-confirm for write tools); detailed workflows load from the MCP with `get_skill`

## Install

In Kiro: **Powers panel → Add Custom Power → Import power from GitHub**, then enter
`https://github.com/costory-io/costory-finops-mcp-skills/tree/main/powers/costory`.
Or clone the repo and use **Import power from a folder** on `powers/costory`.

Requires a Costory workspace with billing data connected.

**Pricing:** Starter from $300/month, flat: unlimited users and data sources, and a fair use that 80% of customers never outgrow. Nothing is ever cut off, and agents connected through the MCP use no AI credits. 30-day free trial.

## Support

support@costory.io · [Documentation](https://docs.costory.io/features/mcp) · [Privacy Policy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
