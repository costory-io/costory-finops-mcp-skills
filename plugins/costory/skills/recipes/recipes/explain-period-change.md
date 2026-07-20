# Explain period change

**When:** one-shot *"why did the bill jump?"* / *"what changed vs last month?"* / *"break down the spike before the finance review"* — reactive investigation, not a standing schedule.
**Audience:** whoever just saw a number move (FinOps, eng lead, finance).
**Outcome:** a **change tree** that decomposes the delta into chaseable drivers, not just a new total.

## Tool sequence

1. `get_context` → currency
2. Resolve scope → `[CONDITIONS_CEL]` and/or `[SCOPE_ID]` (ask; whole org if none)
3. `suggest_groupby` on chosen period + scope → propose root + 1–2 deeper levels in plain language → confirm → `[ROOT_GROUPBY]`, `[ADDITIONAL_GROUPBY]`
4. `preview_report_widget` with skeleton below (chat deliverable) — preview takes `{ context, widget }` with a **single** widget object, so pass `widgets[0]` as `widget`, not the `widgets` array
5. Optional: confirm destination type → `list_available_destinations` → `create_report` with `schedule.mode: "NOW"` — only if they explicitly want to share

## Payload skeleton

```json
{
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "[ROOT_GROUPBY]",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL or omit]",
    "scopeId": "[SCOPE_ID or omit]"
  },
  "widgets": [{
    "type": "DIGEST",
    "queries": [{ "type": "cost", "name": "a", "alias": "Cost change" }],
    "aggBy": "Month",
    "additionalGroupBy": ["[DEEPER_1]", "[DEEPER_2 optional]"],
    "minAbsoluteDiff": 100,
    "minRelativeDiff": 5,
    "topLargestAbsoluteChange": 20
  }]
}
```

Frozen: DIGEST-only; thresholds **100 / 5% / 20**; AI summary **off** by default (`display` omit/`"tree"`). If they opt in, set `display: "summary"` (warn slower). Period default `LAST_MONTH`; use `LAST_INVOICE_MONTH` if they mean invoice close. Comparison auto-derived — do not invent a second range.

## Confirm before build

1. Scope + period (calendar vs invoice month)
2. Hierarchy path in plain language (root → deeper) after `suggest_groupby`
3. AI summary yes/no
4. Headline **total before → after** agreed before digging into the breakdown
5. Stay in chat vs NOW to a channel

## Gotchas

- Don't skip to graph-only — explanation intent → DIGEST.
- Don't invent the tree; let `suggest_groupby` point at it.
- Tune thresholds from preview `recommendations`.

**Brief:** *"One-shot DIGEST on [scope] for [period], tree [root → …], AI [on/off], preview in chat [+ NOW to X if asked]."*

**→ Hand off to `reports` (Explain)** — owns preview → optional NOW.
