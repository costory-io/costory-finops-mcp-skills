# Marketplace spend by vendor

**When:** *"marketplace / private-offer spend by vendor"*, *"AWS Marketplace bill by seller"*, *"which third-party vendors are on our cloud invoice?"* — SaaS-through-cloud that Finance must reconcile to POs/renewals.
**Audience:** Finance / FinOps / procurement.
**Outcome:** monthly vendor ranking + ~12-month trend of marketplace-only spend (native EC2/compute never in the chart).

## Tool sequence

1. `get_context` → currency
2. Confirm cadence monthly + channel type → `list_available_destinations`
3. Confirm `SCHEDULED` → `create_report`

## Payload skeleton

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "MONTHLY",
    "firstRunAt": "[ISO_FIRST_RUN]"
  },
  "reportContext": {
    "datePreset": "LAST_MONTH",
    "groupBy": "cos_invoice_issuer",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "cos_marketplace_purchase == true"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Marketplace spend by vendor",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Marketplace cost",
        "chartType": "LINE"
      }],
      "datePreset": "LAST_12_MONTHS",
      "aggBy": "Month"
    },
    {
      "type": "TOP_FLOP",
      "title": "Top marketplace vendors — last month",
      "queries": [{ "type": "cost", "name": "a", "alias": "Marketplace cost" }],
      "aggBy": "Period",
      "topN": 10,
      "flopN": 0
    }
  ],
  "destinations": [{
    "destinationType": "[SLACK|TEAMS|EMAIL]",
    "channelId": "[DESTINATION_ID]"
  }]
}
```

Frozen: `conditionsCel` = `cos_marketplace_purchase == true` (non-negotiable); `groupBy` = `cos_invoice_issuer`; TOP_FLOP `topN: 10`, **`flopN: 0`**; shared `LAST_MONTH`; cadence MONTHLY; graph preset `LAST_12_MONTHS`.

## Confirm before build

1. Scope stays marketplace-only (don't broaden)
2. Cadence monthly is fine
3. Channel type → destination

## Gotchas

- Marketplace lines often lack your tags — don't try to fix that here; isolate them.
- Splitting by `cos_service_name` buries vendors; issuer is the Finance question.

**Brief:** *"Monthly marketplace-only report: trend by invoice issuer over ~12 months + top-10 vendors last month, delivered to [channel]."*

**→ Hand off to `reports` (Schedule).**
