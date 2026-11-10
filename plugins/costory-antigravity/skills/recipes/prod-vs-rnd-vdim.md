# Prod vs R&D virtual dimension

**When:** *"build a virtual dimension to split production from R&D"*, *"map prod vs research costs"*, *"create an axis for prod / R&D"* — define the custom cost axis, not the report/dashboard that uses it.
**Audience:** FinOps / platform owning allocation rules.
**Outcome:** a published VDIM with **Production** and **R&D** (plus any confirmed intermediate buckets) and acceptable leftover share.

## Tool sequence

1. `get_context` → currency
2. `list_virtual_dimensions` / `search` `{ type: ["virtual_dimensions"], query: "env" }` — reuse an existing env/Prod-R&D VDIM if one already fits
3. Discover mapping signals: `search` `{ type: ["dimensions"], query: "prod" }` / `"rnd"` / `"environment"` / `"account"` → pick strategy (tags, sub-account, project, …)
4. **Present mapping options** → user picks strategy
5. **Present full proposed rules → explicit rule approval** (do not draft until rules are approved)
6. `create_virtual_dimension_draft` with skeleton below
7. `preview_virtual_dimension_draft` `mode: "costs"` → leftover %
8. Iterate: `search` unmapped values → `update_virtual_dimension_draft` (full `rules` array) → preview again
9. Optional: `virtual_dimension_overlap_matrix` if shadowing suspected
10. Explicit user confirm → `publish_virtual_dimension` → wait `computeStatus: COMPLETED` → note `[ENV_BQ_NAME]`

## Payload skeleton

```json
{
  "name": "[Prod vs R&D|Environment]",
  "description": "Split production spend from R&D / non-prod",
  "tagNames": ["finops"],
  "rules": [
    {
      "name": "Production",
      "conditionCel": "[PROD_CEL]",
      "allocation": {
        "allocationType": "dimensionValue",
        "dimensionValue": "Production"
      }
    },
    {
      "name": "R&D",
      "conditionCel": "[RND_CEL]",
      "allocation": {
        "allocationType": "dimensionValue",
        "dimensionValue": "R&D"
      }
    }
  ]
}
```

Frozen: two named buckets **Production** and **R&D** unless the user wants more (e.g. Staging); `allocationType: "dimensionValue"`; **no leftover rule in `rules`** (auto-added). `[PROD_CEL]` / `[RND_CEL]` come from discovery + approved mapping — never invent account/tag values. Pick the final `name` at create — `bqName` is immutable from that name.

## Confirm before build

1. Mapping strategy (tag / sub-account / project / mixed)
2. Exact rule set (CEL + bucket labels) — **explicit approval**
3. Acceptable leftover % from costs preview
4. Publish now vs leave as draft
5. After publish: explore (`prod-vs-rnd-explore`) vs schedule (`env-costs-cto`)

## Gotchas

- Strategy approval ≠ rule approval — wait for the full rule set.
- Cousin of `env-costs-cto` (builds env then schedules an **exec report**) and `prod-vs-rnd-explore` (**query** the split). This card stops at a published axis.
- Always carry rule `id`s forward on `update_virtual_dimension_draft`.
- After publish, `groupBy` / `filterCel` use returned **`bqName`**, not display `name`.

**Brief:** *"VDIM [name] → Production / R&D from [strategy]; leftover [n]% after preview; publish [yes/no] → bqName […]."*

**→ Hand off to `virtual-dimensions`**. Next: `query` (`prod-vs-rnd-explore`) or `reports` (`env-costs-cto`).
