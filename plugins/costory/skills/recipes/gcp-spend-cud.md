# GCP spend-based CUD (dashboard + questions)

**When:** *"explain GCP spend-based CUDs"*, *"how do I use GCP Spend-Based Commitments"*, *"why is leftover fee up"*, *"are we under-using CUDs"*, *"CUD utilization / coverage"* — concept + existing dashboard, then follow-up queries.
**Audience:** FinOps watching GCP Flexible / dollar-based commitments.
**Outcome:** they understand spend-based CUDs, have the **GCP Spend-Based Commitments** dashboard (search by **name**, never hardcode an id), and can run leftover / hourly / project / SKU / family-drift questions when leftover / committed fee is **≥ 2%**. Rate and net savings live on the dashboard — do not re-query them.

**Not this if:** monthly charge-category credit runway → `provider-credits`. Resource-based CUDs only (`cos_commitment_discount_category` = `Usage`) → design from `query`. Standing Slack pulse with no CUD concept → `reports`. BigQuery Analysis / slots / storage warehouse → `bigquery`.

## Concept (teach this first)

A **spend-based CUD** (FOCUS `Spend` / GCP `COMMITTED_USAGE_DISCOUNT_DOLLAR_BASE`) is a **$/hour commitment** for 1 or 3 years on eligible SKUs — Flexible CUDs, Cloud Run / Functions spend commitments. You buy a **fee**, then eligible **usage** offsets that fee.

**Consumption model** (what this dashboard assumes): billing shows the **full fee SKU** plus a **FEE_UTILIZATION_OFFSET**. In Costory:

| Signal | Charge category | Metric | Meaning |
|--------|-----------------|--------|---------|
| **Committed fee** | `Purchase` | `list_cost` | Contracted hourly fee |
| **Leftover fee** | `Purchase` | `cost` (billed) | Fee still on the invoice after offsets. **Healthy = leftover / committed fee < 2%** (treat as ~$0) |
| **Discounted usage** | `Usage` + Spend category | `cost` | Usage that attached to the CUD |
| **Attachable uncovered** | `Usage`, no commitment id | `contracted_cost` | On-demand cores / RAM / serverless CPU-memory that *could* have burned a spend CUD |

**Hourly, no rollup.** GCP settles each hour independently. A weekday surplus does **not** cover a weekend (or night) shortfall. A healthy **month** average can still hide unused hours. Diagnose leftover at **Day**, then **Hour** on a short window — never treat a monthly leftover as "we were 80% utilized so we are fine."

**What can burn a spend CUD:** instance cores/RAM, Cloud SQL vCPU/RAM, Cloud Run / Functions CPU, memory, min instances.

**What cannot:** disks, egress / network, GKE cluster management fees, Spot. Those inflate "on-demand on CUD services" without reducing leftover.

**Project ≠ commitment owner.** Discounted usage often lands on the project that ran the VM. Leftover fee often lands on the **billing / org** project that owns the subscription. Always split leftover and discounted usage by `cos_sub_account_id` separately.

**Rate and net savings** are on the **GCP Spend-Based Commitments** dashboard. Do not rebuild those formulas in `query`. A lower rate than the published term (Compute Flexible 3-year is about **46%**) usually means leftover fee, not a bad list price.

## Dashboard

`search` `{ type: ["dashboards"], query: "Spend-Based Commitments" }` then `get` — name is **GCP Spend-Based Commitments**. Point the user at it. Do **not** restate or catalogue its widgets; they can open the dashboard.

Scope on that dashboard is last month, GCP + Spend. Coverage queries break that filter so uncovered eligible usage is visible.

**Read order** (same story as A–D; do not walk widget titles):

1. Leftover vs committed fee — is leftover / fee **< 2%**?
2. Daily leftover vs attachable uncovered — idle hours or cannot attach?
3. Hourly leftover on one leftover day — CUD does not roll up.
4. Which project burned the CUD vs which project is billed leftover.

## Decide (after A + B)

| Leftover / fee | Attachable uncovered | Meaning | Next |
|----------------|----------------------|---------|------|
| **< 2%** | low | Sized well | Stop; dashboard link is enough |
| **< 2%** | high | Coverage gap (opportunity unused) | **E** — which SKUs to cover; consider buying more / moving work onto Flexible |
| **≥ 2%** | down on leftover days | Unused hours (nights / weekends) | **C** then **D** — shrink, or shift batch into those hours |
| **≥ 2%** | up on leftover days | Mismatch (product / region / term), not idle | **E** + **D** — wrong CUD type, or a different commitment |

Both leftover **and** attachable can be high at once (unused commitment + unused opportunity).

## Tool sequence

1. `get_context` → currency; resolve **GCP Spend-Based Commitments** via `recentDashboards` or `search`
2. `get` that dashboard (for the URL / confirm it exists) — do not walk widgets
3. Teach Concept (above)
4. Leftover share and rate: read the dashboard (or **A** for the two fee numbers). Apply the 2% leftover / fee rule. If not healthy, run **B** and the Decide table — then **C** / **D** / **E** as that table says
5. **F** when leftover rose while the fee stayed flat, after **E** does not explain it, or the user asks whether the fleet moved
6. Optional: `list_events` on CUD start/stop if GCP CUD Metadata is imported

## Payload skeleton

Shared CEL (frozen unless the user narrows to one `cos_commitment_discount_id`):

```text
[SPEND] = cos_provider in ["GCP"] && cos_commitment_discount_category in ["Spend"]
[USAGE] = [SPEND] && cos_charge_category in ["Usage"]
[ATTACHABLE] = cos_provider in ["GCP"] && cos_commitment_discount_id == null && cos_charge_category in ["Usage"] && !(cos_pricing_model in ["Spot"]) && (cos_line_item_usage_type.contains("Instance Core") || cos_line_item_usage_type.contains("Instance Ram") || cos_line_item_usage_type.contains("vCPU") || cos_line_item_usage_type.contains(" RAM") || cos_sku.contains("Services CPU") || cos_sku.contains("Services Memory") || cos_sku.contains("Services Min Instance"))
```

**A — leftover KPI (start here):**

```json
{
  "slug": "[SLUG]",
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Committed fee", "metricId": "list_cost", "currency": "[CURRENCY]", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" },
    { "type": "cost", "name": "b", "alias": "Leftover fee", "metricId": "cost", "currency": "[CURRENCY]", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" }
  ]
}
```

**B — daily leftover vs attachable uncovered (under-use source):**

```json
{
  "slug": "[SLUG]",
  "datePreset": "TRAILING_90_DAYS",
  "aggBy": "Day",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Leftover fee", "metricId": "cost", "currency": "[CURRENCY]", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" },
    { "type": "cost", "name": "b", "alias": "Attachable uncovered", "metricId": "contracted_cost", "currency": "[CURRENCY]", "filterCel": "[ATTACHABLE]" }
  ]
}
```

**C — hourly leftover (no rollup; short window only):**

```json
{
  "slug": "[SLUG]",
  "from": "[LEFTOVER_DAY]",
  "to": "[LEFTOVER_DAY]",
  "aggBy": "Hour",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Leftover fee", "metricId": "cost", "currency": "[CURRENCY]", "groupBy": "cos_line_item_usage_type", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" }
  ]
}
```

**D — which project burned vs which project paid leftover:**

```json
{
  "slug": "[SLUG]",
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Discounted usage", "metricId": "cost", "currency": "[CURRENCY]", "groupBy": "cos_sub_account_id", "filterCel": "[USAGE]" },
    { "type": "cost", "name": "b", "alias": "Leftover fee", "metricId": "cost", "currency": "[CURRENCY]", "groupBy": "cos_sub_account_id", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" }
  ]
}
```

**E — attachable SKU mix (buy more vs move work):**

```json
{
  "slug": "[SLUG]",
  "datePreset": "LAST_MONTH",
  "aggBy": "Period",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Attachable uncovered", "metricId": "contracted_cost", "currency": "[CURRENCY]", "groupBy": "cos_sku", "filterCel": "[ATTACHABLE]" }
  ]
}
```

**F — machine-family drift (fee flat, attachment mix moved):**

```json
{
  "slug": "[SLUG]",
  "datePreset": "LAST_6_MONTHS",
  "aggBy": "Month",
  "queries": [
    { "type": "cost", "name": "a", "alias": "Discounted usage", "metricId": "cost", "currency": "[CURRENCY]", "groupBy": "cos_machine_family", "filterCel": "[USAGE]" },
    { "type": "cost", "name": "b", "alias": "Committed fee", "metricId": "list_cost", "currency": "[CURRENCY]", "filterCel": "[SPEND] && cos_charge_category in [\"Purchase\"]" }
  ]
}
```

`b` has no `groupBy` — one fee line. `a` is the mix. Spend CUDs are **not** family-locked (unlike resource CUDs); **F** is composition, not eligibility. A null / empty family is Cloud Run, Cloud SQL, or other non-VM attach — use **E** for those SKUs. If fee is flat and leftover rose while `a` shifted toward empty family or a new family, the leftover is often ineligible SKUs, not "we bought the wrong family CUD."

Do **not** add formula series. Leftover share, effective rate, and net savings are dashboard widgets.

Frozen: dashboard name **GCP Spend-Based Commitments**; Spend-category scope; leftover = Purchase `cost`; leftover share = leftover / committed `list_cost` on the dashboard (**healthy < 2%**); attachable uses `contracted_cost` + `[ATTACHABLE]` (not all on-demand on CUD services); under-use grain is **Day then Hour**, never Month-only; **F** is `LAST_6_MONTHS` / `Month` / `cos_machine_family`.

## Confirm before build

1. Org slug (multi-org)
2. Last month vs MTD vs a leftover day for Hour
3. All spend CUDs vs one `cos_commitment_discount_id`
4. Whether they want follow-up queries in chat or only the dashboard link

## Questions (run these; do not invent others first)

**Under-used (required):**

1. Is leftover share **< 2%**? → **A**
2. On leftover days, did attachable uncovered fall (unused hours) or rise (cannot attach)? → **B**, then Decide
3. Which hours on that day still show leftover? CUD is **per hour and does not roll up**. → **C**
4. Which project consumed the CUD vs which project is billed leftover? → **D**

**When Decide says so (do not invent CEL):**

5. **Oversized vs mismatched** — leftover share ≥ 2% *and* attachable up → mismatch, not idle. Use Decide; do not add a fifth query.
6. **Attachable SKU mix** — **E**
7. **Effective rate vs published term** — dashboard, not a query
8. **Net savings after waste** — dashboard, not a query
9. **Machine-family drift** — **F** (fee flat vs attachment mix). Not a family-lock test.

**Optional (no skeleton — do not invent one unless the user asks):**

10. **Coverage vs utilization** — leftover = unused commitment; attachable = unused *opportunity*. Both can be true (Decide).
11. **Expiry / start** — `list_events` if CUD Metadata is connected.
12. **Spend vs resource CUD mix** — this card is Spend only; resource-based (`Usage` category) → `query`, not this recipe.

## Gotchas

- Do **not** diagnose leftover with "on-demand on CUD services" — disks and network cannot burn the fee.
- Monthly utilization **averages away** unused hours. Always drop to Day / Hour.
- Org-level CUDs often have an odd `cos_sub_account_id` label on leftover — that is the commitment owner, not a mystery project ([docs](https://docs.costory.io/docs/standard-columns#gcp-5)).
- Never put the dashboard id in the skill. `search` by **GCP Spend-Based Commitments**.
- Cousin of `provider-credits` (invoice charge-category runway), not utilization.

**Brief:** *"Open GCP Spend-Based Commitments for leftover share, rate, and savings. If leftover share ≥ 2%: leftover vs attachable (Decide) → hourly no-rollup → project burn vs leftover → SKU mix if Decide says so → family drift (F) if fee stayed flat."*

**→ Hand off to `query`.** Dashboard already exists — do not `create_dashboard` unless they ask for a copy.
