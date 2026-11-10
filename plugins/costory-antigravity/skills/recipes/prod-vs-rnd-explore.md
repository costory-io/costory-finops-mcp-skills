# Prod vs R&D costs (explore)

**When:** *"explain my Prod vs R&D costs"*, *"how much is prod vs research?"*, *"breakdown production vs R&D spend"* — one-shot exploration, not a standing schedule.
**Audience:** whoever needs the split explained now (FinOps, eng lead, finance).
**Outcome:** clear Prod vs R&D totals for a period + optional drivers (service / account) under each bucket.

## Tool sequence

1. `get_context` → currency; note popular groupBys / existing env-like VDIMs
2. Resolve the split axis:
   - Prefer a published Prod/R&D (or env) VDIM → use its **`bqName`** as `groupBy`
   - Else `search` `{ type: ["dimensions", "virtual_dimensions"], query: "prod" }` / `"environment"` / `"rnd"`
   - If no usable axis → hand off to `prod-vs-rnd-vdim` first, then return here
3. Confirm period (default `LAST_MONTH`) + optional scope → `[CONDITIONS_CEL]` / `[SCOPE_ID]`
4. `query` with skeleton below (composition)
5. Optional deepen: filter to one bucket (`filterCel` on `[ENV_BQ_NAME]`) + `groupBy: "cos_service_name"` (or `suggest_groupby`)
6. Optional: `compare: {}` if they ask "what changed"

## Payload skeleton

```json
{
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "Prod vs R&D",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "groupBy": "[ENV_BQ_NAME]",
    "filterCel": "[CONDITIONS_CEL or omit]",
    "chartType": "BAR"
  }]
}
```

**Optional — MoM change on the same axis:**

```json
{
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "compare": {},
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "Prod vs R&D",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "groupBy": "[ENV_BQ_NAME]",
    "chartType": "BAR"
  }]
}
```

Frozen: `groupBy` = published VDIM **`bqName`** (never draft display name); default period `LAST_MONTH`; composition `aggBy: "Period"`. Do not invent Prod/R&D labels from spend alone — the axis must already exist or be built via `prod-vs-rnd-vdim`.

## Confirm before build

1. Which axis = Prod vs R&D (existing VDIM / `cos_environment` / build new)
2. Period (calendar month vs invoice vs YTD)
3. Whole org vs scoped BU
4. Stay in chat vs also schedule a report (`env-costs-cto`)

## Gotchas

- Cousin of `env-costs-cto` (monthly exec **report**) and `prod-vs-rnd-vdim` (build the axis). This card is **explore-only**.
- "R&D" may map to `dev` / `staging` / a custom bucket — confirm labels with the user.
- Always `groupBy` the published `bqName`.

**Brief:** *"Explore Prod vs R&D on [bqName] for [period][, scoped to …][, + MoM compare / service drill-down]."*

**→ Hand off to `query`** (explore). If the axis is missing → `virtual-dimensions` via `prod-vs-rnd-vdim` first.
