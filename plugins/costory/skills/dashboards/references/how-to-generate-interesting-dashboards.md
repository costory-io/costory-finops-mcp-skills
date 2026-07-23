# How to generate interesting dashboards

Use this playbook when the user wants a **useful FinOps dashboard** but has not specified every widget — e.g. "build me a dashboard for AWS", "overview for the platform team", or "something interesting for Kubernetes costs".

Do **not** ship a wall of identical "by service" BAR charts. Mix composition, trend, comparison, usage (when scoped), and **text widgets** that explain what the reader should look at.

## Discovery before widgets

Always discover axes and metrics from the org — do not guess.

1. `get_context` — popular groupBys, recent dashboards, currency / org habits
2. Lock the **scope** (what to include) as CEL — this becomes `dashboardContext.conditionsCel`
3. Resolve exact dimension names with `search` (`type: ["dimensions"]`) if needed
4. `suggest_groupby` with the planned period (`from`/`to` matching the dashboard range) + the same `filterCel` as the scope → primary and secondary split axes
5. If the scope is **specific** (service, product, cluster, team, …) → `suggest_usage_metrics` with that `filterCel` → candidate usage series
6. Optionally `query` once to validate that the scope and top groupBy return meaningful spend before `create_dashboard`

### `suggest_groupby`

Call when the split axis is unclear or you want the most discriminative dimension for this scope.

```json
{
  "from": "2026-04-18",
  "to": "2026-07-17",
  "filterCel": "cos_provider in [\"AWS\"]"
}
```

- Use the **top suggestion** as `dashboardContext.groupBy` when most widgets share that axis
- Use the next 1–2 suggestions as **widget overrides** (`queries[].groupBy`) for secondary breakdowns
- Pass the **same** `filterCel` you will put in `dashboardContext.conditionsCel`

### `suggest_usage_metrics`

Call only with a **narrow** `filterCel`. Broad scopes (entire org / all providers) return unhelpful units.

```json
{
  "filterCel": "cos_service_name in [\"AmazonEKS\"]"
}
```

- Add 1–2 usage chart widgets (`queries[].type: "usage"`, `metricId` from suggestions)
- When cost + usage both exist for the same scope, prefer a **unit-economics** widget (cost + usage + formula `a / b`) over a second raw usage line

## Interesting widget palette

Build 5–8 widgets from this menu. Prefer **variety of questions**, not variety of chart skins.

| Slot | Question it answers | Widget shape |
|------|---------------------|--------------|
| **Intro (text)** | What is this dashboard? Scope, period, who it is for | `type: "text"` — markdown overview + how to read the charts below |
| **Composition** | Where does money go *now*? | Cost + suggested `groupBy`, `chartType: "BAR"`, `aggBy: "Period"` (product donut) |
| **Trend** | How is spend evolving? | Same or inherited `groupBy`, `BAR` or `LINE`, `aggBy: "Week"` or `"Month"` |
| **Secondary split** | What is the next useful axis? | Override `groupBy` with the 2nd `suggest_groupby` hit (region, env, account, …) |
| **What changed** | What moved vs last period? | Cost widget with `compare: {}` (auto previous period); optional `compare.chartType: "WATERFALL"` |
| **Usage** | Is cost driven by volume? | `type: "usage"` from `suggest_usage_metrics` — only when scope was specific |
| **Unit economics** | Cost per unit of work | Cost (`a`) + usage/metric (`b`) + formula (`c`: `"a / b"`) |
| **Detail table** | Full ranked breakdown | Cost + `groupBy`, `chartType: "TABLE"`, larger `h` |
| **Findings (text)** | So what? | `type: "text"` — 3–5 bullets: top drivers, anomalies to watch, suggested next drill-downs |

### Text widgets — when and what to write

Text is what turns charts into a **story**. Add at least one intro; add a findings block when you already inspected data via `query`.

**Intro text** (place first, often `w: 12`, `h: 2`):

- Dashboard purpose in one sentence
- Scope in plain language + the CEL (or dimension values) applied
- Period (`datePreset`) and default split axis
- One line on how to read the first two charts

**Section / bridge text** (optional, `w: 12`, `h: 1`):

- Short heading between composition and comparison, e.g. "Period-over-period"

**Findings text** (place near the end, `w: 12`, `h: 2`–`3`):

- Top 2–3 cost drivers (names + share or $, if known from `query`)
- One risk / spike / unlabeled bucket if relevant
- Suggested next actions ("Filter to service X", "Open Advanced Explorer by account", …)
- Do **not** invent numbers — only cite figures you observed from `query` / widget preview tools

```json
{
  "type": "text",
  "title": "How to read this dashboard",
  "textContent": "## AWS cost overview\n\nScope: all AWS spend. Period: trailing 90 days. Default split: service.\n\n1. **Composition** — share of spend by service for the whole period.\n2. **Trend** — weekly evolution of the same split.\n3. **MoM movers** — what changed vs the previous period.",
  "w": 12,
  "h": 2
}
```

## Recommended layout (12-col grid)

| Row | Widgets | Suggested sizes |
|-----|---------|-----------------|
| 1 | Intro text | `12 × 2` |
| 2 | Composition (donut) + Trend | `4 × 3` + `8 × 3` |
| 3 | Secondary split + Detail table *or* two secondary bars | `6 × 3` + `6 × 3` (or table `12 × 4`) |
| 4 | Period comparison (`compare: {}`) | `12 × 3` |
| 5 | Usage and/or unit economics (if suggested) | `6 × 3` + `6 × 3` |
| 6 | Findings text | `12 × 2` |

Omit rows that do not apply (no usage suggestions → skip row 5). Prefer fewer strong widgets over many near-duplicates.

## Context-first checklist

Before calling `create_dashboard`:

- [ ] `dashboardContext.conditionsCel` = discovery scope
- [ ] `dashboardContext.groupBy` = top `suggest_groupby` result (when shared by ≥2 widgets)
- [ ] `dashboardContext.datePreset` preferred over frozen dates
- [ ] `dashboardContext.metricId` + `dashboardContext.currency` set once
- [ ] Chart widgets omit inherited fields; only overrides remain
- [ ] At least one **text** widget explains the dashboard
- [ ] Chart types mix composition / trend / comparison — not five identical BARs

## Anti-patterns

- Skipping `suggest_groupby` and always defaulting to `cos_service_name`
- Calling `suggest_usage_metrics` on a broad org-wide filter
- Dashboard with only cost BARs and no text / no comparison / no secondary axis
- Duplicating the same `groupBy` + `aggBy` on every widget
- Putting narrative into chart `title` instead of a text widget
- Inventing "findings" numbers without a prior `query`

## Full example — interesting AWS overview

After `get_context`, `suggest_groupby` (AWS scope → e.g. `cos_service_name`, then `cos_region`), and optional `query` for intro/findings numbers:

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
      "textContent": "## AWS cost overview\n\nTrailing 90 days of AWS spend. Default split: **service** (from suggest_groupby).\n\n- Start with **composition** (share) and **weekly trend**.\n- Use **by region** for geography.\n- **MoM by service** highlights what changed vs the previous period.",
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
      "textContent": "## Next checks\n\n1. Open the largest service from composition and re-run suggest_groupby under that filter.\n2. If unlabeled environment spend is large, consider a virtual dimension or tagging push.\n3. For a cluster/service-heavy slice, call suggest_usage_metrics and add usage / cost-per-unit widgets.",
      "w": 12,
      "h": 2
    }
  ]
}
```

When `suggest_usage_metrics` returns units for a narrower scope (e.g. EKS-only dashboard), add widgets like:

```json
{
  "title": "CPU hours vs cost",
  "queries": [
    { "type": "cost", "name": "a", "chartType": "LINE", "alias": "Cost" },
    { "type": "usage", "name": "b", "metricId": "k8s_cpu_hours", "chartType": "LINE", "alias": "CPU hours" }
  ],
  "aggBy": "Week",
  "w": 6,
  "h": 3
}
```

```json
{
  "title": "Cost per CPU hour",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Cost" },
    { "type": "usage", "name": "b", "metricId": "k8s_cpu_hours", "alias": "CPU hours" },
    { "type": "formula", "name": "c", "formula": "a / b", "alias": "Cost per CPU hour", "chartType": "LINE" }
  ],
  "aggBy": "Week",
  "w": 6,
  "h": 3
}
```
