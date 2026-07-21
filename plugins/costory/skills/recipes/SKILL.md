---
name: recipes
description: "Use when a user states a FinOps *outcome* rather than a tool. Route to one recipe card, then hand off: bill jump / what changed → explain-period-change (preview_report_widget DIGEST, not query+compare); marketplace / private-offer → marketplace-spend; K8s namespace showback → namespace-cost; credits runway → provider-credits; YTD credits → credits-ytd-explore; weekly env+service pulse → service-cost-weekly; tag coverage → untagged-coverage; exec env report → env-costs-cto; explore Prod vs R&D → prod-vs-rnd-explore; build Prod/R&D VDIM → prod-vs-rnd-vdim; reallocate by external metric → reallocate-by-external-metric; budget vs actual daily → budget-vs-actual-dashboard; EC2 +10% alert → ec2-cost-spike-alert; compute drill-down dimension → compute-drilldown-dimension. Skeletons in recipes; HOW in reports/dashboards/virtual-dimensions/query."
---

# Recipes

A **recipe** is a ready-made design for a common FinOps tracking goal — scope, split, period, widgets, delivery — as a **payload skeleton**, then a **hand off** to the mechanics skill (`reports`, `dashboards`, `virtual-dimensions`, `query`). Recipes never restate full delivery safety or live schema edge cases; fill placeholders after confirm, then build with the mechanics skill.

## How to use this skill

1. Match the user's goal using **Pick a recipe** below (triggers + "not this if").
2. **Read only that one file** under `plugins/costory/skills/recipes/`.
3. Run the card's **Tool sequence** through confirm gates — do not fire mutation/delivery tools until confirmed.
4. Fill `[PLACEHOLDERS]` from discovery + user answers; keep frozen fields as written.
5. Restate the **Brief**, then hand off the filled skeleton to the named mechanics skill.
6. If nothing matches → fall through to `reports` (or the relevant mechanics skill) and design from scratch.

## Skeleton contract

Every card has:

| Section | Role |
|---------|------|
| **When / Audience / Outcome** | routing + what "done" looks like |
| **Tool sequence** | ordered Costory MCP tool names (discover → confirm → preview → create) |
| **Payload skeleton** | JSON-shaped `context` / `widgets` / optional `query` with frozen defaults + `[PLACEHOLDERS]` |
| **Confirm before build** | gates that must be answered before preview/create/NOW/SCHEDULED |
| **Gotchas** | recipe-specific traps only |
| **Brief** | one-line restatement |
| **Hand off** | which mechanics skill owns the filled payload |

Placeholders are never invented: resolve via `get_context`, `search`, `suggest_groupby`, `list_available_destinations`, VDIM publish, or the user. Currency comes from `get_context`.

## Pick a recipe

Read the matching file. Do not improvise a blend of two cards until the user asks to combine them.

| If the user means… | Signals (phrases / audience) | You get | Not this if… | Read |
|--------------------|------------------------------|---------|--------------|------|
| **Why did spend move?** (one-shot) | "why did the bill jump", "what changed last month", "explain the spike", finance-review prep — **ad-hoc**, not recurring | **`preview_report_widget` DIGEST** change tree (chat-first; not `query`+`compare`); ± optional AI | standing weekly/monthly channel report → Schedule recipe; explorer PoP totals only → `query` Workflow B | `plugins/costory/skills/recipes/explain-period-change.md` |
| **Marketplace / vendor spend** | "marketplace", "private offer", "third-party on the cloud bill", "spend by seller/issuer" — Finance / procurement | Monthly marketplace-only trend + top vendors by `cos_invoice_issuer` | native cloud usage (EC2 etc.) — wrong recipe | `plugins/costory/skills/recipes/marketplace-spend.md` |
| **K8s cost per namespace** | "namespace", "EKS/GKE", "cluster showback", platform/DevOps weekly | Weekly reallocated-namespace trend + top/flop | env or account rollup without namespaces → `service-cost-weekly` or `env-costs-cto` | `plugins/costory/skills/recipes/namespace-cost.md` |
| **Credits / discounts runway** | "credits burning", "savings plan / CUDs", "promotional credits", "discount lines", charge category | Monthly charge-category trend + movers | usage-by-service operating view → `service-cost-weekly` | `plugins/costory/skills/recipes/provider-credits.md` |
| **Weekly eng FinOps pulse** | "weekly by env and service", "weekly pulse", "what moved per service" — eng/FinOps, **actionable drivers** | Weekly graph + DIGEST env/account → service, **AI on** | exec-only "prod vs staging" one number → `env-costs-cto`; one-shot "why did it jump" → explain | `plugins/costory/skills/recipes/service-cost-weekly.md` |
| **Tagging / allocation coverage** | "untagged", "tag coverage", "missing team/env tag", "can't allocate the bill" | `tagged / all` ratio trend + worst services (formula via `query` first) | absolute spend by tag value (that's a normal split, not coverage) | `plugins/costory/skills/recipes/untagged-coverage.md` |
| **Exec cost per environment** | "CTO wants env costs", "prod vs non-prod", leadership monthly readout — **simple**, not drill-down | Monthly top envs (no flop) ± yearly graph; **build `env` VDIM first** | weekly service drivers / AI narrative → `service-cost-weekly`; one-shot explore → `prod-vs-rnd-explore` | `plugins/costory/skills/recipes/env-costs-cto.md` |
| **Reallocate by external metric** | "split shared cost by usage", "unit economics then allocate", "showback by requests/revenue/CPU", proportional fair share | Validate cost ÷ metric in `query`, then **telemetry VDIM** publish | simple cost ÷ metric KPI only (no reallocation) → `query` Workflow E; fixed 60/40 weights → `virtual-dimensions` splitCost | `plugins/costory/skills/recipes/reallocate-by-external-metric.md` |
| **Explore Prod vs R&D** | "explain Prod vs R&D", "how much is production vs research" — **ad-hoc** split | Composition (and optional MoM) on published env / Prod-R&D `bqName` | no axis yet → `prod-vs-rnd-vdim`; standing monthly exec report → `env-costs-cto` | `plugins/costory/skills/recipes/prod-vs-rnd-explore.md` |
| **Budget vs actual daily** | "budget vs actual per day", "daily burn vs budget", "on pace this month?" — dashboard | Daily cumulated cost + budget (+ optional utilization) dashboard | one-shot numbers only → `query` Workflow F | `plugins/costory/skills/recipes/budget-vs-actual-dashboard.md` |
| **Credits received YTD** | "how much credits this year", "YTD promotional credits" — **explore total** | YTD credit total + charge-category breakdown | standing monthly runway / movers → `provider-credits` | `plugins/costory/skills/recipes/credits-ytd-explore.md` |
| **EC2 +10% spike alert** | "warn if AmazonEC2 up >10%", "EC2 spike alert", "notify on EC2 WoW jump" | Alert: 7-day sum > prior 7-day × 1.1 (preview then create) | dashboard/report trend without notify → `dashboards` / `reports` | `plugins/costory/skills/recipes/ec2-cost-spike-alert.md` |
| **Build Prod vs R&D VDIM** | "virtual dimension for prod vs R&D", "map production from research costs" — **axis first** | Draft → preview → publish Production / R&D rules | already have the axis and want numbers → `prod-vs-rnd-explore`; exec schedule → `env-costs-cto` | `plugins/costory/skills/recipes/prod-vs-rnd-vdim.md` |
| **Compute drill-down dimension** | "what dimension for compute", "best groupBy for EC2/VMs", "break down compute spend" | Scoped `suggest_groupby` + sample query; ranked axes | they already named the axis and want a dashboard → `dashboards` | `plugins/costory/skills/recipes/compute-drilldown-dimension.md` |

### Quick disambiguation

| Sound alike | Prefer |
|-------------|--------|
| "cost per environment" from a **CTO / VP** (monthly, simple) | `env-costs-cto` |
| "cost per environment **and service**" / **weekly** eng pulse | `service-cost-weekly` |
| "explain Prod vs R&D" **once** vs **build the VDIM** vs **monthly Slack** | `prod-vs-rnd-explore` / `prod-vs-rnd-vdim` / `env-costs-cto` |
| "what changed?" **once** vs **every week/month in Slack** | explain-period-change (DIGEST preview) vs the Schedule recipe that matches the topic |
| "what changed?" **drivers tree** vs explorer PoP **numbers** | explain-period-change vs `query` Workflow B |
| "untagged spend $" vs "tag **coverage %**" | raw untagged $ can be a scoped query; **coverage campaign** → `untagged-coverage` |
| "cost per request" KPI vs **reallocate** shared spend by requests | unit economics only → `query`; proportional split → `reallocate-by-external-metric` |
| "credits this year" **total** vs credits **runway report** | `credits-ytd-explore` vs `provider-credits` |
| "budget vs actual" **dashboard** vs quick Explorer numbers | `budget-vs-actual-dashboard` vs `query` Workflow F |

## Notes on presets

Cards use real `datePreset` tokens (`LAST_MONTH`, `LAST_WEEK`, `LAST_INVOICE_MONTH`, `TRAILING_14_WEEKS`, `LAST_12_MONTHS`, `MTD`, `YTD`, `TRAILING_90_DAYS`) — all present in the live DatePreset enum.

## Adding a recipe

One file in `plugins/costory/skills/recipes/` + one row in **Pick a recipe** (triggers, outcome, "not this if"). Card format:

**When · Audience · Outcome · Tool sequence · Payload skeleton · Confirm before build · Gotchas · Brief · Hand off**
