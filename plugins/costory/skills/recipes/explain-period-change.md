# Explain period change

**When:** one-shot *"why did the bill jump?"* / *"what changed vs last month?"* / *"break down the spike before the finance review"* — reactive investigation, not a standing schedule.
**Audience:** whoever just saw a number move (FinOps, eng lead, finance).
**Outcome:** a **change tree** that decomposes the delta into chaseable drivers, not just a new total.

## Tool sequence

1. `get_context` → currency
2. Resolve scope → `[CONDITIONS_CEL]` and/or `[SCOPE_ID]` (ask; whole org if none)
3. `suggest_groupby` on chosen period + scope → propose root + 1–2 deeper levels in plain language → confirm → `[ROOT_GROUPBY]`, `[ADDITIONAL_GROUPBY]`
4. `preview_report_widget` with the **preview** skeleton below (chat deliverable) — body is `{ context, widget }` with a **singular** `widget` object (`reportMcpPreviewWidgetBodySchema`). Never pass `widgets` (plural) to preview.
5. Optional: confirm destination type → `list_available_destinations` → `create_report` with `schedule.mode: "NOW"` — only if they explicitly want to share. Create uses `widgets` (plural array).

## Payload skeleton

### `preview_report_widget`

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
  "widget": {
    "type": "DIGEST",
    "queries": [{ "type": "cost", "name": "a", "alias": "Cost change" }],
    "aggBy": "Month",
    "additionalGroupBy": ["[DEEPER_1]", "[DEEPER_2 optional]"],
    "minAbsoluteDiff": 100,
    "minRelativeDiff": 5,
    "topLargestAbsoluteChange": 20
  }
}
```

### `create_report` (only after they ask to send NOW)

Same `context` + the same DIGEST object inside **`widgets`** (plural), plus schedule/destinations:

```json
{
  "visibility": "PRIVATE",
  "schedule": { "mode": "NOW" },
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "[ROOT_GROUPBY]",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[CONDITIONS_CEL or omit]",
    "scopeId": "[SCOPE_ID or omit]"
  },
  "widgets": [
    {
      "type": "DIGEST",
      "queries": [{ "type": "cost", "name": "a", "alias": "Cost change" }],
      "aggBy": "Month",
      "additionalGroupBy": ["[DEEPER_1]", "[DEEPER_2 optional]"],
      "minAbsoluteDiff": 100,
      "minRelativeDiff": 5,
      "topLargestAbsoluteChange": 20
    }
  ],
  "destinations": [{ "destinationType": "[SLACK|TEAMS|EMAIL]", "...": "from list_available_destinations" }]
}
```

Frozen: DIGEST-only; thresholds **100 / 5% / 20**; AI **off** unless they ask — then set `display: "summary"` for the executive narrative and/or `enableAiInvestigation: true` for per-node deep analysis (both slower; both real DIGEST fields). Period default `LAST_MONTH`; use `LAST_INVOICE_MONTH` if they mean invoice close. Comparison auto-derived from the preset — do not invent a second range. Root axis is **`context.groupBy` only**; put deeper levels in `additionalGroupBy` (never put the root there). `aggBy` is **`Month` or `Week`** only for DIGEST.

## Confirm before build

1. Scope + period (calendar vs invoice month)
2. Hierarchy path in plain language (root → deeper) after `suggest_groupby`
3. AI: tree-only vs `display: "summary"` vs deep investigation (`enableAiInvestigation: true`)
4. Headline **total before → after** agreed before digging into the breakdown
5. Stay in chat vs NOW to a channel

## Gotchas

- Don't skip to graph-only — explanation intent → DIGEST.
- Don't invent the tree; let `suggest_groupby` point at it.
- Preview = `widget` (singular). Create/update = `widgets` (array). Mixing them fails Zod strict (`unrecognized_keys`).
- Tune thresholds from preview `recommendations`.

**Brief:** *"One-shot DIGEST on [scope] for [period], tree [root → …], AI [tree / summary / deep], preview in chat [+ NOW to X if asked]."*

**→ Hand off to `reports` (Explain)** — owns preview → optional NOW.
