# Costory FinOps Agent: Kiro power

A [Kiro power](https://kiro.dev/docs/powers/) in the [Agent Plugins](https://agent-plugins.org/) format:

- `plugin.json`: manifest and activation keywords (finops, cloud cost, aws cost, kubernetes cost, ...)
- `mcp.json`: the hosted Costory FinOps MCP at `https://app-api.costory.io/mcp` (streamable HTTP, OAuth in the browser)
- `skills/costory-finops/SKILL.md`: condensed FinOps playbook (org selection, tool order, preview-then-confirm for write tools); detailed workflows load from the MCP with `get_skill`

## Install

In Kiro: **Powers panel → Add Custom Power → Import power from GitHub**, then enter
`https://github.com/costory-io/costory-finops-mcp-skills/tree/main/powers/costory`.
Or clone the repo and use **Import power from a folder** on `powers/costory`.

Requires a Costory workspace with billing data connected. A 14-day trial is available at [costory.io](https://www.costory.io).

## Support

support@costory.io · [Documentation](https://docs.costory.io/features/mcp) · [Privacy Policy](https://www.costory.io/privacy) · [Terms](https://www.costory.io/terms)

License: Apache-2.0.
