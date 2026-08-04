---
name: dashboards
description: "Use when creating or extending Costory dashboards, generating interesting FinOps overviews (suggest_groupby + suggest_usage_metrics + text widgets), adding or replacing widgets, copying widgets between dashboards, editing dashboardContext (global filter / period / groupBy) via update_dashboard, or applying shared-context inheritance for period, groupBy, metric, currency, and CEL filters. Call get_skill with skillId \"dashboards\" before create_dashboard or update_dashboard."
---

# Dashboards

A **DashboardV2** has a shared **`dashboardContext`** input (global theme) and **widgets** that inherit it by default. Each widget is either a **chart** (Advanced Explorer queries + `chartType`) or a **text block** (`type: "text"`, `textContent`).

**Context-first rule:** define the full `dashboardContext` **before** listing widgets. Widgets should contain **only overrides** — never duplicate what already lives in `dashboardContext`. If two or more widgets share the same period, `groupBy`, metric, currency, or filter scope, extract that shared value to `dashboardContext` first.

> **Temporary compatibility:** the deprecated top-level `context` alias is accepted during migration, but always generate `dashboardContext`. Never send both.

## When to Trigger

- Creating a new Costory dashboard
- Generating an **interesting** overview when the user has not specified every widget (use **How to generate interesting dashboards** below)
- Adding, replacing, or removing widgets on an existing dashboard
- Editing `dashboardContext` (global filter, period, groupBy, metric, currency) without recreating
- Copying a widget to another dashboard
- User asks for a cost overview "by service / by region" style dashboard

## Dashboard `dashboardContext` fields

| Field | Role | Widget inheritance |
|-------|------|-------------------|
| `metricId` | Default cost column (e.g. `"cost"`) | Cost widgets omit `metricId` to inherit |
| `currency` | USD, EUR, GBP, CNY | Cost widgets omit `currency` to inherit |
| `groupBy` | Default split dimension — use when most widgets share the same axis | Cost widgets omit `groupBy` to inherit (or set `null` explicitly for totals) |
| `datePreset` **or** `startDate`/`endDate` | Dashboard period — **required when chart widgets are present** (text-only dashboards may omit). Prefer `datePreset` whenever an available preset matches the requested range | Widgets omit `from`/`to`/`datePreset` to inherit |
| `conditionsCel` | Dashboard-wide CEL filter (scope) | AND-merged into every cost widget by default |
| `scopeId` | Optional saved scope | Inherited like other context fields |

Each query still needs `"type": "cost"` (query kind — not inherited). What you omit on cost widgets is `metricId`, `currency`, `groupBy`, and period when they match `dashboardContext`.

## Inheritance semantics

- **Period, metricId, currency, groupBy:** omit on widgets when they match `dashboardContext`.
- **Similarity rule:** when widgets or dashboards are very similar, identify the common `datePreset`/dates, `groupBy`, `metricId`, `currency`, and `conditionsCel` before writing widgets. Put common values in `dashboardContext`; widget fields are for exceptions only.
- **`groupBy`:** omit the field to inherit `dashboardContext.groupBy`; set `groupBy: null` to explicitly disable grouping for that widget.
- **Period presets:** if the user's range maps to an available `DatePreset` (for example `TRAILING_30_DAYS`, `TRAILING_90_DAYS`, `LAST_MONTH`, `YTD`), use `dashboardContext.datePreset`. Use `startDate`/`endDate` only for truly custom ranges.
- **`conditionsCel`:** inherited by default. Every cost widget AND-merges dashboard `conditionsCel` with its own `filterCel`.
- **`extendDashboardConditions`:** defaults to `true` (inherit dashboard filter). Set `false` **only** when the widget must ignore the dashboard filter entirely.
- **Widget `filterCel`:** widget-specific scope only — do **not** put the dashboard-wide filter here; put it in `dashboardContext.conditionsCel`.

## Widget types

Two top-level widget kinds in `create_dashboard` / `update_dashboard`:

| Kind | How to identify | Required fields |
|------|-----------------|-----------------|
| **Chart** | omit `type`, or `"type": "chart"` | `title`, `queries` (each with `type` + `name`), usually `aggBy` |
| **Text** | `"type": "text"` | `title`, `textContent` (markdown or plain text) |

### Chart widget — `queries[].chartType`

Set `chartType` on each series (`cost` / `metric` / `usage` / `formula` / `budget` / `externalMetric`). Omit → **`LINE`**. Primary series for sizing = first non-formula query (else first query).

| `chartType` | Use when | Typical `aggBy` | Notes |
|-------------|----------|-----------------|-------|
| `BAR` | Categorical / stacked spend over time | `Day` / `Week` / `Month` | Default for breakdowns ("by service", "by region") |
| `LINE` | Trends / continuous evolution | `Day` / `Week` / `Month` | MCP default if `chartType` omitted |
| `AREA` | Filled trend (e.g. budget vs spend) | `Day` / `Week` / `Month` | Budget queries often default to AREA in the product |
| `TABLE` | Tabular breakdown / many groups | any | Needs more grid space; also a valid `compare.chartType` |
| `WATERFALL` | Rare as a series `chartType` | — | Prefer enabling comparison via widget `compare` (default comparison render is WATERFALL) |

**Donut (composition for one period):** there is no separate `DONUT` enum. Use `chartType: "BAR"` + widget `aggBy: "Period"` (and a `groupBy`). That is the product donut.

### Time series vs period comparison

These are different modes — pick one per widget based on the question.

| Mode | What it answers | How to configure | Prefer when the user asks… |
|------|-----------------|------------------|----------------------------|
| **Time series** | How cost (or a metric) evolves **within** one period | No `compare`. Series `chartType`: `BAR` / `LINE` / `AREA`. `aggBy`: `Day` / `Week` / `Month` | "trend", "evolution", "over time", "daily/weekly/monthly spend", "seasonality", "when did it spike" |
| **Period comparison** | How this period differs from **another** period (delta / movers) | Widget `compare` (see below). Renders via `compare.chartType` | "vs last month", "WoW / MoM", "what changed", "increase/decrease", "period-over-period", "top movers" |

**Enabling `compare`:**

| Shape | Meaning |
|-------|---------|
| `compare: {}` or `compare: { enabled: true }` | Turn comparison **on**. Omit `from`/`to` → server auto-derives the **preceding period** from the widget/dashboard primary period (same as the UI — **preset-aware**, e.g. `LAST_MONTH` → previous calendar month, not "N days earlier") |
| `compare: { from, to }` | Custom comparison range (both required together) |
| `compare: { …, chartType }` | How comparison renders: `WATERFALL` (default), `TABLE`, or `KPI_BREAKDOWN`. Note: `KPI_BREAKDOWN` / `TABLE` cannot be exported as PNG via `get_dashboard_widget_image` |

Prefer `datePreset` on the dashboard (or widget) + `compare: {}` for "vs previous period". Only pass `compare.from`/`to` for a custom other range.

```json
{
  "title": "MoM by service",
  "queries": [{ "type": "cost", "name": "a", "chartType": "BAR", "groupBy": "cos_service_name" }],
  "aggBy": "Period",
  "compare": {}
}
```

Rules of thumb:

- Default to **time series** for overview dashboards and monitoring (spend over the last 30/90 days).
- Use **comparison** only when the insight is the *change between two ranges*, not the shape of a single range.
- Do **not** add `compare` just to show a BAR/LINE — that switches the widget into comparison mode.
- Donut (`BAR` + `aggBy: "Period"`) is a single-period composition view, not a comparison and not a time series.
- Series `chartType` (`queries[].chartType`) and comparison `compare.chartType` are different enums — `KPI_BREAKDOWN` exists only on `compare.chartType`.

### Text widgets

Notes, headings, or commentary — no queries, period, or dashboard inheritance.

```json
{
  "type": "text",
  "title": "Notes",
  "textContent": "## AWS overview\nKey findings..."
}
```

## Grid sizing (`w` / `h` / `x` / `y`)

Dashboard layout is a **12-column** grid. Rows grow downward.

- **`w`**: width in columns (1–12)
- **`h`**: height in rows
- **`x` / `y`**: optional pin (0-based). On `update_dashboard` `op: "add"`, supply **both** together or the widget is auto-packed at the end. Use values from `get` when copying a layout.

| Widget / chart | Default `w` × `h` | Fits per row (12-col) |
|----------------|-------------------|------------------------|
| `BAR` / `LINE` / `AREA` / `WATERFALL` | **3 × 2** | 4 |
| `TABLE` | **4 × 3** | 3 |
| Text (`type: "text"`) | **6 × 2** | 2 |
| Fallback (empty / unknown) | **4 × 3** | 3 |

**When to override size** (set `w`/`h` explicitly):

| Goal | Suggested `w` × `h` |
|------|---------------------|
| Half-width chart (two side-by-side) | `6 × 3` or `6 × 4` |
| Full-width chart or table | `12 × 3` (chart) / `12 × 4` (table) |
| Compact donut next to a wide bar | donut `3 × 3`, bar `9 × 3` |
| Section heading / short note | text `12 × 1` or `6 × 2` |
| Long markdown commentary | text `12 × 3`+ |

**Layout tips:** omit `w`/`h` for a simple row of equal BAR/LINE charts (four 3×2). Prefer larger `h` for TABLE and multi-series AREA. Pin with `x`/`y` only when placing next to an existing widget or preserving a copied layout.

## Supporting tools

| When | Tool |
|------|------|
| Org context, recent dashboards, popular groupBys | `get_context` |
| Discover CEL field names and values | `search` with `type: ["dimensions"]` |
| Validate filters / explore spend before building | `query` |
| Pick a meaningful default / secondary groupBy | `suggest_groupby` |
| Usage units for a **specific** scope (CPU hours, …) | `suggest_usage_metrics` |
| Read existing dashboard + widget layout | `get` |
| Create a new dashboard | `create_dashboard` |
| Add / replace / remove widgets | `update_dashboard` (Workflow B) |
| Patch shared `dashboardContext` / global filter | `update_dashboard` with `dashboardContext` (Workflow D) |
| Run saved widget data | `get_dashboard_widget_data` |

## How to generate interesting dashboards

Use when the user wants a useful FinOps overview but has **not** listed every widget — e.g. "build a dashboard for AWS", "something interesting for the platform team", "Kubernetes cost overview".

Do **not** ship a wall of identical "by service" BARs. Discover axes with tools, then mix composition, trend, comparison, usage (when scoped), and **text widgets** that explain the story.

Full playbook (layout recipe, anti-patterns, extended examples): call `get_skill` with `skillId: "plugins/costory/skills/dashboards/references/how-to-generate-interesting-dashboards.md"`

### Discovery (required)

1. `get_context` — popular groupBys, recent dashboards, currency habits
2. Lock **scope** as CEL → this becomes `dashboardContext.conditionsCel` (provider, team, service, …)
3. `suggest_groupby` with period `from`/`to` matching the dashboard range + the same `filterCel` as that scope
   - Top hit → `dashboardContext.groupBy` when ≥2 widgets share it
   - Next 1–2 hits → secondary widget `groupBy` overrides
4. If scope is **specific** (not org-wide) → `suggest_usage_metrics` with that `filterCel` → optional usage / unit-economics widgets
5. Optionally `query` once so intro/findings text can cite real drivers (never invent numbers)
6. Draft `dashboardContext` first, then the widget palette below → `create_dashboard`

```json
// suggest_groupby — same filterCel as dashboardContext.conditionsCel
{ "from": "2026-04-18", "to": "2026-07-17", "filterCel": "cos_provider in [\"AWS\"]" }

// suggest_usage_metrics — only with a narrow scope
{ "filterCel": "cos_service_name in [\"AmazonEKS\"]" }
```

### Interesting widget palette

Build **5–8** widgets that answer different questions:

| Slot | Answers | Shape |
|------|---------|-------|
| **Intro (text)** | What is this? Scope, period, how to read | `type: "text"`, usually `w: 12`, `h: 2` |
| **Composition** | Where money goes *now* | Cost + `groupBy`, `chartType: "BAR"`, `aggBy: "Period"` (donut) |
| **Trend** | Evolution | Same/inherited `groupBy`, `BAR`/`LINE`, `aggBy: "Week"` or `"Month"` |
| **Secondary split** | Next useful axis | Override `groupBy` from 2nd `suggest_groupby` hit |
| **What changed** | vs previous period | `compare: {}` (WATERFALL by default) |
| **Usage** | Volume behind cost | `type: "usage"` from `suggest_usage_metrics` (specific scope only) |
| **Unit economics** | Cost per unit | Cost `a` + usage/metric `b` + formula `c`: `"a / b"` |
| **Detail table** | Ranked breakdown | `chartType: "TABLE"`, larger `h` |
| **Findings (text)** | So what? | `type: "text"` — top drivers, risks, next drill-downs (only facts from `query`) |

**Text widgets** turn charts into a story. Always include an **intro**; add **findings** when you inspected data. Put narrative in `textContent`, not stuffed into chart titles.

### Layout sketch (12-col)

1. Intro text — `12 × 2`
2. Composition `4 × 3` + Trend `8 × 3`
3. Secondary split + table (or two bars) — `6 × 3` each
4. Period comparison — `12 × 3`
5. Usage / unit economics if suggested — `6 × 3` each
6. Findings text — `12 × 2`

### Example — interesting AWS overview

```json
{
  "name": "AWS interesting overview",
  "dashboardContext": {
    "conditionsCel": "cos_provider in [\"AWS\"]",
    "groupBy": "cos_service_name",
    "metricId": "cost",
    "currency": "USD",
    "datePreset": "TRAILING_90_DAYS"
  },
  "widgets": [
    {
      "type": "text",
      "title": "How to read this dashboard",
      "textContent": "## AWS cost overview\n\nTrailing 90 days. Default split: **service** (from suggest_groupby).\n\n1. **Composition** — share by service.\n2. **Weekly trend** — evolution of the same split.\n3. **MoM movers** — what changed vs the previous period.",
      "w": 12,
      "h": 2
    },
    {
      "title": "Spend composition by service",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR" }],
      "aggBy": "Period",
      "w": 4,
      "h": 3
    },
    {
      "title": "Weekly trend by service",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR" }],
      "aggBy": "Week",
      "w": 8,
      "h": 3
    },
    {
      "title": "By region",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR", "groupBy": "cos_region" }],
      "aggBy": "Month",
      "w": 6,
      "h": 3
    },
    {
      "title": "Top services (table)",
      "queries": [{ "type": "cost", "name": "a", "chartType": "TABLE" }],
      "aggBy": "Period",
      "w": 6,
      "h": 3
    },
    {
      "title": "MoM movers by service",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR" }],
      "aggBy": "Period",
      "compare": {},
      "w": 12,
      "h": 3
    },
    {
      "type": "text",
      "title": "What to watch",
      "textContent": "## Next checks\n\n1. Drill into the largest service and re-run suggest_groupby under that filter.\n2. For a narrow slice (e.g. EKS), call suggest_usage_metrics and add usage / cost-per-unit widgets.",
      "w": 12,
      "h": 2
    }
  ]
}
```

## Workflow A — Create a new dashboard

1. `get_skill` with `skillId: "dashboards"` (this guide) — you are here
2. `get_context` + optional `search` / `query` to pick dimensions and validate CEL. If the widget list is open-ended, follow **How to generate interesting dashboards** (`suggest_groupby`, optional `suggest_usage_metrics`, text + mixed chart types)
3. **Draft `dashboardContext` first:**
   - `metricId`, `currency`, default `groupBy` when two or more widgets share the same split
   - **Period:** prefer `datePreset` (e.g. `TRAILING_90_DAYS`) when possible; otherwise use `startDate` + `endDate`
   - Global scope: `conditionsCel` when all widgets share a filter
4. **List widgets as minimal overrides** — each chart widget: `title`, `queries` (`type`, `name`, `chartType`, optional `groupBy` / `filterCel`), `aggBy`. Pick `chartType` + size from **Widget types** / **Grid sizing** above. Omit `metricId`, `currency`, period, `groupBy`, and `conditionsCel` when they match `dashboardContext`.
5. `create_dashboard` — include the returned URL in your reply

**Example — AWS overview, by service and by region:**

> **`dashboardContext.metricId` ≠ query `"type"`.** Both can be `"cost"` but mean different things:
> - `dashboardContext.metricId` → which cost **column** to query (inherited — widgets omit `metricId`)
> - `queries[].type` → query **kind** (`"cost"`, `"metric"`, `"formula"`…) — required on every query, not inherited
>
> Widgets below do **not** repeat `metricId`, `currency`, period, `groupBy`, or `conditionsCel` from `dashboardContext`. Only `"type": "cost"` stays on each query.

```json
{
  "name": "AWS Overview",
  "dashboardContext": {
    "conditionsCel": "cos_provider in [\"AWS\"]",
    "groupBy": "cos_service_name",
    "metricId": "cost",
    "currency": "USD",
    "datePreset": "TRAILING_90_DAYS"
  },
  "widgets": [
    {
      "title": "By Service",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR" }],
      "aggBy": "Month"
    },
    {
      "title": "By Region",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR", "groupBy": "cos_region" }],
      "aggBy": "Month"
    }
  ]
}
```

## Workflow B — Extend an existing dashboard

1. `search` → dashboard id
2. `get` → read `context` and each widget's `queryConfig`
3. `update_dashboard` with `op: "add"` (or `replace` / `remove`) — new widgets omit fields that match the dashboard context, especially the period and common `groupBy`
4. Response includes `inheritedContext` — use it to know what widgets can omit
5. Include the returned URL in your reply

**Example — add a region breakdown:**

```json
{
  "dashboardId": "clx9abc",
  "operations": [{
    "op": "add",
    "widget": {
      "title": "By Region",
      "queries": [{ "type": "cost", "name": "a", "chartType": "BAR", "groupBy": "cos_region" }],
      "aggBy": "Month"
    }
  }]
}
```

## Workflow C — Copy a widget to another dashboard

1. `get` on the source dashboard → note `x`, `y`, `w`, `h` and the widget's effective query (resolved `queryConfig`)
2. `get` on the target dashboard → read its `context`
3. `update_dashboard` on the target with `op: "add"`:
   - Pass `x`, `y`, `w`, `h` to preserve layout
   - Rebuild queries using **overrides only** relative to the **target** dashboard context — do not copy inherited metricId/currency/period/filter fields that the target already provides

## Workflow D — Edit dashboard context

Change the shared context through the `dashboardContext` input (global filter, period, `groupBy`, `metricId`, `currency`, `scopeId`) **without** recreating the dashboard or rewriting widgets.

1. `search` → dashboard id (optional if already known)
2. `get` → read current `context` / `conditionsCel`
3. `update_dashboard` with a **partial** `dashboardContext` patch — omit fields you want to keep
4. Response includes updated `inheritedContext`; `applied` is `[]` when no widget ops were passed
5. Include the returned URL in your reply

Patch semantics:

| Input | Effect |
|-------|--------|
| Field omitted | Keep existing value |
| Field provided | Overwrite |
| `conditionsCel: ""` | Clear the dashboard-wide filter |
| `conditionsCel: "<cel>"` | Replace the global filter |

Chart dashboards must keep a resolvable period after the merge — do not clear `datePreset` / dates if chart widgets remain.

**Example — change the global filter only:**

```json
{
  "dashboardId": "clx9abc",
  "dashboardContext": {
    "conditionsCel": "cos_provider in [\"AWS\"]"
  }
}
```

**Example — clear the global filter:**

```json
{
  "dashboardId": "clx9abc",
  "dashboardContext": {
    "conditionsCel": ""
  }
}
```

You can combine `dashboardContext` with `operations` in one call (new/replaced widgets inherit the merged context).

## Per-widget overrides (complete catalog)

Chart widgets inherit `dashboardContext` for shared fields. Set a field on the widget (or query) **only** when it differs from `dashboardContext`. Text widgets ignore all of these (except title / description / layout).

### Widget-level fields

| Field | Inherits from `dashboardContext`? | Default if omitted | Override when… |
|-------|--------------------------|--------------------|----------------|
| `datePreset` | yes | dashboard period | This widget needs a different relative range (e.g. `LAST_MONTH` while dashboard is `TRAILING_90_DAYS`) |
| `from` + `to` | yes (together) | dashboard period | This widget needs a custom absolute range. Prefer `datePreset` when a preset fits. Do not mix with widget `datePreset` |
| `aggBy` | no (widget-only) | `Month` | Granularity: `Hour` / `Day` / `Week` / `Month` / `Period` (Period = donut / single-bucket) |
| `compare` | no | off | Period comparison — prefer `{}` / `{ enabled: true }` to auto-derive previous period from primary `datePreset`; or `{ from, to }` for a custom range; optional `chartType`: `WATERFALL` \| `TABLE` \| `KPI_BREAKDOWN` |
| `extendDashboardConditions` | n/a (controls inheritance) | `true` | Set `false` **only** when this widget must ignore `dashboardContext.conditionsCel` entirely |
| `scopeId` | yes | dashboard `scopeId` | Widget should use a different saved team scope (`list_teams`) |
| `limit` | no | `100` | Need more than 100 groups/rows (max `1000`) |
| `title` / `description` | no | — | Always set `title`; `description` optional |
| `w` / `h` / `x` / `y` | no | auto size / auto pack | See **Grid sizing** |

### Query-level fields (`queries[]`)

| Field | Inherits from `dashboardContext`? | Default if omitted | Override when… |
|-------|--------------------------|--------------------|----------------|
| `type` + `name` | no | — | Always required (`type` = query kind; `name` = single letter `a`/`b`/`c`) |
| `chartType` | no | `LINE` | Choose `BAR` / `LINE` / `AREA` / `TABLE` / `WATERFALL` (see **Widget types**) |
| `groupBy` | yes | `dashboardContext.groupBy` | Different split on this series. Set `null` for an ungrouped total (do not omit if you need to *disable* context groupBy) |
| `metricId` | yes (cost queries) | `dashboardContext.metricId` → `"cost"` | Different cost column (`effective_cost`, etc.) |
| `currency` | yes (cost queries) | `dashboardContext.currency` → `"USD"` | Different currency on this series |
| `filterCel` | no (widget-specific) | none | Extra scope for this series only — AND-merged with dashboard conditions when inheriting (see below) |
| `alias` | no | — | Human label for the series (**max 50 characters**); never put labels in `name` |
| `rollingAggregation` | no | off | Running totals / windowed agg (e.g. MTD budget burn: `aggBy: Day` + `{ aggregator: "SUM", window: { preset: "MONTH" } }`) |

### Conditions inheritance (`extendDashboardConditions` + `filterCel`)

Dashboard filter = `dashboardContext.conditionsCel`. Widget-specific filter = query `filterCel`.

| `extendDashboardConditions` | Effective CEL for cost/usage queries |
|-----------------------------|--------------------------------------|
| omit or `true` (default) | `(dashboardContext.conditionsCel) AND (filterCel)` — if one side is empty, the other applies alone |
| `false` | `filterCel` only — dashboard `conditionsCel` is ignored for this widget |

Examples (dashboard `conditionsCel`: `cos_provider in ["AWS"]`):

```json
// Inherit AWS + narrow to EC2 (effective: AWS AND EC2)
{ "title": "EC2", "extendDashboardConditions": true,
  "queries": [{ "type": "cost", "name": "a", "chartType": "BAR",
    "filterCel": "cos_service_name in [\"AmazonEC2\"]" }], "aggBy": "Month" }

// Ignore dashboard AWS filter — show all providers for this widget
{ "title": "All clouds", "extendDashboardConditions": false,
  "queries": [{ "type": "cost", "name": "a", "chartType": "BAR" }], "aggBy": "Month" }
```

Do **not** repeat the dashboard-wide CEL inside every `filterCel`. Put shared scope in `dashboardContext.conditionsCel`; put exceptions in `filterCel` and/or `extendDashboardConditions: false`.

## QUERY NAMING

`name` is a single letter (`a`, `b`, `c`) for formulas; put human labels in `alias`, never in `name`.

## Safety Rules / Anti-patterns

- Do not repeat `metricId`, `currency`, `groupBy`, `from`/`to`, or `datePreset` on every widget when already in `dashboardContext`
- Do not use widget `from`/`to` for a common dashboard period when a `dashboardContext.datePreset` is possible
- Do not repeat the same `groupBy` across similar widgets instead of setting `dashboardContext.groupBy`
- Do not put the global scope (e.g. `cos_provider in ["AWS"]`) in each widget's `filterCel` instead of `dashboardContext.conditionsCel`
- Do not set `extendDashboardConditions: true` on every widget — omit it (default is inherit)
- Do not call `create_dashboard` without a period in `dashboardContext` — validation rejects it (chart widgets need a dashboard period; text widgets do not)
- Do not generate the deprecated `context` alias or send it together with `dashboardContext`
- Do not skip `suggest_groupby` on open-ended "interesting dashboard" requests and always default to `cos_service_name`
- Do not call `suggest_usage_metrics` without a specific `filterCel`
- Do not ship chart-only dashboards with no text intro / findings when generating an overview
- Do not invent findings numbers — only cite figures observed via `query`

## Related Skills / Next Steps

- `query` — validate scope, inspect drivers before writing findings text, or explore usage / formulas
- `reports` — when the user wants a scheduled DIGEST or delivered report instead of (or in addition to) a live dashboard
- `virtual-dimensions` — when the desired `groupBy` axis does not exist yet and must be built as a custom cost axis
