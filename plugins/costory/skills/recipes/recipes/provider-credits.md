# Provider credits

**When:** *"how many credits are we burning?"*, *"track committed-use / savings-plan discounts"*, *"when do promotional credits run out?"*, *"break down discount lines on the bill"* — net-bill / credit-runway question.
**Audience:** FinOps / Finance (monthly close).
**Outcome:** charge-category trend over ~12 months + last-month movers, so a shrinking credit line is visible before the real invoice arrives.

## Tool sequence

1. `get_context` → currency
2. **Pin scope** with the user → `[CONDITIONS_CEL]` (credits-only vs full charge-category mix). Do not build with an unconfirmed empty filter
3. Confirm channel type → `list_available_destinations`
4. Confirm `SCHEDULED` → `create_report`

## Payload skeleton

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "MONTHLY",
    "firstRunAt": "[ISO_FIRST_RUN]"
  },
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "cos_charge_category",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL — required, confirm]"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Charge category — trailing months",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost by charge category",
        "chartType": "LINE"
      }],
      "datePreset": "LAST_12_MONTHS",
      "aggBy": "Month"
    },
    {
      "type": "TOP_FLOP",
      "title": "Last month charge-category movers",
      "queries": [{ "type": "cost", "name": "a", "alias": "Cost by charge category" }],
      "aggBy": "Period",
      "topN": 10,
      "flopN": 10
    }
  ],
  "destinations": [{
    "destinationType": "SLACK",
    "channelId": "[CHANNEL_ID]"
  }]
}
```

Frozen: `groupBy` = `cos_charge_category`; TOP_FLOP `topN`/`flopN` **10**; shared `LAST_MONTH`; cadence MONTHLY; graph preset `LAST_12_MONTHS`. Scope is **not** frozen — must be confirmed (empty = accidental org-wide).

## Confirm before build

1. **Scope** — pin it; do not build with an unconfirmed empty filter
2. Credits only vs full charge-category mix
3. Channel type → destination

## Gotchas

- Destinations: SLACK/TEAMS use `channelId`; EMAIL uses `{ "destinationType": "EMAIL", "email": "[ADDRESS]" }` — never put an email in `channelId`.
- Charge category is the point — don't re-split by service and lose the credit signal.
- Negative / credit lines can look "weird" in TOP_FLOP; that's expected — explain it.

**Brief:** *"Monthly charge-category report [scoped to …]: 12-month trend + last-month top/flop 10, to [channel]."*

**→ Hand off to `reports` (Schedule).**
