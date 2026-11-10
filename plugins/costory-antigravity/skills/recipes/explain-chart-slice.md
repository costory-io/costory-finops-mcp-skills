# Explain a clicked chart slice

**When:** the user clicked a bar or point on a Costory chart and asked to explain **that slice** — "explain this bar", "cost evolution of this slice", "why is this point this high". The user message names the slice; it does not carry the full dashboard filter.
**Audience:** whoever is looking at the chart (FinOps, eng, finance).
**Outcome:** a short explanation of how `[DIMENSION]=[SLICE]` moved over `[PERIOD]` on `[METRIC]`, then — only after they agree — a dashboard that shows the diff, the time series, and a usage correlation if one exists.

## Tool sequence

1. `get_context` → currency, popular groupBys, integrations.
2. Read the clicked slice from the user message: `[DIMENSION]`, `[SLICE]`, `[PERIOD]`, `[METRIC]` (default metric `cost`), and the formatted value if present. Do not ask them to paste `filtersCEL`.
3. Recover the rest of the scope from the live page, not from a dumped CEL string:
   - **Dashboard:** `pageUrl` contains the dashboard id (path `/dashboards/<id>`). `get` that id and use its `dashboardContext` (`conditionsCel`, `scopeId`, period, metric, currency).
   - **Explore:** when `pageForm` is present, use its metric, groupBy, period, and conditions.
   - **Team scope** comes from that dashboard or Explore `scopeId`. Do not rebuild team-scope `in ["id", …]` lists yourself.
   - If the user message already includes a `conditionsCel` / `filtersCEL` (clipboard / external client), use it as the base scope and still add the slice filter below.
4. Build `[SLICE_CEL]`: the recovered conditions **plus** a filter for the click, `[DIMENSION] == "[SLICE]"` (or `in ["[SLICE]"]` when the value is a list). This is the only scope for every call below.
5. `query` the slice twice:
   - **Diff** — `[PERIOD]` with `compare: {}` (auto previous period) so current vs previous is visible.
   - **Time series** — same `[SLICE_CEL]` and `[PERIOD]`, `aggBy` matching the chart grain (else `Day`), so the evolution is visible.
6. `suggest_groupby` with the same period and `[SLICE_CEL]`. Propose one better split; `query` it only after they pick, or immediately if there is a single obvious axis.
7. `suggest_usage_metrics` with `[SLICE_CEL]`. If a metric clearly tracks the same scope, `query` cost (`a`) + usage (`b`) on `[PERIOD]`. Skip if suggestions are empty.
8. Explain in chat from those results (headline delta, series shape, suggested split, usage correlation). Then offer a dashboard. Call `create_dashboard` or `update_dashboard` **only after they agree** — discuss the widgets first.

## Payload skeleton

**Diff (`query`):**

```json
{
  "datePreset": "[PERIOD]",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "[SLICE] [METRIC]",
    "metricId": "[METRIC]",
    "currency": "[CURRENCY]",
    "filterCel": "[SLICE_CEL]"
  }],
  "compare": {}
}
```

**Time series (`query`):**

```json
{
  "datePreset": "[PERIOD]",
  "aggBy": "[GRAIN]",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "[SLICE] over time",
    "metricId": "[METRIC]",
    "currency": "[CURRENCY]",
    "filterCel": "[SLICE_CEL]",
    "groupBy": "[SUGGESTED_GROUP_BY]",
    "chartType": "BAR"
  }]
}
```

Use explicit `from` / `to` (and `compare: { from, to }` on the diff) when `[PERIOD]` is not a `datePreset` token. Omit `groupBy` on the time series until `suggest_groupby` returns an axis.

**Usage correlation (`query`), only when `suggest_usage_metrics` returns a match:**

```json
{
  "datePreset": "[PERIOD]",
  "aggBy": "[GRAIN]",
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "[SLICE] cost",
      "metricId": "[METRIC]",
      "currency": "[CURRENCY]",
      "filterCel": "[SLICE_CEL]"
    },
    {
      "type": "usage",
      "name": "b",
      "alias": "[USAGE_METRIC_LABEL]",
      "metricId": "[USAGE_METRIC_ID]",
      "filterCel": "[SLICE_CEL]"
    }
  ]
}
```

**Dashboard (only after confirm).** New board via `create_dashboard`, or `update_dashboard` on the dashboard id from `pageUrl` when they want the widgets added there. Same series as above: diff, time series, optional usage. Period, metric, currency, and `[SLICE_CEL]` live on `dashboardContext` — do not repeat them on every widget.

Frozen: tool name is `query` (not `queryCost`). `[METRIC]` defaults to `cost`. Do not invent `[DIMENSION]` or `[SLICE]`; they come from the user message.

## Confirm before build

**Chat explanation (default):** do not block on a questionnaire. Slice, period, and metric are already in the user message; scope comes from the page.

**Dashboard:** confirm before `create_dashboard` / `update_dashboard` — new dashboard vs update the one they are viewing, and which suggested split (if several) to chart.

## Gotchas

- Standing "why did the bill jump?" with **no clicked slice** → `explain-period-change` (DIGEST preview). This card is the slice they clicked, and it uses `query`, not a DIGEST tree.
- Do not paste or request the full dashboard `filtersCEL`. Team scope is `scopeId` on the dashboard / Explore context.
- `suggest_groupby` and `suggest_usage_metrics` must use `[SLICE_CEL]`. Org-wide suggestions answer a different question.
- Discuss the dashboard before mutating it. A chat explanation is a complete answer on its own.

**Brief:** *"Explain [DIMENSION]=[SLICE] ([formatted value]) on [METRIC] for [PERIOD]; diff + time series under [SLICE_CEL]; optional usage correlation; dashboard only after they agree."*

**→ Hand off to `query`** (Workflow B for the diff, Workflow A/C for the series and split, Workflow D for usage). **→ Hand off to `dashboards`** only after they confirm a board.
