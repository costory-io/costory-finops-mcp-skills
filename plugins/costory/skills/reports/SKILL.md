---
name: reports
description: "Use when creating, previewing, updating, scheduling, or exploring Costory reports. Route by intent: scheduled Slack/Teams/email sharing (Schedule workflow — accompany the user, offer starter report designs, ask before build) vs explain last-month cost with DIGEST + suggest_groupby (Explain workflow — NOW). Workflows are named, never lettered. DIGEST AI summary is optional and slower. Covers GRAPH_SNAPSHOT, TOP_FLOP, DASHBOARD_PDF, context-first inheritance, destinations discovered only after the channel type is known, and delivery safety. Call get_skill with skillId \"reports\" before create_report or substantial report work."
---

# Reports

**Skill body version 0.4.1.** Workflows here are **named** — Schedule, Explain, Update, Run, Explore. Older bodies lettered them A–E, and other Costory surfaces used a different letter order. If you are holding a lettered routing table for reports, it is stale: route by the names in this body and ignore the letters.

A **report** has a shared **`context`** (global theme) and **widgets** that inherit it by default. It delivers those widgets (chart snapshot, PDF, top/flop, text, or **DIGEST** cost-change tree) to one or more destinations (Slack, Teams, email). Same mental model as dashboards: shared `context` + per-widget overrides. `create_report`, `update_report`, and `preview_report_widget` all take the same report-level `context`.

**Load this skill first** for any report creation, DIGEST preview, substantial update, or delivery conversation.

## Must-follow rules

Everything below this section is detail. These six are the contract:

1. **Ask before build.** Do not pick widgets, call `preview_report_widget`, or call `create_report` until the design is confirmed. Never silently default to DIGEST, a graph, or an AI summary.
2. **Context-first.** Once answers are in, define the full report `context` before listing widgets. Widgets carry **only overrides** — never duplicate what already lives in `context`.
3. **Destinations last.** Do not call `list_available_destinations` until the user has named the channel **type** (Slack / Teams / email). Then list only that type and propose matches by name — never paste a whole channel list into the conversation.
4. **AI summary is opt-in.** DIGEST-only, noticeably slower, off unless the user asks for it.
5. **Confirm delivery.** Both `NOW` and `SCHEDULED` need explicit confirmation before `create_report`. `UNSCHEDULED` does not.
6. **`datePreset`, never frozen dates,** on `SCHEDULED` reports.

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
3. **Optional AI summary** (DIGEST only) — ask explicitly, mention it takes longer, default off unless they want the narrative
4. **Cadence details** — WEEKLY needs `weekday` (0 = Sunday … 6 = Saturday); set `firstRunAt` (ISO-8601 UTC); use presets that roll forward (`LAST_WEEK`, `LAST_MONTH`, …). Refuse frozen `from`/`to` on SCHEDULED reports
5. **Currency / metric** — usually `cost` + org currency from `get_context`

The channel **type** is already known from step 1; the concrete destination is resolved at build time, not here.

Then restate a **one-paragraph design brief** (audience, cadence, widgets, scope, AI summary yes/no) and get confirmation before building.

### 4 — Build

1. `get_context`, plus `list_teams` only if team scoping applies
2. Steps 1–3 above → design brief confirmed
3. If the DIGEST hierarchy was open-ended → `suggest_groupby` with the planned period + scope filter → propose root + `additionalGroupBy` → confirm
4. **Draft the report `context` first** — shared `datePreset` / metric / currency / `groupBy` / scope
5. If DIGEST is in the mix → `preview_report_widget` (defaults **100 / 5% / 20**; AI summary only if opted in) → tune → re-preview
6. **Now** resolve delivery: `list_available_destinations` for the chosen channel type → propose matches by name → confirm the specific destination. Missing Slack/Teams integration → https://app.costory.io/integration
7. Confirm `schedule.mode: "SCHEDULED"` (period, weekday, `firstRunAt`) → `create_report` with the **same `context` + widgets** you previewed

---

## Workflow: Explain — last month (NOW + DIGEST)

**Triggers:** explain last month's cost; what changed last month and why; one-shot narrative for stakeholders.

**Goal:** a `NOW` (or preview-first, then NOW) report whose primary widget is **DIGEST**, with hierarchy from `suggest_groupby`. The core deliverable is the **change tree**; the AI summary is optional and slower.

### 1 — Ask first (required)

1. **Scope** — whole org, team (`scopeId`), and/or CEL filter?
2. **Audience** — preview in chat only, or send NOW to Slack / Teams / email?
3. **Hierarchy preference** — if they already know the tree (e.g. team → service), confirm it; otherwise you will propose from `suggest_groupby`
4. **Optional AI summary** — ask if they want the narrative add-on; warn it takes longer; default off unless they need an exec-style write-up
5. **Optional extras** — add a GRAPH_SNAPSHOT (e.g. `TRAILING_14_WEEKS`) only if they also want a trend image

Do not skip to a graph-only report for this trigger — the core ask is explanation → DIGEST tree.

### 2 — Discover the tree with `suggest_groupby`

1. `get_context` (+ `list_teams` if needed)
2. Resolve period: prefer `context.datePreset: "LAST_MONTH"` (or `LAST_INVOICE_MONTH` if they mean invoice close)
3. Call `suggest_groupby` with `from`/`to` matching that month and the same `filterCel` as the planned scope
4. Propose a tree: **root** = top hit → `context.groupBy`; **deeper levels** = next useful hits → `additionalGroupBy` (typically 1–2 levels)
5. Confirm the path in plain language before preview

### 3 — Preview DIGEST

1. Draft `context`: `datePreset: "LAST_MONTH"`, confirmed `groupBy`, `metricId`, `currency`, optional scope
2. `preview_report_widget` with a minimal DIGEST widget (`additionalGroupBy`, thresholds **100 / 5% / 20**, `aggBy: "Month"`; AI summary only if opted in at step 1)
3. Present totals, top increases/decreases, and the tree outline (`rootNodes`); include `summaryMarkdown` only when the AI summary was enabled
4. Tune from `recommendations` / user feedback → re-preview

### 4 — Deliver

1. Stop here if they only wanted an explanation in chat — no `create_report` needed
2. To send: the channel type came from step 1 → `list_available_destinations` for that type → confirm the channel → confirm `schedule.mode: "NOW"`
3. `create_report` with the **same** `context` + DIGEST widget (GRAPH_SNAPSHOT only if requested at step 1)

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
- **TEXT and DASHBOARD_PDF do not inherit query fields.** PDF period lives on the referenced dashboard; TEXT is static markdown.
- Use `startDate`/`endDate` (or widget `from`/`to`) only for truly custom one-off ranges — never on `SCHEDULED` reports.

## Widget types

| Type | What it delivers | Best for |
|------|------------------|----------|
| `DIGEST` | Cost-change **tree** vs previous period; optional AI executive summary | Explain what moved (monthly / weekly) |
| `GRAPH_SNAPSHOT` | Chart image from a cost/usage/metric query | Trends, migration tracking, composition over time |
| `TOP_FLOP` | Ranked increases / decreases for a period | "What blew up / what dropped last week?" |
| `DASHBOARD_PDF` | PDF of an existing dashboard (`dashboardId`) | Weekly/monthly pack of a known dashboard |
| `TEXT` | Static markdown | Context, how-to-read notes, links (**not** live AI) |

A report can combine several widgets. Prefer variety of **questions**, not duplicate charts.

## Optional AI summary (DIGEST only)

DIGEST always delivers the change tree. The AI executive summary (`summaryMarkdown`) is an add-on the user must opt into.

- **Offer it when** they want a written narrative of what changed and why (exec / stakeholder read).
- **Say the trade-off out loud:** richer narrative, but noticeably slower to preview and to deliver. Ask plainly: *"Do you want the optional AI summary? Useful for exec readouts, but slower to generate."*
- **Default to tree-only** (faster) unless they opt in — omit `display` or set `display: "tree"`.
- **Opt in via MCP:** set `display: "summary"` on the DIGEST widget (preview + create). That is the executive narrative flag. Separately, `enableAiInvestigation: true` runs deep per-node analysis before delivery (even slower) — only when the user asks for that, not for a normal AI summary.
- Never invent a field, a separate AI widget, or a fake summary via TEXT.
- Only DIGEST produces `summaryMarkdown` — never claim one from GRAPH_SNAPSHOT, TOP_FLOP, TEXT, or DASHBOARD_PDF.

Pairings: tree is enough → DIGEST alone. Narrative + tree → DIGEST with `display: "summary"`. Trend or movers only → GRAPH_SNAPSHOT / TOP_FLOP. Static intro → TEXT. Narrative + trend → DIGEST (`display: "summary"`) plus GRAPH_SNAPSHOT / TOP_FLOP.

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

Always show `resolvedPeriod` (when present), `comparisonPeriodSummary`, `totals`, `counts`, `topIncreases` / `topDecreases`, and `rootNodes`. Show `summaryMarkdown` only when the user opted into the AI summary (and warn that that preview may take longer).

Tune from `recommendations`: thresholds, `topLargestAbsoluteChange` (only 5, 10, 15, or 20), grouping. Repeat `preview_report_widget` until satisfied.

## Examples

**Monthly DIGEST (after the user confirmed the recipe):**

```json
{
  "context": {
    "datePreset": "LAST_MONTH",
    "groupBy": "environment",
    "metricId": "cost",
    "currency": "USD"
  },
  "widgets": [{
    "type": "DIGEST",
    "queries": [{ "type": "cost", "name": "a", "alias": "Cost by environment" }],
    "aggBy": "Month",
    "additionalGroupBy": ["cos_sub_account_id", "cos_service_name"],
    "minAbsoluteDiff": 100,
    "minRelativeDiff": 5,
    "topLargestAbsoluteChange": 20
  }]
}
```

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
- Do not use a different shape for preview vs create — same `context` + widget object
- Do not skip preview before create when DIGEST is in the mix
- Do not skip confirming the DIGEST tree path after `suggest_groupby`
- Do not substitute destinations silently

## Related skills

- `dashboards` — when the user wants an interactive dashboard instead of a delivered report (same context-first inheritance model; also uses `suggest_groupby`)
- `query` — validate scope or inspect drivers before locking DIGEST thresholds; `suggest_groupby` shape matches the query skill
- `virtual-dimensions` — when a DIGEST hierarchy needs a custom axis that does not exist yet
