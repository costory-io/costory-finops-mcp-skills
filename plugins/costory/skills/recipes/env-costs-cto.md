# Env costs (CTO)

**When:** *"cost per environment for leadership"*, *"CTO wants prod vs non-prod each month"*, *"how much are we burning on dev/test?"* — exec readout, not a drill-down tool.
**Audience:** CTO / VP Eng / Head of Platform (non-FinOps; one clean number per env).
**Outcome:** monthly ranked env costs (biggest items, not symmetric movers) + optional yearly trend for non-prod creep.

## Tool sequence

1. `get_context` → currency
2. **Prerequisite:** `virtual-dimensions` — ordered CEL → prod / staging / dev / … → preview (bulk mapped; unmapped not huge) → publish → wait `computeStatus: COMPLETED` → `[ENV_BQ_NAME]`
3. Confirm scope / optional 12-month graph / channel type → `list_available_destinations`
4. Optional preview of TOP_FLOP
5. Confirm `SCHEDULED` → `create_report`

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
    "groupBy": "[ENV_BQ_NAME]",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL or omit]",
    "scopeId": "[SCOPE_ID or omit]"
  },
  "widgets": [
    {
      "type": "TOP_FLOP",
      "title": "Top environments — last month",
      "queries": [{ "type": "cost", "name": "a", "alias": "Cost by env" }],
      "aggBy": "Period",
      "topN": 10,
      "flopN": 0
    },
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Env cost — last 12 months",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost by env",
        "chartType": "LINE"
      }],
      "datePreset": "LAST_12_MONTHS",
      "aggBy": "Month"
    }
  ],
  "destinations": [{
    "destinationType": "[SLACK|TEAMS|EMAIL]",
    "channelId": "[DESTINATION_ID]"
  }]
}
```

Frozen: `groupBy` = published `env` **`bqName`** (never draft display name); TOP_FLOP `topN: 10`, **`flopN: 0`**; shared period `LAST_MONTH`; cadence MONTHLY; graph preset `LAST_12_MONTHS`. GRAPH widget is **optional** — omit entirely if they don't want the yearly view. No DIGEST / AI unless they later ask "what *changed*".

**Why `env` is special:** providers don't ship a clean prod/staging/dev axis — the VDIM is the real work; the report is trivial once mapping is solid. Don't schedule if preview shows ~40% unmapped.

## Confirm before build

1. VDIM mapping covers the bulk of spend (preview numbers)
2. Scope all-up vs BU
3. Optional 12-month graph yes/no
4. Channel type → destination

## Gotchas

- Exec cousin of `service-cost-weekly` — keep it simple; no service drill-down unless they ask.
- Always groupBy the published `bqName`.

**Brief:** *"Monthly exec env report on published `env` ([bqName]): top-10 last month (no flop) [, + 12-month graph], [scope], to [channel]."*

**→ Hand off to `virtual-dimensions` (build `env`) → `reports` (Schedule).**
