---
name: reports
description: "Use when creating, previewing, updating, scheduling, or exploring Costory reports. Route by intent: scheduled Slack/Teams/email sharing (Schedule — ask before build) vs explain last-month cost with preview-first DIGEST (Explain — chat preview required; not query+compare). Workflows are named, never lettered. DIGEST AI is opt-in via display: \"summary\" and/or enableAiInvestigation — both slower. Covers GRAPH_SNAPSHOT, TOP_FLOP, DASHBOARD_PDF, context-first inheritance, destinations after channel type is known, and delivery safety. Call get_skill with skillId \"reports\" before create_report or substantial report work."
---

# Reports

**Skill body version 0.5.0.** Workflows here are **named** — Schedule, Explain, Update, Run, Explore. Older bodies lettered them A–E, and other Costory surfaces used a different letter order. If you are holding a lettered routing table for reports, it is stale: route by the names in this body and ignore the letters.

A **report** has a shared **`context`** (global theme) and **widgets** that inherit it by default. It delivers those widgets (chart snapshot, PDF, top/flop, text, or **DIGEST** cost-change tree) to one or more destinations (Slack, Teams, email). Same mental model as dashboards: shared `context` + per-widget overrides. `create_report`, `update_report`, and `preview_report_widget` all take the same report-level `context`.

**Load this skill first** for any report creation, DIGEST preview, substantial update, or delivery conversation.

## Must-follow rules

Everything below this section is detail. These seven are the contract:

1. **Ask before build (Schedule / delivery).** Do not pick widgets, call `preview_report_widget`, or call `create_report` until the design is confirmed. Never silently default to a graph or an AI summary on Schedule.
2. **Explain exception — preview-first DIGEST.** For one-shot *"what changed / explain last month"* (Explain workflow or `explain-period-change` recipe), **defaulting to DIGEST chat preview is required.** Do not wait for a full design questionnaire; do not substitute `query` + `compare` for the first answer.
3. **Context-first.** Once answers are in, define the full report `context` before listing widgets. Widgets carry **only overrides** — never duplicate what already lives in `context`.
4. **Destinations last.** Do not call `list_available_destinations` until the user has named the channel **type** (Slack / Teams / email). Then list only that type and propose matches by name — never paste a whole channel list into the conversation.
5. **AI features are opt-in.** DIGEST-only. Set `display: "summary"` for the executive narrative and/or `enableAiInvestigation: true` for per-node deep analysis — both default off (tree-only) and are slower.
6. **Confirm delivery.** Both `NOW` and `SCHEDULED` need explicit confirmation before `create_report`. `UNSCHEDULED` does not.
7. **`datePreset`, never frozen dates,** on `SCHEDULED` reports.

## Routing

| User intent (examples) | Workflow |
|------------------------|----------|
| "Generate a scheduled report / message for Slack, Teams, or email" | **Schedule** |
| "I want to share FinOps information with my team" (recurring or channel delivery) | **Schedule** |
| "I need to explain last month's cost" / "what changed last month and why" | **Explain** (NOW + DIGEST) |
| Update an existing report | **Update** |
| Run / retry / transfer / archive | **Run** |
| Explore a delivered DIGEST execution | **Explore** |

If intent is ambiguous between Schedule and Explain, ask: **recurring delivery to a channel**, or **one-shot explanation of last month's costs?**

## Tool order

1. `get_skill` with `skillId: "reports"` — this guide
2. `get_context` — org context, currency, popular groupBys
3. `list_teams` — only when team scoping may apply; each scope's `id` is the `scopeId` value
4. `suggest_groupby` — when a DIGEST hierarchy is open-ended; needs the planned period + the report's scope filter
5. `list_available_destinations` — **only after the channel type is known**, and only for a delivered report
6. `preview_report_widget` → `create_report` / `update_report` — only after the design brief is confirmed

Destination discovery is a late step, not a warm-up. An org can have dozens of Slack channels; enumerating them before the user has said "Slack" (or "Teams", or "email") buries the design conversation in noise. Once the type is known, list only that type — and if the filtered list is still long, ask for a channel name or keyword and match against it rather than echoing every entry.

---

## Workflow: Schedule — create a scheduled report

**Triggers:** scheduled message for Slack / Teams / email; share FinOps information with a team on a cadence.

**Goal:** accompany the user to **design** a recurring report they will actually use, then create a `SCHEDULED` report. Do not assume DIGEST or jump to create.

**Tone:** collaborative design partner. Explain options in plain language, offer concrete starters, confirm a short design brief before any preview or create. The user should feel guided, not interrogated with a long form.

### 1 — Open the design conversation

Acknowledge the goal (recurring FinOps delivery), then ask a few open questions in one short turn:

1. **What should this deliver?** — what should recipients learn or do when it lands?
2. **Who / where?** — Slack, Teams, or email (the type only; concrete destinations come at build time)
3. **How often?** — weekly or monthly (rough is fine)

If they are unsure what to put in the report, do not wait passively — go straight to step 2 and offer starter ideas.

### 2 — Offer starter reports (proactively)

Present 3–5 options as a friendly menu (value prop first, then widgets). Invite pick / mix / customize:

| Starter | Plain-language pitch | Widgets | Typical schedule / presets |
|---------|----------------------|---------|----------------------------|
| Monthly cost digest | "What changed last month?" — ranked movers as a tree; optional AI narrative for execs | DIGEST (± optional AI summary) | MONTHLY · `LAST_MONTH` |
| Weekly team pulse | "What blew up or dropped for each team last week?" | TOP_FLOP ± GRAPH_SNAPSHOT | WEEKLY · `LAST_WEEK` / `TRAILING_14_WEEKS` |
| Migration / spend tracker | "Are we on track?" — trend over weeks + last-week movers | GRAPH_SNAPSHOT + TOP_FLOP | WEEKLY · GRAPH `TRAILING_14_WEEKS`, TOP_FLOP `LAST_WEEK` |
| Dashboard pack | "Send our existing FinOps dashboard on a cadence" | DASHBOARD_PDF | WEEKLY + weekday · period lives on the dashboard |
| Invoice / finance close | "Close the books on what moved last invoice month" | DIGEST (± DASHBOARD_PDF; optional AI summary) | MONTHLY · `LAST_INVOICE_MONTH` or `LAST_MONTH` |

Example framing: *"Here are a few patterns that work well for teams. Which fits, or should we combine two?"*

For a **concrete, business-specific design** (marketplace spend, namespace cost, provider credits, untagged coverage, env costs, …) load the **`recipes`** skill and Read the matching card — it comes back as a confirmed brief you build with this workflow.

### 3 — Deepen the design (after they pick a direction)

Fill only what is still missing — keep it conversational:

1. **Scope** — whole org, `scopeId` from `list_teams`, and/or `conditionsCel`
2. **Split / hierarchy** — if DIGEST is in the mix: confirm root + deeper levels, or run `suggest_groupby` and propose a tree in plain language
3. **Optional AI** (DIGEST only) — ask explicitly: executive narrative (`display: "summary"`) and/or per-node investigation (`enableAiInvestigation: true`); both slower; default tree-only (`display: "tree"`, investigation off)
4. **Cadence details** — WEEKLY needs `weekday` (0 = Sunday … 6 = Saturday); set `firstRunAt` (ISO-8601 UTC); use presets that roll forward (`LAST_WEEK`, `LAST_MONTH`, …). Refuse frozen `from`/`to` on SCHEDULED reports
5. **Currency / metric** — usually `cost` + org currency from `get_context`

The channel **type** is already known from step 1; the concrete destination is resolved at build time, not here.

Then restate a **one-paragraph design brief** (audience, cadence, widgets, scope, AI narrative / investigation yes/no) and get confirmation before building.

### 4 — Build

1. `get_context`, plus `list_teams` only if team scoping applies
2. Steps 1–3 above → design brief confirmed
3. If the DIGEST hierarchy was open-ended → `suggest_groupby` with the planned period + scope filter → propose root + `additionalGroupBy` → confirm
4. **Draft the report `context` first** — shared `datePreset` / metric / currency / `groupBy` / scope
5. If DIGEST is in the mix → `preview_report_widget` (defaults **100 / 5% / 20**; set `display: "summary"` / `enableAiInvestigation` only if opted in) → tune → re-preview
6. **Now** resolve delivery: `list_available_destinations` for the chosen channel type → propose matches by name → confirm the specific destination. Missing Slack/Teams integration → https://app.costory.io/integration
7. Confirm `schedule.mode: "SCHEDULED"` (period, weekday, `firstRunAt`) → `create_report` with the **same `context` + widgets** you previewed

---

## Workflow: Explain — last month (NOW + DIGEST)

**Triggers:** explain last month's cost; what changed last month and why; one-shot narrative for stakeholders. Prefer recipe `explain-period-change` when that card was loaded.

**Goal:** a **chat DIGEST preview** (then optional NOW) whose primary widget is **DIGEST**, with hierarchy from `suggest_groupby`. The core deliverable is the **change tree from `preview_report_widget`**; do not rebuild it with `query`.

### 1 — Defaults vs questions

**Chat-only (default when they did not name a channel):** assume whole org, `LAST_MONTH` (or `LAST_INVOICE_MONTH` if they said invoice), tree-only AI off. Ask only if scope/period is ambiguous.

**Before NOW delivery** (they asked to send): confirm scope, hierarchy, AI (`display: "summary"` / `enableAiInvestigation`), and channel type. GRAPH_SNAPSHOT only if they also want a trend image.

Do not skip to a graph-only report for this trigger — the core ask is explanation → DIGEST tree. Do not call `query` for the first answer.

### 2 — Discover the tree with `suggest_groupby` (same turn as preview)

1. `get_context` (+ `list_teams` only if team scoping is needed)
2. Resolve period: prefer `context.datePreset: "LAST_MONTH"` (or `LAST_INVOICE_MONTH`)
3. Call `suggest_groupby` with `from`/`to` matching that month and the same `filterCel` as the planned scope
4. Pick a tree: **root** = top hit → `context.groupBy`; **deeper levels** = next useful hits → `additionalGroupBy` (typically 1–2 levels). If suggestions are empty/weak → `popularGroupBys`, else `cos_provider` → `cos_service_name`
5. **Chat-only:** state the path in one sentence and go straight to preview (no confirmation wait). **NOW:** confirm the path before create

### 3 — Preview DIGEST (primary data tool)

1. Draft `context`: `datePreset: "LAST_MONTH"`, chosen `groupBy`, `metricId`, `currency`, optional scope
2. `preview_report_widget` with `{ context, widget }` (**singular** `widget`, never `widgets`) — minimal DIGEST (`additionalGroupBy`, thresholds **100 / 5% / 20**, `aggBy: "Month"`; `display: "summary"` / `enableAiInvestigation: true` only if opted in)
3. **Present only preview fields** — required shape:
   - Headline from `resolvedPeriod` + `totals`
   - `topIncreases` / `topDecreases` (path + Δ)
   - Tree outline from `rootNodes` (label, Δ, childCount)
   - Footer: thresholds + optional `explorerUrl` / `comparisonPeriodSummary`
   - `summaryMarkdown` only when `display: "summary"`
4. Stop. Offer drill-down / AI re-preview / NOW. Tune from `recommendations` → re-preview — still without `query` unless the user names a node to dig into

### 4 — Deliver

1. Stop here if they only wanted an explanation in chat — no `create_report` needed
2. To send: channel type known → `list_available_destinations` for that type → confirm the channel → confirm `schedule.mode: "NOW"`
3. `create_report` with the **same** `context` + that DIGEST object inside **`widgets: […]`** (plural array; GRAPH_SNAPSHOT only if requested)

---

## Workflow: Update

`get` (report id) → preview if DIGEST content changes → `update_report`.

- Patch `context` alone to change the report-wide period / groupBy / filter / currency without rewriting widgets.
- Pass `widgets` only when replacing the widget list (wholesale replace). The field is always the `widgets` **array** — a singular `widget` key is silently ignored.
- New widgets follow the same inheritance rules: omit fields that match `context`.

## Workflow: Run — run, retry, transfer, archive

Retry **failed executions only** — never `run_report_now` to fix one destination. Do not poll executions in an unbounded loop.

## Workflow: Explore — delivered DIGEST

`get_report_execution` → `get_report_execution_widget` with `view: "tree"` for the full formatted tree.

---

# Payload cookbook

Read this half when drafting JSON — the conversation rules above still apply.

## Report `context` fields

| Field | Role | Widget inheritance |
|-------|------|-------------------|
| `metricId` | Default cost column (e.g. `"cost"`) | Cost widgets omit `metricId` to inherit |
| `currency` | USD, EUR, GBP, CNY | Cost widgets omit `currency` to inherit |
| `groupBy` | Default split — for DIGEST this is the **root** hierarchy axis | Cost widgets omit `groupBy` to inherit; DIGEST deeper levels go in `additionalGroupBy` |
| `datePreset` **or** `startDate`/`endDate` | Report period — prefer `datePreset` for scheduled reports | Widgets omit `from`/`to`/`datePreset` to inherit |
| `conditionsCel` | Report-wide CEL filter (scope) | Merged into every cost widget at runtime |
| `scopeId` | Optional saved team scope | Report-wide unless a widget overrides |

Each query still needs `"type": "cost"` — the query kind is not inherited.

Report **ownership** (`teamId`, `visibility` on `create_report`) is separate from `scopeId` / `conditionsCel` (query filter only). Ownership controls who sees and edits the report; scope controls which costs the widgets query.

## Inheritance resolution

At preview, create, and each scheduled run:

1. Start from report `context` (period, `metricId`, `currency`, `groupBy`, `conditionsCel`, `scopeId`).
2. Apply **widget-level** overrides (`datePreset` / `from`/`to`, `scopeId`, DIGEST thresholds, `aggBy`, title, …).
3. Apply **query-level** overrides (`groupBy`, `metricId`, `currency`, `filterCel`, `chartType`, `alias`).
4. Effective cost filter = report `conditionsCel` **AND** widget/query `filterCel` (an empty side is a no-op).

Consequences worth knowing:

- **Similarity rule:** when widgets share a period, split, metric, currency, or scope, put it in `context` first. Widget fields are exceptions only.
- **Per-widget periods are normal** — e.g. GRAPH = `TRAILING_14_WEEKS`, TOP_FLOP = `LAST_WEEK`. Keep shared fields on `context`; override only `datePreset` (and presentation) on the widget.
- **Never combine** `datePreset` with explicit dates on the same layer.
- **`conditionsCel` is report-wide** — do not repeat it inside every widget `filterCel`.
- **TEXT and DASHBOARD_PDF do not inherit query fields.** PDF period lives on the referenced dashboard; TEXT is static markdown in `contentMarkdown` (not dashboard `textContent`).
- Use `startDate`/`endDate` (or widget `from`/`to`) only for truly custom one-off ranges — never on `SCHEDULED` reports.

## Widget types

| Type | What it delivers | Best for |
|------|------------------|----------|
| `DIGEST` | Cost-change **tree** vs previous period; optional `display: "summary"` narrative and/or `enableAiInvestigation` | Explain what moved (monthly / weekly) |
| `GRAPH_SNAPSHOT` | Chart image from a cost/usage/metric query | Trends, migration tracking, composition over time |
| `TOP_FLOP` | Ranked increases / decreases for a period | "What blew up / what dropped last week?" |
| `DASHBOARD_PDF` | PDF of an existing dashboard (`dashboardId`) | Weekly/monthly pack of a known dashboard |
| `TEXT` | Static markdown via `contentMarkdown` | Context, how-to-read notes, links (**not** live AI) |

A report can combine several widgets. Prefer variety of **questions**, not duplicate charts.

## DIGEST AI options (`display` + `enableAiInvestigation`)

Two independent DIGEST fields control AI (both opt-in; both slower):

| Field | Values / default | What it does |
|-------|------------------|--------------|
| `display` | `"tree"` (default) \| `"table"` \| `"summary"` | Presentation. `"summary"` = LLM executive narrative (`summaryMarkdown`). `"tree"` / `"table"` = structured view only (faster). |
| `enableAiInvestigation` | `false` (default) \| `true` | Per-node deep analysis on deepest-leaf movers (async; delivery waits). **Independent** of `display`. |

Analysis presets (map intent → fields; do not invent other knobs):

| Intent | Fields |
|--------|--------|
| Direct tree (default) | `display: "tree"`, `enableAiInvestigation: false` (or omit both) |
| Executive summary | `display: "summary"`, `enableAiInvestigation: false` |
| Deep investigation | `display: "summary"`, `enableAiInvestigation: true`, `topLargestAbsoluteChange: 20` |

- **Offer narrative** when they want a written overview of what changed and why (exec / stakeholder read). Set `display: "summary"`.
- **Offer investigation** only when they want per-node AI explanations under the tree — warn it is noticeably slower. Set `enableAiInvestigation: true`. Prefer `display: "summary"` with it so the narrative can ground on those findings; do not ask a second tree-vs-summary question once deep investigation is chosen.
- **Say the trade-off out loud** before enabling either. Ask plainly: *"Tree-only (faster), an AI executive summary, or deep per-node investigation (slowest)?"*
- **Default to tree-only** unless they opt in. Never invent a separate AI widget or fake a summary via TEXT.
- Only DIGEST produces `summaryMarkdown` — never claim one from GRAPH_SNAPSHOT, TOP_FLOP, TEXT, or DASHBOARD_PDF.

Pairings: tree is enough → DIGEST alone (defaults). Narrative + tree → DIGEST with `display: "summary"`. Deep dig → DIGEST with `display: "summary"` + `enableAiInvestigation: true`. Trend or movers only → GRAPH_SNAPSHOT / TOP_FLOP. Static intro → TEXT.

## TEXT widgets

Free-form markdown block. Shape is **`{ type: "TEXT", contentMarkdown }`** — optional `title` / `description`.

- Field name is **`contentMarkdown`** (1–10,000 characters after trim). Do **not** use `textContent` — that is the **dashboards** text-widget field (`type: "text"`).
- No queries, period, or inheritance from report `context`.
- Not a substitute for the DIGEST AI summary (`summaryMarkdown`).

```json
{
  "type": "TEXT",
  "title": "How to read this report",
  "contentMarkdown": "## Weekly pulse\n\nFocus on the TOP_FLOP movers below, then open the graph for trend context."
}
```

## DIGEST hierarchy

DIGEST builds a tree. Levels map to fields:

| Level | Field | Notes |
|-------|-------|-------|
| Root (top of tree) | `context.groupBy` | Preferred; use `queries[0].groupBy` only when one DIGEST must differ |
| Deeper levels (in order) | `additionalGroupBy` | Ordered children only — **never** put the root here |

| User says | `context.groupBy` (root) | `additionalGroupBy` |
|-----------|---------------------------|---------------------|
| provider → service | `cos_provider` | `["cos_service_name"]` |
| env → project → service | `environment` | `["cos_sub_account_id", "cos_service_name"]` |
| team → service | `team` | `["cos_service_name"]` |

- Confirm the resolved path with the user when they describe a hierarchy in plain language (e.g. "production → Costory → AmazonEC2").
- Discover CEL field names via `search` with `type: ["dimensions"]`, or from `get_context` popularGroupBys.
- For an open-ended hierarchy ("good split for last month"), call `suggest_groupby` with the planned period + the same `filterCel` / scope as the report — top hit as root, next hits as `additionalGroupBy` candidates. Prefer dimensions that also appear in popularGroupBys when ties exist.
- Never invent a deep tree without confirming it.

## Query config (DIGEST, GRAPH_SNAPSHOT, TOP_FLOP)

Same query shape as `query`, minus anything that matches `context`:

- `queries` — for DIGEST: exactly one `{ type: "cost", name: "a", filterCel? }` (omit `groupBy` / `metricId` / `currency` when they match context)
- **Period** — omit on widgets when `context.datePreset` (or start/end) is set; widget-level `datePreset` / `from`/`to` are overrides only. Never hand-compute dates for recurring schedules. For DIGEST the server also resolves the comparison period and echoes `resolvedPeriod`.
- `aggBy`
- Widget `scopeId?` only when it must differ from `context.scopeId`

## Preview defaults (DIGEST)

| Field | Default |
|-------|---------|
| `minAbsoluteDiff` | **100** |
| `minRelativeDiff` | **5** (percent) |
| `topLargestAbsoluteChange` | **20** (allowed: 5, 10, 15, or 20) |
| `display` | **`"tree"`** (set `"summary"` only when opted in) |
| `enableAiInvestigation` | **`false`** (set `true` only when opted in) |

Always show `resolvedPeriod` (when present), `comparisonPeriodSummary`, `totals`, `counts`, `topIncreases` / `topDecreases`, and `rootNodes`. Show `summaryMarkdown` only when `display: "summary"` (and warn that that preview may take longer).

Tune from `recommendations`: thresholds, `topLargestAbsoluteChange` (only 5, 10, 15, or 20), grouping. Repeat `preview_report_widget` until satisfied.

## Examples

**`preview_report_widget` — monthly DIGEST (tree-only defaults):**

```json
{
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "environment",
    "metricId": "cost",
    "currency": "USD"
  },
  "widget": {
    "type": "DIGEST",
    "queries": [{ "type": "cost", "name": "a", "alias": "Cost by environment" }],
    "aggBy": "Month",
    "additionalGroupBy": ["cos_sub_account_id", "cos_service_name"],
    "minAbsoluteDiff": 100,
    "minRelativeDiff": 5,
    "topLargestAbsoluteChange": 20
  }
}
```

**Same DIGEST widget with executive AI summary** (add on the widget when opted in):

```json
{
  "type": "DIGEST",
  "queries": [{ "type": "cost", "name": "a", "alias": "Cost by environment" }],
  "aggBy": "Month",
  "additionalGroupBy": ["cos_sub_account_id", "cos_service_name"],
  "minAbsoluteDiff": 100,
  "minRelativeDiff": 5,
  "topLargestAbsoluteChange": 20,
  "display": "summary",
  "enableAiInvestigation": false
}
```

**`create_report` / `update_report` — same DIGEST goes in `widgets` (plural array).**

**Migration tracker (graph + top/flop — not DIGEST):**

```json
{
  "context": {
    "metricId": "cost",
    "currency": "USD",
    "groupBy": "team",
    "datePreset": "LAST_WEEK"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Cost by team — trailing weeks",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost by team",
        "chartType": "LINE"
      }],
      "datePreset": "TRAILING_14_WEEKS",
      "aggBy": "Week"
    },
    {
      "type": "TOP_FLOP",
      "title": "Last week movers by team",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost by team"
      }],
      "aggBy": "Period",
      "topN": 5,
      "flopN": 5
    }
  ]
}
```

**Weekly dashboard PDF widget:**

```json
{
  "type": "DASHBOARD_PDF",
  "dashboardId": "<id from search>"
}
```

**TEXT intro (report field is `contentMarkdown`, not dashboard `textContent`):**

```json
{
  "type": "TEXT",
  "title": "How to read this report",
  "contentMarkdown": "## Weekly pulse\n\nFocus on the TOP_FLOP movers below."
}
```

**Full `create_report` (scheduled):**

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "MONTHLY",
    "firstRunAt": "2026-08-01T10:00:00.000Z"
  },
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "environment",
    "metricId": "cost",
    "currency": "USD"
  },
  "widgets": [{ "type": "DIGEST", "...": "minimal widget overrides only" }],
  "destinations": [{ "destinationType": "SLACK", "channelId": "C…" }]
}
```

## Payload anti-patterns

The conversation rules live in **Must-follow rules** above. These are the shape mistakes:

- Do not put the root hierarchy axis in `additionalGroupBy` instead of `context.groupBy`
- Do not repeat `metricId`, `currency`, `groupBy`, `from`/`to`, or `datePreset` on a widget when it already matches `context`
- Do not put the global scope in each widget's `filterCel` instead of `context.conditionsCel`
- Do not confuse ownership (`teamId` / `visibility`) with query scope (`scopeId` / `conditionsCel` / `filterCel`)
- Do not use a different widget *content* for preview vs create — same `context` + DIGEST fields — but the key differs: preview = `widget`, create/update = `widgets[]`
- Do not pass `widgets` to `preview_report_widget` or a singular `widget` to `create_report` (Zod strict rejects both)
- Do not send dashboard `textContent` on a report TEXT widget — the field is `contentMarkdown` with `type: "TEXT"`
- Do not skip preview before create when DIGEST is in the mix
- Do not skip confirming the DIGEST tree path after `suggest_groupby` when delivering NOW/SCHEDULED (chat-only Explain may preview in the same turn)
- Do not substitute destinations silently
- Do not use `query` + `compare` as a substitute for Explain’s DIGEST preview on the first answer
- Do not rebuild DIGEST movers as explorer tables or a canvas before presenting preview fields

## Related skills

- `dashboards` — when the user wants an interactive dashboard instead of a delivered report (same context-first inheritance model; also uses `suggest_groupby`)
- `query` — **after** DIGEST preview, drill a user-named node (`filterCel`); not the primary tool for “what changed last month”
- `recipes` → `explain-period-change` — one-shot explain outcome card (preview-first DIGEST)
- `virtual-dimensions` — when a DIGEST hierarchy needs a custom axis that does not exist yet
