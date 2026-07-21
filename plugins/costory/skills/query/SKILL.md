---
name: query
description: "Use when exploring Costory cost, usage, metric, formula, budget, or external-metric data with the query tool — scope vs split (filterCel / groupBy), explorer PoP comparison, CEL discovery, unit economics, budgets. Not for one-shot \"what changed last month\" change trees — those use recipes explain-period-change / reports Explain (DIGEST preview). Call get_skill with skillId \"query\" before non-trivial query work."
---

# Query

`query` is the unified Costory explorer for **cost**, **usage**, **saved metrics**, **external metrics**, **formulas**, and **budgets**. Supporting tools discover dimensions, pick a groupBy, resolve metric IDs, and suggest next steps — call them before guessing field names.

**Load this skill first** for investigation, breakdowns, comparisons, unit economics (cost per X), or budget burn analysis.

## When to Trigger

- Exploring or breaking down cloud costs (by service, region, team, environment, …)
- Explorer-style period comparison (PoP totals / composition) — **not** the “why did spend move?” change-tree explanation
- Cost + usage or cost + business metric (unit economics / formulas)
- Budget vs spend, including month-to-date rolling burn and cumulated budget vs cumulated cost
- K8s waste vs K8s cost (usage metrics + optional waste ratio)
- Effective savings (`contracted_cost` minus `effective_cost`)
- Validating CEL filters before building a dashboard or report
- User asks "what are we spending on X?"
- **After** a DIGEST explain preview: drill into a user-named node with `filterCel`

**Hand off instead of this skill:** *"why did the bill jump?"* / *"what changed last month?"* / *"explain the spike"* → load `recipes` → `explain-period-change` (or `reports` Explain). First answer is `preview_report_widget` DIGEST, not `query` + `compare`.

## Core concepts

### SCOPE vs SPLIT

| Idea | Field | Meaning | Example |
|------|-------|---------|---------|
| **SPLIT** | `groupBy` | How to break the result into series/rows | "Cost **per environment**" → `groupBy: "cos_environment"`, no filter |
| **SCOPE** | `filterCel` | Which rows to include (CEL) | "**EC2** costs" → `filterCel: "cos_service_name in [\"AmazonEC2\"]"`, groupBy something else |
| **Combined** | both | Scoped breakdown | "EC2 **per region**" → filter EC2 + `groupBy: "cos_region"` |

Never put the split dimension into `filterCel` (or the filter into `groupBy`). Discover exact CEL names with `search` (`type: ["dimensions"]`) — Costory labels use a `cos_` prefix (e.g. `cos_service_name`).

### Query naming

- `name` — single lowercase letter (`a`–`z`) for formula references
- `alias` — human label (e.g. `"Cost by environment"`)
- Never put descriptive text in `name`

### Period and aggregation

Provide a period with **either** `datePreset` **or** `from` + `to` (ISO dates, inclusive). Do **not** combine `datePreset` with explicit dates.

| Mode | When |
|------|------|
| `datePreset` | **Preferred** whenever an available `DatePreset` matches (e.g. `TRAILING_30_DAYS`, `TRAILING_90_DAYS`, `LAST_MONTH`, `LAST_WEEK`, `YTD`) |
| `from` + `to` | Truly custom absolute ranges only |

| `aggBy` | Use when |
|---------|----------|
| `Day` / `Week` / `Month` | Evolution / trends within the range |
| `Period` | Single-bucket total or composition for the whole range (good with `groupBy`) |
| `Hour` | Rare; fine-grained short windows |

**Time series vs comparison:** omit `compare` for evolution within one range. For period-over-period deltas, pass `compare` — prefer `datePreset` on the primary period + `compare: {}` (or `{ enabled: true }`) so the server auto-derives the **preceding period** (preset-aware, e.g. `LAST_MONTH` → previous calendar month). Pass `compare: { from, to }` only for a custom other range. Optional `compare.chartType`: `WATERFALL` (default), `TABLE`, or `KPI_BREAKDOWN`.

### Limit

Omit `limit` unless needed (default **100** groups). Raise up to **1000** only for long-tail / full breakdown lists.

## Supporting tools

| When | Tool | Notes |
|------|------|-------|
| Start of conversation / org context | `get_context` | Popular groupBys, recent dashboards, external integrations |
| Discover CEL field names & values | `search` `type: ["dimensions"]` | `query: ""` lists all dimensions; keyword narrows values |
| Find dashboards, budgets, VDIMs, events | `search` (keyword) | Short terms (`"kubernetes"`, not full sentences) |
| Unsure which axis to split by | `suggest_groupby` | Pass the planned period (`from`/`to`) + `filterCel` |
| Saved business metrics | `list_metrics` | Use returned `id` as `metricId` for `type: "metric"` |
| Live Tsuga / BigQuery metrics | `list_metrics` `includeExternal: true` + `search` | Never call `includeExternal` without a search term |
| Infra usage units (CPU hours, …) | `suggest_usage_metrics` | Needs a **specific** `filterCel` scope |
| Budget version ID | `search` → `get` | `search` returns parent budget id; `get` yields `budgetVersionId` for `type: "budget"` |
| Correlate spikes | `list_events` | Same date range as the query |
| Offer next steps | `suggest_actions` | After `query` / `get`; set `hasEvents` / `hasDiff` |

If slug auto-detect fails, call `list_organizations` and pass `slug`.

## Query types

| `type` | Required extras | Discover via |
|--------|-----------------|--------------|
| `cost` | `metricId` (default `"cost"`), `currency` (default `"USD"`) | — |
| `usage` | `metricId` | `suggest_usage_metrics` |
| `metric` | `metricId` | `list_metrics` |
| `externalMetric` | `integrationId`, `metricName`, `aggregator`; BigQuery also needs `dateColumn`, `metricColumn`, `gapFillingMethod` | `list_metrics` `includeExternal: true` |
| `formula` | `formula` referencing letters (`"a / b"`) | — |
| `budget` | `budgetId` = **budget version id** (not parent id from search) | `search` → `get` |

Optional per series: `chartType` (`BAR` \| `LINE` \| `AREA` \| `WATERFALL` \| `TABLE`, default `LINE`), `groupBy`, `filterCel` (cost/usage), `alias`, `rollingAggregation`.

**Cost columns (`metricId`):** `cost`, `effective_cost`, `list_cost`, `contracted_cost`, `unblended_cost`, `net_unblended_cost`, `amortized_cost`, `net_amortized_cost`.

**Null labels:** unlabelled resources are CEL `null` — use `== null` / `!= null`, not `is_null` or the string `"null"`.

**Virtual dimensions:** use immutable `bqName` from list/get VDIM tools as `groupBy` / `filterCel` (not display `name`). After publish, wait until `computeStatus` is `COMPLETED`.

**External metrics:** prefer saved `{ type: "metric" }` when one exists. Tsuga: `metricName` = provider metric; `groupByFields` = attributes; `conditions` = optional provider filter. BigQuery: `provider: "bigquery"`, `metricName` = `project.dataset.table`.

## Workflow A — Standard cost investigation

1. `get_skill` with `skillId: "query"` (this guide)
2. `get_context`
3. Resolve dimensions: `search` with `type: ["dimensions"]` (empty or keyword)
4. If the split axis is unclear → `suggest_groupby` with the planned period + `filterCel`
5. `query` with `type: "cost"`, correct `groupBy` / `filterCel`, period (`datePreset` preferred), `aggBy`
6. Optional: `list_events` for the same range; then `suggest_actions`

**Example — total costs last month:**

```json
{
  "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" }],
  "datePreset": "LAST_MONTH",
  "aggBy": "Month"
}
```

**Example — AWS by service (trailing 90 days composition):**

```json
{
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "AWS by service",
    "metricId": "cost",
    "currency": "USD",
    "groupBy": "cos_service_name",
    "filterCel": "cos_provider in [\"AWS\"]",
    "chartType": "BAR"
  }],
  "datePreset": "TRAILING_90_DAYS",
  "aggBy": "Period"
}
```

**Example — resources without an environment label:**

```json
{
  "queries": [{
    "type": "cost",
    "name": "a",
    "metricId": "cost",
    "currency": "USD",
    "filterCel": "cos_environment == null"
  }],
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Month"
}
```

## Workflow B — Explorer period comparison (PoP totals)

**Not for change-tree explanation.** If the user wants *why spend moved* / *explain last month* / *what drove the delta* as a drivers tree → **stop**. Load `recipes` → `explain-period-change` (or `reports` Explain) and call `preview_report_widget` DIGEST. Do not use this workflow as a substitute.

Use this workflow only for explorer-style period-over-period **numbers** (totals, composition, a single `groupBy` table) when DIGEST is not the ask.

1. Same discovery as Workflow A
2. `query` with a primary period **and** `compare`:
   - Prefer `datePreset` + `compare: {}` (or `{ enabled: true }`) to auto-derive the preceding period
   - Use `compare: { from, to }` only when the other range is a custom window
3. Optional: `list_events` + `suggest_actions` with `hasDiff: true`

**Example — last month vs the preceding period (auto-derived):**

```json
{
  "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" }],
  "datePreset": "LAST_MONTH",
  "compare": {}
}
```

**Example — custom comparison window (explicit dates on both sides):**

```json
{
  "queries": [{ "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" }],
  "from": "2026-07-01",
  "to": "2026-07-17",
  "compare": { "from": "2026-06-01", "to": "2026-06-30" }
}
```

Add `groupBy` (e.g. `cos_service_name`) when you need a flat explorer breakdown — not a DIGEST hierarchy.

## Workflow C — Pick a groupBy when stuck

1. Define scope (`filterCel`) and period
2. `suggest_groupby` with that `from`/`to`/`filterCel`
3. `query` using the suggested dimension as `groupBy`

**Example — EC2 spiked; what to investigate?**

```json
// suggest_groupby
{ "from": "2026-06-01", "to": "2026-06-30", "filterCel": "cos_service_name in [\"AmazonEC2\"]" }

// then query with the suggested groupBy
{
  "queries": [{
    "type": "cost",
    "name": "a",
    "metricId": "cost",
    "currency": "USD",
    "filterCel": "cos_service_name in [\"AmazonEC2\"]",
    "groupBy": "<suggested-dimension>",
    "chartType": "BAR"
  }],
  "datePreset": "LAST_MONTH",
  "aggBy": "Day"
}
```

## Workflow D — Usage alongside cost

1. Narrow scope with `filterCel` (required for useful suggestions)
2. `suggest_usage_metrics` with that `filterCel`
3. `query` with cost (`a`) + usage (`b`)

```json
{
  "queries": [
    { "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" },
    { "type": "usage", "name": "b", "metricId": "k8s_cpu_hours", "alias": "CPU hours" }
  ],
  "datePreset": "LAST_MONTH",
  "aggBy": "Week"
}
```

## Workflow E — Unit economics (cost per metric)

**Saved metric:**

1. `list_metrics` → pick `id`
2. `query` cost + metric + formula

```json
{
  "queries": [
    { "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" },
    { "type": "metric", "name": "b", "metricId": "<metric-id>" },
    { "type": "formula", "name": "c", "alias": "Cost per unit", "formula": "a / b" }
  ],
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Week"
}
```

**Live external (Tsuga) — after `list_metrics` with `includeExternal: true`, `search: "request"`:**

```json
{
  "queries": [
    { "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" },
    {
      "type": "externalMetric",
      "name": "b",
      "provider": "tsuga",
      "integrationId": "<integration-id>",
      "metricName": "<metric-name>",
      "aggregator": "SUM"
    },
    { "type": "formula", "name": "c", "formula": "a / b" }
  ],
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Week"
}
```

**BigQuery revenue table — after `list_metrics` with `includeExternal: true`, `search: "revenue"`:**

```json
{
  "queries": [
    { "type": "cost", "name": "a", "metricId": "cost", "currency": "USD" },
    {
      "type": "externalMetric",
      "name": "b",
      "provider": "bigquery",
      "integrationId": "<integration-id>",
      "metricName": "my-project.analytics.revenue",
      "dateColumn": "event_date",
      "metricColumn": "amount",
      "gapFillingMethod": "ZERO",
      "aggregator": "SUM"
    },
    { "type": "formula", "name": "c", "formula": "a / b" }
  ],
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Week"
}
```

## Workflow F — Budgets

1. `search` with `type: ["budgets"]` (or keyword) → parent budget `id`
2. `get` with that id → read `budgetVersionId` (and align `costMetricId` / `currency` when present)
3. `query` with `type: "budget"` and `budgetId: "<budgetVersionId>"`

**Monthly budget line:**

```json
{
  "queries": [{ "type": "budget", "name": "a", "budgetId": "<budgetVersionId>" }],
  "datePreset": "YTD",
  "aggBy": "Month"
}
```

**Month-to-date burn (which day did we hit the budget?) — use `datePreset: "MTD"` for the current month (`LAST_MONTH` for the previous one):**

```json
{
  "queries": [{
    "type": "budget",
    "name": "a",
    "budgetId": "<budgetVersionId>",
    "rollingAggregation": { "aggregator": "SUM", "window": { "preset": "MONTH" } }
  }],
  "datePreset": "MTD",
  "aggBy": "Day"
}
```

**MTD cost vs MTD budget utilization:**

```json
{
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "metricId": "cost",
      "currency": "USD",
      "rollingAggregation": { "aggregator": "SUM", "window": { "preset": "MONTH" } }
    },
    {
      "type": "budget",
      "name": "b",
      "budgetId": "<budgetVersionId>",
      "rollingAggregation": { "aggregator": "SUM", "window": { "preset": "MONTH" } }
    },
    { "type": "formula", "name": "c", "alias": "Budget utilization", "formula": "a / b" }
  ],
  "datePreset": "MTD",
  "aggBy": "Day"
}
```

**Cumulated budget vs cumulated cost (same rolling window, no ratio — plot both series):**

Use `rollingAggregation` on **both** legs with the same window preset so each day shows a running total within the month (or use `aggBy: "Month"` without rolling for calendar-month buckets). Resolve `budgetVersionId` via `search` → `get`; align `metricId` / `currency` with the budget when `get` returns them.

```json
{
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "Cumulated cost",
      "metricId": "cost",
      "currency": "USD",
      "rollingAggregation": { "aggregator": "SUM", "window": { "preset": "MONTH" } }
    },
    {
      "type": "budget",
      "name": "b",
      "alias": "Cumulated budget",
      "budgetId": "<budgetVersionId>",
      "rollingAggregation": { "aggregator": "SUM", "window": { "preset": "MONTH" } }
    }
  ],
  "datePreset": "YTD",
  "aggBy": "Day"
}
```

## Workflow G — K8s waste vs K8s cost

1. Scope to clusters: `filterCel: "cos_cluster_name != null"` (or a specific cluster in CEL)
2. `suggest_usage_metrics` with that `filterCel` → confirm `k8s_cost` and `k8s_waste` (or org-specific IDs returned by the tool)
3. `query` waste, cost, and optionally a waste ratio

**Example — waste vs allocated K8s cost + waste ratio:**

```json
{
  "queries": [
    {
      "type": "usage",
      "name": "a",
      "alias": "K8s cost",
      "metricId": "k8s_cost",
      "filterCel": "cos_cluster_name != null"
    },
    {
      "type": "usage",
      "name": "b",
      "alias": "K8s waste",
      "metricId": "k8s_waste",
      "filterCel": "cos_cluster_name != null"
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Waste ratio",
      "formula": "b / a"
    }
  ],
  "datePreset": "TRAILING_30_DAYS",
  "aggBy": "Week"
}
```

Add `groupBy` (e.g. `cos_cluster_name` or `cos_namespace_reallocated`) when the user wants waste drivers, not just totals.

## Workflow H — Effective savings (contracted vs effective)

**Effective savings** ≈ on-demand / contracted spend minus what you actually pay after commitments (Savings Plans, RIs, CUDs):

`contracted_cost − effective_cost`

Use the same `filterCel` / `groupBy` on every leg so scope and split stay aligned.

**Example — total effective savings last month:**

```json
{
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "Contracted (without savings)",
      "metricId": "contracted_cost",
      "currency": "USD"
    },
    {
      "type": "cost",
      "name": "b",
      "alias": "Effective cost",
      "metricId": "effective_cost",
      "currency": "USD"
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Effective savings",
      "formula": "a - b"
    }
  ],
  "datePreset": "LAST_MONTH",
  "aggBy": "Month"
}
```

**Example — savings by service (composition for the range):**

```json
{
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "Contracted",
      "metricId": "contracted_cost",
      "currency": "USD",
      "groupBy": "cos_service_name",
      "chartType": "BAR"
    },
    {
      "type": "cost",
      "name": "b",
      "alias": "Effective",
      "metricId": "effective_cost",
      "currency": "USD",
      "groupBy": "cos_service_name"
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Effective savings",
      "formula": "a - b",
      "chartType": "BAR"
    }
  ],
  "datePreset": "LAST_MONTH",
  "aggBy": "Period"
}
```

For credit/discount **runway** reporting (charge-category trend), use the `recipes` skill → `provider-credits` instead.

## Workflow I — From search hit to query

| `search` hit | Next step |
|--------------|-----------|
| Dimension value | `query` with that field in `groupBy` or `filterCel` |
| Dashboard | `get` (or `get_skill` `dashboards` + `update_dashboard` if extending) |
| Budget | `get` → `budgetVersionId` → `query` `type: "budget"` |
| Report | Open/use the report URL; for DIGEST work load `reports` skill |
| Virtual dimension | Use `bqName` in `groupBy`/`filterCel`; load `virtual-dimensions` if editing |

## Post-query

After useful results, consider:

1. `list_events` for the resolved date range (explain spikes)
2. `suggest_actions` — `{ "hasEvents": true|false, "hasDiff": true|false }`
3. Hand off: `dashboards` to persist the view; for a change-tree explanation use `recipes` → `explain-period-change` (not “schedule a DIGEST” as a substitute for preview); `virtual-dimensions` if the needed axis does not exist

## Safety / anti-patterns

- Prefer `datePreset` over hand-computed `from`/`to` when a preset matches — do not freeze last-month/trailing windows as absolute dates
- Do not combine `datePreset` with `from`/`to`
- Do not invent CEL field names — use `search` `type: ["dimensions"]` or `get_context` popular groupBys
- Do not put descriptive labels in `name` — use `alias`
- Do not confuse SCOPE (`filterCel`) with SPLIT (`groupBy`)
- Do not pass the parent budget id from `search` as `budgetId` — resolve `budgetVersionId` via `get`
- Do not call `suggest_usage_metrics` without a specific `filterCel`
- Do not call `list_metrics` with `includeExternal: true` without a `search` term
- Do not set `limit` by default — only when >100 groups are needed
- Do not use string `"null"` for missing labels — use CEL `== null`
- Prefer saved `type: "metric"` over `externalMetric` when a saved metric already exists
- Do not answer *"explain / what changed last month"* with `query` + `compare` — hand off to DIGEST preview (`explain-period-change` / `reports` Explain)

## Related Skills / Next Steps

- `dashboards` — persist a validated query as widgets (`skillId: "dashboards"`)
- `reports` — Explain (DIGEST preview) or scheduled delivery (`skillId: "reports"`)
- `recipes` → `explain-period-change` — one-shot spend-change tree (prefer over Workflow B)
- `virtual-dimensions` — custom cost axis when no dimension fits (`skillId: "virtual-dimensions"`)
- `recipes` — reallocate shared cost by an external / usage metric → `reallocate-by-external-metric`
