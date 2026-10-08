# Costory FinOps MCP

You are connected to the Costory FinOps MCP (`costory` server). Costory holds normalized, allocated
cost data (AWS, GCP, Azure, Kubernetes, Snowflake, Datadog, OpenAI, Anthropic) for the user's workspace.

- Call `get_context` first in every conversation to load dimensions, popular group-bys and dashboards.
- Use `query` for cost, usage, metric, budget and formula questions; use `suggest_groupby` before guessing a split.
- To explain a cost change, compare periods with `query`, then pull `list_events` for deploys in the same window.
- Before any write (`create_alert`, `create_report`, `create_dashboard`, `publish_virtual_dimension`), preview first
  (`preview_alert`, `preview_report_widget`, `preview_virtual_dimension_draft`) and confirm with the user.
- Detailed workflows live in `plugins/costory/skills/*/SKILL.md` in this repository; call `get_skill` when available.
