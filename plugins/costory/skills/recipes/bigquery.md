# BigQuery warehouse (dashboard + caveats)

**When:** *"explain BigQuery costs"*, *"on-demand vs slots"*, *"physical vs logical"*, *"why is Analysis still high"*, *"load jobs using our reservation"*, *"allocate BQ by dbt model"*, *"we have no job labels"* — warehouse billing, not GCP CUDs.
**Audience:** FinOps / data platform watching BigQuery.
**Outcome:** they understand SKU families and the known caveats, have the **[BigQuery] BigQuery Dashboard** template, and can allocate or optimize (Jobs metadata / physical-vs-logical forecast).

**Not this if:** GCP spend-based CUD leftover / utilization → `gcp-spend-cud`. Whole-bill "what changed last month" → `explain-period-change`. Generic Explorer outside BigQuery → `query`.

## Tool sequence

1. `get_skill` with `skillId: "bigquery"` — **stop and follow that skill**. Do not invent widgets or CEL from this card.
2. That skill owns dashboard discovery (search by **name**, never an id), teaching, allocation, and SQL.

## Confirm before build

None on this card. The `bigquery` skill asks before `ALTER SCHEMA`, dashboard clone, VDIM publish, or pinging Costory for slot reallocation.

## Gotchas

- Cousin of `gcp-spend-cud` (Flexible / dollar commitments), not BQ Analysis / Edition / storage SKUs.
- Never put the dashboard id in a payload — `search` **[BigQuery] BigQuery Dashboard** (often a template).

**Brief:** *"Load skill `bigquery`: teach drivers and caveats, open [BigQuery] BigQuery Dashboard, then allocate or optimize."*

**→ Hand off to `bigquery`.**
