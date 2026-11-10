# Compute cost drill-down dimension

**When:** *"what dimension can I use to drill down compute costs?"*, *"best groupBy for EC2 / VMs / GCE"*, *"how should I break down compute spend?"* — dimension discovery, not a dashboard build.
**Audience:** FinOps / eng exploring compute drivers.
**Outcome:** a short ranked list of useful drill-down axes for compute, validated with one sample `query`, so the user picks before building further.

## Tool sequence

1. `get_context` → currency, popular groupBys
2. Lock compute **scope** with the user → `[COMPUTE_CEL]` (default below; widen/narrow as needed)
3. `search` `{ type: ["dimensions"], query: "compute" }` (also `"instance"`, `"family"`, `"region"`) — note candidate fields
4. `suggest_groupby` with the planned period + same `filterCel` as `[COMPUTE_CEL]`
5. Present top 3–5 dimensions in plain language (what question each answers) → user picks `[GROUP_BY]`
6. `query` sample breakdown with skeleton below
7. Optional next: hand off to `dashboards` / `reports` / deeper `filterCel` + another `suggest_groupby`

## Payload skeleton

**`suggest_groupby`:**

```json
{
  "from": "[PERIOD_FROM]",
  "to": "[PERIOD_TO]",
  "filterCel": "[COMPUTE_CEL]"
}
```

**Sample drill-down `query`:**

```json
{
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "Compute by [GROUP_BY]",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "groupBy": "[GROUP_BY]",
    "filterCel": "[COMPUTE_CEL]",
    "chartType": "BAR"
  }]
}
```

Default `[COMPUTE_CEL]` (confirm / adjust per cloud mix):

```text
cos_service_name in ["AmazonEC2", "AmazonECS", "AmazonEKS", "Compute Engine", "Virtual Machines"]
```

Frozen: always run `suggest_groupby` **scoped to compute** (empty org-wide suggestions are the wrong recipe); sample query uses `aggBy: "Period"` + BAR. Do not invent dimension names — only propose fields returned by `search` / `suggest_groupby`.

## Confirm before build

1. Compute scope (AWS EC2-only vs multi-cloud compute list above)
2. Period for suggestions + sample
3. Which suggested axis to sample first

## Gotchas

- **SCOPE** = compute filter; **SPLIT** = the drill-down dimension — don't put the service list in `groupBy`.
- Common useful axes (when suggested): instance family / type, region, account / project, environment VDIM, purchase option — still confirm against tool output.
- If they already know the axis and want a standing view → `dashboards` or `reports`, not this card.

**Brief:** *"Compute drill-down: scope [COMPUTE_CEL], suggested axes […], sample on [GROUP_BY] for [period]."*

**→ Hand off to `query`** (Workflow C). After they pick an axis for a standing view → `dashboards` or `reports`.
