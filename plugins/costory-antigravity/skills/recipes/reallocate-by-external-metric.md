# Reallocate cost by external metric

**When:** *"reallocate shared cost by usage"*, *"split the platform bill by requests / CPU / revenue"*, *"unit economics with an external metric"*, *"showback by business driver"* — proportional allocation, not just cost ÷ metric in Explorer.
**Audience:** FinOps / platform / product ops defining fair shared-cost showback.
**Outcome:** a **telemetry virtual dimension** that splits in-scope spend by a live external-metric integration, after validating unit economics in `query`.

## Tool sequence

1. `get_context` → currency, integrations
2. `list_metrics` `{ includeExternal: true, search: "[KEYWORD]" }` → `[PROVIDER]`, `[INTEGRATION_ID]`, `[METRIC_NAME]`, split-by attribute from `attributes`
3. Define `[SCOPE_CEL]` (shared spend to split — e.g. a cluster, account, or service)
4. `query` — validate **cost per unit** with `{ type: "externalMetric", ... }` and inspect top series keys (same period as the eventual VDIM)
5. Present proposed mapping for **explicit rule approval**
6. `create_virtual_dimension_draft` / `update_virtual_dimension_draft` with `telemetry` + inline `externalMetric` (do not set `datasource`)
7. `preview_virtual_dimension_draft` `mode: "costs"` → confirm split + leftover share
8. `publish_virtual_dimension` — only on explicit user confirmation

## Payload skeleton — validate unit economics in `query` first

Use `{ type: "externalMetric" }` for the metric leg — new telemetry reallocations are integration-backed.

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
      "externalMetric": {
        "provider": "[PROVIDER]",
        "integrationId": "[INTEGRATION_ID]",
        "metricName": "[METRIC_NAME]",
        "aggregator": "SUM",
        "groupByFields": ["[PROVIDER_ATTRIBUTE]"]
      },
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
2. Which live integration metric drives the split (`provider` / `integrationId` / `metricName` / split-by)
3. Which dimension / groupBy aligns cost with metric volume
4. Proposed mapping table (metric value → bucket label) — user approves before draft
5. Acceptable leftover % after `preview_virtual_dimension_draft`
6. Explicit confirmation before `publish_virtual_dimension`

## Gotchas

- **Unit economics in `query` ≠ reallocation.** Cost ÷ metric validates the driver; the VDIM performs proportional **split** of dollars.
- Do not call `list_metrics` with `includeExternal: true` without a search term.
- Mapping keys are metric **value names**, not `cos_*` CEL fields — discover via `query` `{ type: "externalMetric", groupByFields }`.
- Do not set `datasource` on the VDIM telemetry allocation — persist inline `externalMetric` only.
- If leftover share stays high after preview, add mapping entries for the largest unmapped values or narrow `[SCOPE_CEL]`.
- After publish, poll `computeStatus` until `COMPLETED` before using `bqName` in dashboards/reports.

**Brief:** *"Validate cost per [metric] on [scope], then publish a telemetry VDIM that reallocates shared spend by [metric] into [buckets] — leftover target [X]%."*

**→ Hand off to `query` (validate unit economics) → `virtual-dimensions` (draft, preview, publish).**
