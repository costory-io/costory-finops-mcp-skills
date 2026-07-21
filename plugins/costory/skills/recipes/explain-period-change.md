# Explain period change

**When:** one-shot *"why did the bill jump?"* / *"what changed vs last month?"* / *"break down the spike before the finance review"* — reactive investigation, not a standing schedule.
**Audience:** whoever just saw a number move (FinOps, eng lead, finance).
**Outcome:** a **DIGEST change tree** from `preview_report_widget` that decomposes the delta into chaseable drivers — not explorer tables rebuilt via `query`.

## Tool sequence

1. `get_context` → currency + `popularGroupBys` (fallback hierarchy)
2. Resolve scope → `[CONDITIONS_CEL]` and/or `[SCOPE_ID]` (ask only if ambiguous; **default whole org**)
3. Resolve period → default `LAST_MONTH` (or `LAST_INVOICE_MONTH` if they mean invoice close)
4. `suggest_groupby` on chosen period + scope → pick root + 1–2 deeper levels
   - If suggestions are empty / weak → fall back to `popularGroupBys`, else `cos_provider` → `cos_service_name`
   - **Chat-only:** propose the path in one sentence and **preview in the same turn** (do not wait for hierarchy confirmation)
   - **NOW / delivery:** confirm the path before create
5. `preview_report_widget` with the **preview** skeleton below — **this is the only cost-data tool for the first answer**
6. Present the preview using **Present the preview** below — then stop
7. Optional follow-ups (only if the user asks): drill one node via `query` + `filterCel`, AI summary re-preview, or NOW delivery

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

**Chat-only explain (default):** do **not** block on a long questionnaire. Defaults — whole org, `LAST_MONTH`, tree-only AI off. Ask only when scope/period is ambiguous or they want AI / NOW.

**NOW / channel delivery:** confirm scope, hierarchy, AI, and destination before `create_report`.

## Present the preview (required shape)

After `preview_report_widget` succeeds, the user answer **must** be built only from preview fields (cite them; do not re-fetch):

1. **Headline** — `resolvedPeriod` labels + `totals` (current ← previous, absoluteDelta, relativeDelta)
2. **Largest movers** — `topIncreases` / `topDecreases` (path + Δ + %). Lead with the direction that dominates.
3. **Tree outline** — `rootNodes` (label, Δ, childCount). Do not invent children the preview did not return.
4. **Footer** — thresholds used + optional `explorerUrl`. Note `comparisonPeriodSummary` when present.
5. **Stop.** Offer: drill a named node, re-preview with AI (`display: "summary"`), or send NOW. Do not open a parallel investigation.

Include `summaryMarkdown` only when `display: "summary"`.

## Anti-patterns

- Do **not** call `query` for the first answer (no total, no `groupBy` + `compare`, no daily/monthly series). DIGEST preview already returns totals, comparison period, movers, and tree.
- Do **not** rebuild the explanation as explorer tables or a canvas from parallel `query` calls. DIGEST preview is the deliverable.
- Do **not** skip to GRAPH_SNAPSHOT / TOP_FLOP only — explanation intent → DIGEST.
- Do **not** invent the tree; use `suggest_groupby`, then `popularGroupBys` / `cos_provider` → `cos_service_name` if empty.
- Do **not** wait for hierarchy confirmation on chat-only explain — preview in the same turn after stating the proposed path.
- Preview = `widget` (singular). Create/update = `widgets` (array). Mixing them fails Zod strict (`unrecognized_keys`).
- Tune thresholds from preview `recommendations`, then re-preview — still without `query` unless drilling a node the user named.
- `query` is allowed **only after** the user asks to dig into a specific node/path (then scope with `filterCel`).

**Brief:** *"One-shot DIGEST preview on [scope] for [period], tree [root → …], AI [tree / summary / deep], present preview fields in chat [+ NOW to X if asked]."*

**→ Hand off to `reports` (Explain)** — owns preview → optional NOW.
