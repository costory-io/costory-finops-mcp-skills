# Budget vs actual (daily dashboard)

**When:** *"dashboard for budget vs actual per day"*, *"track burn vs budget daily"*, *"are we on pace vs budget?"* — standing dashboard, not a one-shot query.
**Audience:** FinOps / finance / eng lead watching in-month burn.
**Outcome:** a dashboard with day-grain cumulated (or daily) cost vs budget so overspend is visible before month-end.

## Tool sequence

1. `get_context` → currency
2. `search` `{ type: ["budgets"], query: "[BUDGET_KEYWORD]" }` → parent budget `id` (confirm which budget if several)
3. `get` that id → `[BUDGET_VERSION_ID]`, align `metricId` / currency when present
4. Confirm period default (`MTD` for in-month burn; `YTD` for year view) + optional scope
5. Optional: `query` once with the skeleton series to validate numbers before create
6. `create_dashboard` with skeleton below — include returned URL in the reply

## Payload skeleton

```json
{
  "name": "Budget vs actual — daily",
  "dashboardContext": {
    "datePreset": "MTD",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL or omit]",
    "scopeId": "[SCOPE_ID or omit]"
  },
  "widgets": [
    {
      "type": "text",
      "title": "How to read this",
      "textContent": "## Budget vs actual\n\nMonth-to-date. Both series use the same monthly rolling sum so each day shows **cumulated cost** vs **cumulated budget**.",
      "w": 12,
      "h": 2
    },
    {
      "title": "Cumulated cost vs budget (daily)",
      "aggBy": "Day",
      "w": 12,
      "h": 4,
      "queries": [
        {
          "type": "cost",
          "name": "a",
          "alias": "Cumulated cost",
          "chartType": "LINE",
          "rollingAggregation": {
            "aggregator": "SUM",
            "window": { "preset": "MONTH" }
          }
        },
        {
          "type": "budget",
          "name": "b",
          "alias": "Cumulated budget",
          "budgetId": "[BUDGET_VERSION_ID]",
          "chartType": "AREA",
          "rollingAggregation": {
            "aggregator": "SUM",
            "window": { "preset": "MONTH" }
          }
        }
      ]
    },
    {
      "title": "Budget utilization (cost / budget)",
      "aggBy": "Day",
      "w": 12,
      "h": 3,
      "queries": [
        {
          "type": "cost",
          "name": "a",
          "alias": "Cumulated cost",
          "rollingAggregation": {
            "aggregator": "SUM",
            "window": { "preset": "MONTH" }
          }
        },
        {
          "type": "budget",
          "name": "b",
          "alias": "Cumulated budget",
          "budgetId": "[BUDGET_VERSION_ID]",
          "rollingAggregation": {
            "aggregator": "SUM",
            "window": { "preset": "MONTH" }
          }
        },
        {
          "type": "formula",
          "name": "c",
          "alias": "Budget utilization",
          "formula": "a / b",
          "chartType": "LINE"
        }
      ]
    }
  ]
}
```

Frozen: `budgetId` = **`budgetVersionId` from `get`** (never parent id from `search`); both cost and budget legs share the same `rollingAggregation` window; widget `aggBy: "Day"`; default `dashboardContext.datePreset: "MTD"`. Utilization widget is optional — omit if they only want the dual series.

## Confirm before build

1. Which budget (name + version after `get`)
2. MTD vs YTD (or LAST_MONTH for a closed month)
3. Scope: budget's natural scope vs extra `conditionsCel`
4. Utilization ratio widget yes/no

## Gotchas

- Parent budget id from `search` ≠ `budgetId` on the series — always resolve via `get`.
- Daily **without** rolling aggregation shows day spend vs day budget slice — usually wrong for "pace vs budget"; keep the monthly rolling window.
- Align currency / cost metric with the budget when `get` returns them.

**Brief:** *"Daily budget-vs-actual dashboard on [budget name] ([budgetVersionId]), [MTD|YTD][, scoped], cumulated cost + budget [, + utilization]."*

**→ Hand off to `dashboards`** (create). For ad-hoc numbers only → `query` Workflow F.
