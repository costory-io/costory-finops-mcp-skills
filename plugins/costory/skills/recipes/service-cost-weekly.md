# Service cost weekly

**When:** *"weekly cost per environment down to service"*, *"weekly breakdown of spend by service"*, *"weekly FinOps pulse for eng"* — operating report broader than the exec-only env recipe.
**Audience:** eng / FinOps teams who need actionable weekly drivers, not a single exec number.
**Outcome:** env (or account) weekly trend + two-level DIGEST (env → service) with **AI narrative on**.

## Tool sequence

1. `get_context` → currency; check whether a usable `env` dimension exists
2. If `env` preferred but missing → hand off `virtual-dimensions` (ordered CEL → preview → publish → wait `computeStatus: COMPLETED`) → return with `[ENV_BQ_NAME]`; or ship now with `cos_sub_account_id`
3. Confirm scope / weekday / Slack channel → `list_available_destinations` (Slack only once type known)
4. `preview_report_widget` on DIGEST (AI on) → tune thresholds
5. Confirm `SCHEDULED` → `create_report`

## Payload skeleton

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "WEEKLY",
    "weekday": "[WEEKDAY]",
    "firstRunAt": "[ISO_FIRST_RUN]"
  },
  "reportContext": {
    "datePreset": "LAST_WEEK",
    "groupBy": "[ROOT_GROUPBY]",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL or omit]",
    "scopeId": "[SCOPE_ID or omit]"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Weekly cost by [env|sub-account]",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost",
        "chartType": "LINE"
      }],
      "datePreset": "TRAILING_14_WEEKS",
      "aggBy": "Week"
    },
    {
      "type": "DIGEST",
      "title": "What moved by service",
      "queries": [{ "type": "cost", "name": "a", "alias": "Cost change" }],
      "aggBy": "Week",
      "additionalGroupBy": ["cos_service_name"],
      "minAbsoluteDiff": 100,
      "minRelativeDiff": 5,
      "topLargestAbsoluteChange": 20,
      "display": "summary",
      "enableAiInvestigation": false
    }
  ],
  "destinations": [{
    "destinationType": "SLACK",
    "channelId": "[SLACK_CHANNEL_ID]"
  }]
}
```

Frozen: shared `LAST_WEEK` (GRAPH overrides to `TRAILING_14_WEEKS` for the trend — without it the graph inherits `LAST_WEEK` and renders a single point); deeper split always `cos_service_name`; DIGEST thresholds **100 / 5% / 20**; **AI summary ON** via `display: "summary"` (exception — call out when confirming; slower). `[ROOT_GROUPBY]` = published `env` `bqName` if available, else `cos_sub_account_id`. Default delivery type Slack — still resolve the concrete channel after type is known.

## Confirm before build

1. Root axis: `env` (build VDIM?) vs `cos_sub_account_id` (ship now)
2. Scope whole-org vs team/BU
3. Weekday + Slack channel
4. `display: "summary"` stays on (slower) — user OK?

## Gotchas

- One-level env alone isn't actionable — keep the service deeper split.
- Don't silently flip AI off; this recipe opts in on purpose (`display: "summary"`).

**Brief:** *"Weekly pulse: graph by [env|sub-account] + DIGEST [root] → service with display summary, every [weekday] to [Slack channel]."*

**→ Hand off to `dashboards` (interactive graph, optional) and/or `reports` (Schedule).**
