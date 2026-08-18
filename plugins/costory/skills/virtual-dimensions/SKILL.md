---
name: virtual-dimensions
description: "Use when creating, editing, previewing, or publishing Costory virtual dimensions (custom cost axes) with ordered CEL rules, leftover buckets, overlap/shadowing checks, or telemetry split-by-usage allocations. Call get_skill with skillId \"virtual-dimensions\" before create_virtual_dimension_draft or update_virtual_dimension_draft."
---

# Virtual Dimensions

A **virtual dimension** is a custom cost axis (e.g. "Environment", "Team", "Product line") built from **ordered rules**. Each rule has:

- **`conditionCel`** — CEL filter selecting rows (e.g. `cos_environment in ["prod"]`)
- **`allocation`** — where matched spend goes. Supports all types: `dimensionValue`, `existingColumn`, `splitCost`, `telemetry` (same shapes as `get` returns for a virtual dimension)

**Rules are evaluated top-to-bottom; first match wins.** A **leftover rule** catches everything else.

**Edits happen on a draft.** Publishing is explicit and triggers BigQuery refresh. Production is unchanged until `publish_virtual_dimension`.

## When to Trigger

- Building a new custom cost axis (Environment, Team, Product line, …)
- Editing rules on an existing virtual dimension
- Previewing leftover %, breakdown, or rule overlap/shadowing
- Splitting shared cost by a usage metric (`telemetry` allocation)
- Publishing a draft after user confirmation

## Auth

Write tools (`create` / `update` / `publish`) are Clerk-only — not available on the service route. Read tools work on both routes.

## Concepts

| Concept | Meaning |
|---------|---------|
| Draft vs published | Mutations touch the latest pending draft; published state stays live until publish |
| Draft persistence | `preview_virtual_dimension_draft` and `virtual_dimension_overlap_matrix` (when `includeDraft: true`) return `draftPersisted: true` when a pending draft exists, `draftPersisted: false` when analyzing published-only state in memory (not publishable). `create_virtual_dimension_draft` and `update_virtual_dimension_draft` persist only when validation passes — on success `draftPersisted: true` and `draftValidation: { ok: true }`; on failure nothing is saved (`draftPersisted: false`). `publish_virtual_dimension` requires a **persisted** pending draft — call `update_virtual_dimension_draft` for an existing virtualDimensionId (or `create_virtual_dimension_draft` for a brand-new VDIM) |
| Rule order | Earlier rules claim spend; later overlapping rules are **shadowed** |
| Leftovers | Spend not matched by any named rule — high leftover means add more rules |
| Leftover (MCP) | Auto catch-all returned as `leftoverRule` (not in `rules`). **Do not include a catch-all in `rules` on create or update** — duplicate leftovers cause ordering confusion. There is no leftover-label input on create/update — rename the leftover bucket in the web app |
| Rule ids | **Create:** saveVirtualDimensionDraft re-stamps all rule ids — response ids are authoritative. **Update:** preserves ids from input (including when seeding a draft from published state). **Both:** ids are sticky to logical rules — reorder by moving `id` + `name` + `conditionCel` + `allocation` together; attaching an id to a different condition silently mis-assigns spend |
| Tags | `get` returns `tags` (names) for both published and draft. Pass `tagNames` (strings) on create/update — not tag UUIDs |
| `values` | Output-only — derived from rule allocations; do not send on create/update |
| Allocation shapes | `dimensionValue`: `{ allocationType: "dimensionValue", dimensionValue: "prod" }`; `existingColumn`: `{ allocationType: "existingColumn", existingColumn: "cos_team" }`; `splitCost`: `{ allocationType: "splitCost", reAllocationParams: { type: "custom", partitions: [{ label: "A", weight: 60 }, { label: "B", weight: 40 }] } }` (weights sum to 100); `telemetry`: `{ allocationType: "telemetry", externalMetric: { provider, integrationId, metricName, aggregator, groupByFields: ["<split-by>"] }, mappingType: "mapping" \| "regexMapping" \| "identity", mappingParams: { mapping: { "key": "value" } }, regexTransformation: "..." }` — discover `externalMetric` via `list_metrics` `{ includeExternal: true, search }`. Do **not** set `datasource` on new reallocations (leftover stored rows may still have it — echo unchanged). mapping keys are metric **value** names (not `cos_*` dimensions); unmapped values → leftover. Echo non-`dimensionValue` allocations from get unchanged unless the user explicitly asks to edit advanced allocation behavior |
| CEL only | Always use `conditionCel` strings; never raw query-builder JSON |
| Unlabelled values | Use `cos_environment == null` in `conditionCel` (not `is_null` or string `"null"`) |
| Declarative updates | `update_virtual_dimension_draft` takes the **full desired rules array**, not diffs |
| `name` vs `bqName` | `name` is the display label (can change in drafts). `bqName` is the **immutable** BigQuery / CEL field name (e.g. `virtual_environment`) — set once at `create_virtual_dimension_draft` from the initial name and **never updated**, even after rename or publish. Read `bqName` from create/update/publish/get responses (or `list_virtual_dimensions` when resolving by name only) before `groupBy` / `filterCel`; **never derive it from `name`**. |
| Post-publish query | After publish, use returned `bqName` (not `name`) in `query`. `computeStatus`: `REFRESHING` = refresh job queued — poll `get` or `list_virtual_dimensions` until `COMPLETED` before querying; `TO_REFRESH` = draft promoted but refresh job queuing failed — retry `publish_virtual_dimension` or poll `computeStatus` until `COMPLETED` |

## Strategy approval ≠ rule approval

When several mapping approaches exist (e.g. reuse `env` vs derive from `cos_sub_account_id` patterns vs split by `cos_team`), **present the options and let the user pick a strategy first**.

**Picking a strategy is not approval to create the draft.** After the user chooses an approach:

1. **Draft the concrete rules** for that strategy — each rule's `name`, `conditionCel`, and `allocation` (leftover catch-all is added automatically; do not put it in `rules`).
2. **Present the full proposed rule set** — or, when there are many rules, a representative summary (rule order, main buckets, example CEL per bucket) **plus called-out edge cases** (null/unlabelled values, ambiguous values, known shadowing risks, spend you could not classify).
3. **Wait for explicit rule approval** — the user must confirm the rules (or request edits) before you call `create_virtual_dimension_draft`.

Do not call `create_virtual_dimension_draft` immediately after the user says "use approach B" or "map by sub-account" — that only approves the **mapping strategy**, not the **rule set**.

## When target values are unclear

If you do not know what buckets the finished dimension should have — vague ask ("organize our cloud spend"), no obvious `cos_*` field, or search results do not suggest a sensible split — **ask the user for the values they expect** before proposing mapping strategies or rules.

Example prompt: *"What values should this dimension use? For example: Data, ML, Infra — or prod, staging, dev?"*

Use their answer as:

- Target `allocation.dimensionValue` labels (one rule per bucket, unless they specify otherwise)
- Keywords for `search` and `query` when drafting `conditionCel`
- A checklist when presenting the proposed rule set (did every expected value get a rule? what lands in leftover?)

Do not invent bucket names from spend patterns alone without confirming — inferred labels often miss how the org actually thinks about costs. Batch value questions with other clarifying questions in a single message when possible.

## Supporting tools (before VDIM tools)

| When | Tool |
|------|------|
| Discover valid `cos_*` dimension names | `search` with `type: ["dimensions"]` (empty query lists all) |
| Find dimension values / existing VDIMs for a topic | `search` with `type: ["dimensions", "virtual_dimensions"]` |
| Explore spend to decide what rules to write | `query` |
| Pick a natural split dimension for unclaimed spend | `suggest_groupby` |
| Discover a live integration metric for a new `telemetry` allocation | `list_metrics` `{ includeExternal: true, search }` — persist inline `externalMetric` (do not set `datasource`) |
| Inspect leftover saved-metric telemetry (already-stored `datasource` only) | `list_metrics` `{ datasourceId }` — not for new reallocations |
| See series keys of a chosen external metric before mapping | `query` (`type: "externalMetric"`, `groupByFields`) |

## Workflow A — Create a new virtual dimension on a topic

Use when the user wants a **new** axis (e.g. "build an Environment dimension", "allocate costs by team").

1. `search` with `type: ["dimensions"]` — discover fields + values: `{ query: "", type: ["dimensions"] }` lists all `cos_*` field names and top values; repeat with keywords (e.g. `"prod"`, `"staging"`) to find values for `conditionCel`. Optional: `query` to understand spend shape.

> **Tip:** Don't hesitate to search extensively at this stage. The goal is to gather all available context for your virtual dimension.
> - If you do not know what buckets the dimension should have, **ask for expected values first** — e.g. Data, ML, Infra.
> - `search` with `type: ["dimensions"]` combines keyword + semantic value discovery. `virtual_dimensions` hits are name/description keyword search only.
> - If building a "Team" dimension, ask the user for specific team names to scan for using search.
> - Ask clarifying questions if needed. Batch questions into a single message.

2. **If multiple mapping approaches exist**, present options → user picks a **strategy**.
3. **Present proposed rules → explicit rule approval** — show the full rule set (or representative summary + edge cases). **Do not** call `create_virtual_dimension_draft` until the user confirms the rules.
4. `create_virtual_dimension_draft` — `name`, optional `description`, `tagNames`, `rules` only (no catch-all — a default leftover rule is added automatically as `leftoverRule`). **Pick the final name now** — `bqName` is derived from this name and is immutable. Rejects invalid payloads — fix errors and retry. On success, `draftPersisted: true`.
5. `preview_virtual_dimension_draft` `mode: "costs"` — check leftover %
6. **[if shadowing concerns]** `virtual_dimension_overlap_matrix` — shadowing between named rules only
7. `update_virtual_dimension_draft` — iterate with full `rules` array (must pass validation to persist)
8. `preview_virtual_dimension_draft` `mode: "costs"` — repeat until satisfied
9. `publish_virtual_dimension` — only when the user explicitly confirms

**First draft tips:** start with 1–5 high-confidence rules; do not put a catch-all in `rules`; use `tagNames` (strings), not tag UUIDs.

## Workflow B — Iterate on an existing virtual dimension

1. `list_virtual_dimensions` with `query` — find by name and resolve `virtualDimensionId` (prefer over `search` for id resolution; use `search` only for discovery, then `list` for id)
2. `get` — published + draft rules (CEL), dependencies (`type: "virtualDimension"`)
3. Optional diagnostics:
   - `preview_virtual_dimension_draft` `mode: "costs"` — per-rule costs + leftover %
   - `preview_virtual_dimension_draft` `mode: "breakdown"` — drill into a rule with `ruleId` + `groupBy`
   - `virtual_dimension_overlap_matrix` — shadowing / overlap between rules
4. `update_virtual_dimension_draft` — full `rules` array (declarative replace). Copy every existing rule's `id` from `get` (draft rules if pending, else published). For `dimensionValue` rules you may edit `name`, `conditionCel`, and `allocation.dimensionValue`; for other allocation types echo `allocation` unchanged and edit only `name` / `conditionCel`. When reordering, move each `id` with its rule — ids are sticky. Rejects invalid payloads — nothing is saved until validation passes.
5. `preview_virtual_dimension_draft` `mode: "costs"`
6. `publish_virtual_dimension` — only when the user explicitly confirms

If no pending draft exists, `update_virtual_dimension_draft` seeds one from published state (response may include `warning: "Initialized draft from published state"`).

## Workflow C — Reallocate shared cost by a usage metric

Use when the user wants to split shared spend proportionally to a live integration metric (e.g. CPU-hours, requests) — the `telemetry` allocation type. New reallocations persist inline `externalMetric` only.

1. `list_metrics` `{ includeExternal: true, search }` → pick `provider`, `integrationId`, `metricName`, and a split-by attribute from `attributes`.
2. `query` `{ type: "externalMetric", provider, integrationId, metricName, aggregator, groupByFields: ["<attribute>"] }` → see top series keys. These become `mappingParams.mapping` keys.
3. Present the proposed mapping for **explicit rule approval** (do not guess bucket labels; map significant values and let the long tail go to leftover).
4. `create_virtual_dimension_draft` / `update_virtual_dimension_draft` with a rule whose `allocation` is `{ allocationType: "telemetry", externalMetric: { provider, integrationId, metricName, aggregator, groupByFields }, mappingType: "mapping", mappingParams: { mapping } }` (or `regexMapping` + `regexTransformation` with named capture groups). Do not set `datasource`.
5. `preview_virtual_dimension_draft` `mode: "costs"` → confirm the split and leftover share.
6. `publish_virtual_dimension` on explicit confirmation.

If `get` returns a leftover stored `{ datasource }` telemetry rule, echo it unchanged unless the user asks to switch it to an integration.

## The iteration loop

After a successful create or update, run **preview** (BigQuery cost check).

Repeat until leftovers are acceptable or the user is satisfied:

1. `preview_virtual_dimension_draft` (`mode: "costs"`) — read `summary`, `totals.leftoverSharePercent`, and `totals.namedRulesSharePercent`
2. If leftovers are high → `search` / `query` to find unallocated spend
3. `update_virtual_dimension_draft` — add or reorder rules (must pass validation to persist)
4. If overlap / shadowing concerns → `virtual_dimension_overlap_matrix`
5. Go to step 1

**Heuristics:**

- **Leftover > ~10–20%:** add rules for the largest unclaimed buckets
- **Rule order wrong:** put narrower/specific rules **before** broader ones
- **Shadowing:** off-diagonal entries in the overlap matrix show spend captured by an earlier rule that also matches a later rule's condition
- **Before writing a rule:** `preview` with `mode: "breakdown"`, `ruleId` + `groupBy` to see top values inside a rule's scope

## Watch for rule shadowing

After adding a broad rule above a narrower one, a later rule can show **0** in preview because the new rule captures that spend too.

**After reordering or adding rules**, call `virtual_dimension_overlap_matrix` — off-diagonal entries show spend captured by an earlier rule that also matches a later rule's condition. If a rule you meant to keep shows 0 in costs preview, check overlap before publishing.

## VDIM tool reference

| Tool | Mutates? | Use |
|------|----------|-----|
| `list_virtual_dimensions` | no | Find VDIM by name/tag |
| `get` | no | Published + draft rules (CEL), dependencies (`type: "virtualDimension"`) |
| `create_virtual_dimension_draft` | draft | New VDIM |
| `update_virtual_dimension_draft` | draft | Full rules array replace (or metadata-only when `rules` omitted) |
| `preview_virtual_dimension_draft` | no | BQ: costs or breakdown |
| `virtual_dimension_overlap_matrix` | no | BQ: rule overlap / shadowing |
| `publish_virtual_dimension` | **yes** | Promote draft → production + refresh |

## Example: "Create an Environment virtual dimension"

1. `search` `{ query: "", type: ["dimensions"] }` → find `env` and its values; `{ query: "prod", type: ["dimensions"] }` → find values like `sub_account_id` containing "prod"
2. **Present mapping options** → user picks strategy
3. **Present proposed rules → explicit rule approval**
4. `create_virtual_dimension_draft` with approved prod/staging/dev rules
5. `preview_virtual_dimension_draft` `mode: "costs"` → leftover 35%
6. `search` for remaining unallocated values → `update_virtual_dimension_draft` with full rules array
7. `preview_virtual_dimension_draft` → leftover 4%
8. **[if needed]** `virtual_dimension_overlap_matrix` → check for rule shadowing
9. User confirms → `publish_virtual_dimension`

## Safety Rules / Anti-patterns

- Do not publish after preview/overlap when `draftPersisted: false` — call `update_virtual_dimension_draft` (or `create_virtual_dimension_draft` for a new VDIM)
- Do not publish without `preview` when leftovers are still high
- Do not expect create/update to save an invalid draft — they reject invalid payloads (`draftPersisted: false`)
- Do not send partial rule arrays to `update_virtual_dimension_draft` (always emit full end state when `rules` is provided)
- Do not include a catch-all / leftover rule in `rules` on create or update
- Do not omit `id` on rules you intend to keep — always carry forward each existing rule's `id`
- Do not reorder rules without moving `id` with the same logical rule
- Do not use draft-version ids — always use `virtualDimensionId`
- Do not put broad rules before narrow ones (causes shadowing)
- Do not call `create_virtual_dimension_draft` after strategy-only approval — get **explicit rule approval** first
- Do not invent VDIM bucket names from spend alone without asking what values the user expects
- Do not call `publish_virtual_dimension` before the user explicitly asks to go live
- Do not try to rename the leftover bucket via `update_virtual_dimension_draft` — use the Costory web app
- Do not derive `groupBy` / `filterCel` from the display `name` — always use immutable `bqName`
- Do not set `datasource` / `datasourceId` on new telemetry reallocations — persist inline `externalMetric` via `list_metrics` `{ includeExternal: true, search }`

## Related Skills / Next Steps

- `dashboards` — after publish, build charts that `groupBy` the new `bqName`
- `reports` — include the new axis in DIGEST hierarchy once `computeStatus` is `COMPLETED`
