# Reallocate cost by external metric

**When:** *"reallocate shared cost by usage"*, *"split the platform bill by requests / CPU / revenue"*, *"unit economics with an external metric"*, *"showback by business driver"* — proportional allocation, not just cost ÷ metric in Explorer.
**Audience:** FinOps / platform / product ops defining fair shared-cost showback.
**Outcome:** a **telemetry virtual dimension** that splits in-scope spend by an external or saved usage metric, after validating unit economics in `query`.

## Tool sequence

1. `get_context` → currency, integrations
2. `list_metrics` `{}` → pick saved metric **or** note none exists yet
3. If no saved metric → `list_metrics` `{ includeExternal: true, search: "[KEYWORD]" }` → `[INTEGRATION_ID]`, `[METRIC_NAME]`, provider fields
4. Define `[SCOPE_CEL]` (shared spend to split — e.g. a cluster, account, or service)
5. `query` — validate **cost per unit** and inspect top `[GROUP_BY]` values (same period as the eventual VDIM)
6. `list_metrics` `{ datasourceId: "[DATASOURCE_ID]" }` → read `groupByDimensions`; pick `[TELEMETRY_DIMENSION]`
7. `query` `{ type: "metric", metricId: "[METRIC_ID]", groupBy: "[TELEMETRY_DIMENSION]" }` → build mapping keys
8. Present proposed mapping for **explicit rule approval**
9. `create_virtual_dimension_draft` / `update_virtual_dimension_draft` with `telemetry` allocation
10. `preview_virtual_dimension_draft` `mode: "costs"` → confirm split + leftover share
11. `publish_virtual_dimension` — only on explicit user confirmation

## Payload skeleton — validate unit economics in `query` first

Prefer saved `{ type: "metric" }` when one exists; otherwise use `externalMetric` for exploration.

**Saved metric:**

```json
{
  "datePreset": "[LAST_MONTH|TRAILING_30_DAYS]",
  "aggBy": "[Week|Month]",
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "In-scope cost",
      "metricId": "cost",
      "currency": "[CURRENCY]",
      "groupBy": "[GROUP_BY]",
      "filterCel": "[SCOPE_CEL]"
    },
    {
      "type": "metric",
      "name": "b",
      "alias": "[METRIC_LABEL]",
      "metricId": "[METRIC_ID]",
      "groupBy": "[GROUP_BY]"
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Cost per unit",
      "formula": "a / b"
    }
  ]
}
```

**Live external (Tsuga) — after `list_metrics` with `includeExternal: true`:**

```json
{
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Week",
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "Shared platform cost",
      "metricId": "cost",
      "currency": "[CURRENCY]",
      "groupBy": "[GROUP_BY]",
      "filterCel": "[SCOPE_CEL]"
    },
    {
      "type": "externalMetric",
      "name": "b",
      "alias": "[METRIC_LABEL]",
      "provider": "tsuga",
      "integrationId": "[INTEGRATION_ID]",
      "metricName": "[METRIC_NAME]",
      "aggregator": "SUM",
      "groupByFields": ["[PROVIDER_ATTRIBUTE]"]
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Cost per unit",
      "formula": "a / b"
    }
  ]
}
```

**`aggregator` accepts only `SUM` \| `AVG` \| `MAX` \| `MIN`** (uppercase) — any other value is rejected with a zod validation error.

**BigQuery table — add `dateColumn`, `metricColumn`, `gapFillingMethod` on the external leg.**

Adjust `[GROUP_BY]` / `groupByFields` so cost and metric break down on the **same axis** you will map in the VDIM (e.g. service, team, product id).

## Payload skeleton — telemetry virtual dimension (after mapping approved)

Single rule with proportional split; unmapped metric values → leftover.

```json
{
  "name": "[VDIM_NAME]",
  "rules": [{
    "name": "Reallocate by [METRIC_LABEL]",
    "conditionCel": "[SCOPE_CEL]",
    "allocation": {
      "allocationType": "telemetry",
      "datasource": "[DATASOURCE_ID]",
      "mappingType": "mapping",
      "mappingParams": {
        "mapping": {
          "[METRIC_VALUE_A]": "[BUCKET_LABEL_A]",
          "[METRIC_VALUE_B]": "[BUCKET_LABEL_B]"
        }
      }
    }
  }]
}
```

Frozen: use `telemetry` allocation (not manual weights); map significant metric values only — long tail stays in leftover; echo advanced allocation fields unchanged on edit per `virtual-dimensions` skill.

## Confirm before build

1. Which shared spend is in scope (`[SCOPE_CEL]`)
2. Which metric drives the split (saved id vs external integration)
3. Which dimension / groupBy aligns cost with metric volume
4. Proposed mapping table (metric value → bucket label) — user approves before draft
5. Acceptable leftover % after `preview_virtual_dimension_draft`
6. Explicit confirmation before `publish_virtual_dimension`

## Gotchas

- **Unit economics in `query` ≠ reallocation.** Cost ÷ metric validates the driver; the VDIM performs proportional **split** of dollars.
- Do not call `list_metrics` with `includeExternal: true` without a search term.
- Mapping keys are metric **value names**, not `cos_*` CEL fields — discover via `query` on the metric with `groupBy`.
- If leftover share stays high after preview, add mapping entries for the largest unmapped values or narrow `[SCOPE_CEL]`.
- After publish, poll `computeStatus` until `COMPLETED` before using `bqName` in dashboards/reports.

**Brief:** *"Validate cost per [metric] on [scope], then publish a telemetry VDIM that reallocates shared spend by [metric] into [buckets] — leftover target [X]%."*

**→ Hand off to `query` (validate unit economics) → `virtual-dimensions` (draft, preview, publish).**
