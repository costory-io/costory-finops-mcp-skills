---
name: bigquery
description: "Use when the user asks about BigQuery costs, SKUs, on-demand Analysis vs capacity slots/editions, leftover on-demand, physical vs logical storage, time travel or fail-safe, streaming ingest, load/import jobs consuming a reservation, dbt or job labels, slot cost reallocation, dataset/table storage allocation, or unlabeled expensive queries. Call get_skill with skillId \"bigquery\" before investigating BigQuery spend or cloning the [BigQuery] dashboard. Not for GCP spend-based CUDs (recipes gcp-spend-cud)."
---

# BigQuery

Invoice SKUs first. Bytes scanned, slot-ms, and table storage are **not on the bill**. Teach slots vs on-demand, open **[BigQuery] BigQuery Dashboard**, then allocate or optimize.

**Not this if:** GCP spend CUD leftover → `recipes` → `gcp-spend-cud`. Generic Explorer → `query`. Whole-bill "what changed" → `recipes` → `explain-period-change`.

## Workflow

1. `get_context` — currency; note `bq_*` / `user_email` if already ingested
2. `search` `{ type: ["dashboards"], query: "BigQuery" }` — use the template **[BigQuery] BigQuery Dashboard**. Org copies can differ. **Never hardcode an id.**
3. `get` that template for the URL. Do not walk widgets.
4. Teach **Slots vs on-demand** (two sentences): *On-demand = pay per TiB scanned. Slots = pay for a pool; unused slots still cost, and bytes no longer price the job.* Then the SKU table, then the caveat that matches the ask.
5. Read the dashboard (order below). Branch: leftover Analysis → assignment / load jobs; storage → `TABLE_STORAGE`; showback → **Allocate**; no labels → **Jobs metadata**.
6. Costory `query` only after the dashboard. Persist via `query` / `dashboards` / `virtual-dimensions`. No `create_dashboard` unless they ask to clone.

## Slots vs on-demand

A **slot** is a unit of BigQuery compute. `total_slot_ms` in `JOBS` is how hard the query worked — it is **not** the invoice line unless you buy slots.

| | On-demand | Slots (capacity / editions) |
|--|-----------|-----------------------------|
| What you buy | Nothing in advance | A **reservation** assigned to a project, folder, or org |
| Invoice meter | Bytes scanned (`total_bytes_billed`) | Slot-hours reserved or autoscaled, used or not |
| Costory SKU | `cos_sku` contains `Analysis` (not `Slots`) | `cos_sku` contains `Edition` (Standard / Enterprise / Enterprise Plus); service often `BigQuery Reservation API` |
| Narrow scan | Cheap | You still pay the reservation |
| Wide `SELECT *` | Expensive (TiB) | Same reservation price; the job occupies more slots / queues |
| LOAD / COPY / EXTRACT | **Free** | **Uses the reservation** if that job type is assigned |
| Idle | $0 | Baseline / committed slots still bill |
| Who used the $ | Each job’s bytes | Shared pool. Invoice does **not** split Edition SKUs per query unless Costory **slot reallocation** is on |

**On-demand shop:** every QUERY job bills `Analysis`. Lever = partition / cluster / avoid `SELECT *` / cache / `maximum_bytes_billed`.

**Slot shop:** bytes stop pricing jobs **that run in the reservation**. Lever = reservation size, autoscaling max, assignment scope, keep LOAD off the pool. Some `Analysis` in a slot shop is **leftover on-demand** (job missed the reservation) — not “slots billed as Analysis”.

Reservations are **regional**. Project A can be reserved and B on-demand; `US` reserved does not cover `EU`. Editions are *which* slot product (features + $/slot-hour), not a third billing model.

## SKU families

```text
[BQ] = cos_service_name in ["BigQuery", "BigQuery Reservation API", "BigQuery Storage API"]
```

Template uses `contracted_cost` (pre-credit). Use `cost` only when they want billed-after-credits.

| Family | `cos_sku` | Meter | Lever |
|--------|-----------|-------|-------|
| On-demand analysis | `Analysis`, not `Slots` | Bytes scanned | Query hygiene |
| Capacity slots | `Edition` | Reserved / autoscaled slot-hours | Size vs leftover Analysis; LOAD off the reservation |
| Storage | `Logical` or `Physical`; `Active` vs `Long Term` | Bytes stored | Billing model, time travel, expiration |
| Streaming | `Streaming Insert` | Ingest | Batch load or Storage Write API |
| Network | `Networking` / replication | Cross-region bytes | Dataset location |
| Storage API | service `BigQuery Storage API` | Read / Write API | Spark / Storage API clients |

- **Leftover Analysis** (Analysis **and** Edition both large): jobs missed the reservation (wrong project / folder) or ran in a region with no reservation.
- **Query cache:** identical on-demand queries can bill **0** analysis bytes until the table changes.
- **Long-term storage:** rate drops after **90 days** untouched. Expiration is cheaper if nobody reads the data.

## Compute

On-demand, LOAD / COPY / EXTRACT (and many Data Transfer / `bq load` jobs) are **free**. On a reservation they **consume slots** unless the assignment is `QUERY` only — then pipelines steal capacity and interactive / dbt work spills to Analysis.

- Assign **QUERY only** to the paid reservation. Leave `PIPELINE` on-demand or on a cheap/isolated pool.
- Smoking gun in `JOBS`: `job_type IN ('LOAD','COPY','EXTRACT')` and `reservation_id IS NOT NULL`.
- Unassigned projects in a slot org still pay Analysis. Autoscaling: unused baseline is waste; max you hit every hour can still overflow to on-demand.
- Cross-region queries and authorized views in another region can miss the reservation and add network SKUs.

## Storage — logical vs physical

BigQuery always stores compressed data. The dataset **billing model** only changes **which bytes you are charged for**. Default **logical**. Switch is **per dataset** (not per table). **14-day** lock before switching again; billing catches up in ~**24 hours**.

| | Logical (default) | Physical |
|--|-------------------|----------|
| Meter | Uncompressed size | Compressed size on disk |
| Active (US multi-region list) | ~$0.02 / GiB / mo | ~$0.04 / GiB / mo |
| Long-term (90 days untouched) | ~$0.01 / GiB / mo | ~$0.02 / GiB / mo |
| Time travel (2–7 days, default 7) | Included | **Extra**, at **active physical** |
| Fail-safe (fixed **7 days** after time travel) | Included | **Extra**, at **active physical** |

Physical often wins on compressible append-only data (logs, events). **Do not move high-churn tables** (`MERGE` / `UPDATE` / `DELETE` / daily full refresh): old versions stay in time travel, then **7 more days** fail-safe, billed at active physical. High **Active Physical** vs tiny **Long-Term Physical** on the dashboard is usually churn, not “new data”.

You can shorten time travel (`max_time_travel_hours`, min **48**) at dataset or table. You **cannot** shorten fail-safe.

Forecast from `TABLE_STORAGE`, **not** invoice dollars. List prices above are **US multi-region** — change them for EU / other regions. A mixed dataset (mutating fact + append-only logs) can make physical lose even if some tables look great: split first, or only switch when the **dataset** rollup still wins after time travel + fail-safe.

## Dashboard

Name: **[BigQuery] BigQuery Dashboard**. Scope: `[BQ]`, last 3 complete months, split SKU, `contracted_cost`.

1. Monthly family trend — slot shop, on-demand shop, or **both** (leftover Analysis)
2. Compute — Edition vs Analysis
3. Storage — logical vs physical, then active vs long-term, then streaming. Empty physical = still on default logical
4. Where it runs — `cos_region`, `cos_sub_account_id` (invoice-native; no job labels needed)
5. What moved — SKU table + waterfall vs the previous 3 months
6. Next layer (text on the template) — `INFORMATION_SCHEMA` for bytes / slot-ms / query hash; table storage for a physical forecast

If job labels or BQ telemetry are connected, `search` `type: ["dimensions"]` for `user_email`, `bq_referenced_tables`, and ingested keys (`node_name`, …) before adding widgets.

## Allocate

Three grains. Invoice `cos_resource_id` is **not** a table.

- **Analysis / job metadata:** job query labels. dbt: [Increase visibility on your BigQuery costs with dbt](https://docs.costory.io/use-cases/dbt_bigquery_visibility/increase_visibility) — `query-comment` + `job-label: True` (`node_name`, `package_name`, `schema`, `target_name`, …). Ingest in Costory feature engineering; optional VDIM for team / product.
- **Slots:** dollars land on the reservation. **Reallocation is not self-serve** — ask the Costory team (billing-export slot usage → jobs → labels). Until then, `JOBS.total_slot_ms` shows *who used slots*; the invoice will not split Edition SKUs.
- **Storage:** `cos_resource_id` is the **dataset**. Per-table bytes live in `TABLE_STORAGE`. **Dataset labels** (`ALTER SCHEMA … SET OPTIONS(labels=…)`) show up on the bill after ingest. Table labels are operational only (`TABLE_OPTIONS`) — not a billing dimension.

Map ingested labels to team / product with `virtual-dimensions`. Preview + user confirm before publish.

## Jobs metadata (no labels required)

Do not block on tagging. Invoice-native first: `[BQ]` by `cos_sku`, `cos_sub_account_id`, `cos_region`, storage `cos_resource_id`. Then `user_email` / `bq_referenced_tables` if `search` found them.

Write SQL in the **project + region that ran the jobs**. Qualify views as `` `region-us`.INFORMATION_SCHEMA.… `` (change region). Last 7 days is enough.

**Expensive QUERY jobs** — `JOBS_BY_PROJECT`, `job_type = 'QUERY'`, `parent_job_id IS NULL`, `state = 'DONE'`. Group `user_email`, `query_info.query_hashes.normalized_literals` (`query_hash`), and `referenced_tables` (unnest → `project.dataset.table`). Sum `total_bytes_billed` and `total_slot_ms`. Order by bytes billed. `query_hash` finds repeat SQL; `referenced_tables` finds hot tables; `user_email` is often a service account (dbt, Looker, scheduled query).

**Load jobs on the reservation** — same view, `job_type IN ('LOAD','COPY','EXTRACT')` and `reservation_id IS NOT NULL`. Group `job_type`, `reservation_id`, `user_email`. Should be empty or tiny.

If they want this as Costory dimensions, **ask the Costory team** — do not invent a connector.

## Physical vs logical forecast

`TABLE_STORAGE_BY_PROJECT`, `table_type = 'BASE TABLE'`, per region. Convert bytes → GiB (`/ POWER(1024, 3)`).

```text
logical  = active_logical * 0.02 + long_term_logical * 0.01
physical = active_physical * 0.04 + long_term_physical * 0.02
           + (time_travel_physical + fail_safe_physical) * 0.04
churn    = (time_travel_physical + fail_safe_physical) / active_physical
```

US multi-region list prices — change them before trusting $. Include time travel + fail-safe or the forecast is a lie.

- `physical < 0.8 * logical` and `churn < 0.3` → `GOOD_CANDIDATE`
- Saves on paper but `churn >= 0.3` → `SAVES_ON_PAPER_BUT_CHURNY`
- Else `KEEP_LOGICAL`

The `ALTER` grain is the **dataset**. Roll up the same numbers with `GROUP BY dataset`; count tables that are losing or churny. Skip datasets already on physical (`SCHEMATA_OPTIONS.storage_billing_model`). Savings **and** many risky tables → split the dataset first.

If the dataset rollup is a clear win **and the user confirms**:

```sql
ALTER SCHEMA my_dataset SET OPTIONS(storage_billing_model = 'PHYSICAL');
-- optional, before or after: max_time_travel_hours = 48  (fail-safe stays 7)
```

Never run `ALTER SCHEMA` / `ALTER TABLE` without explicit confirmation.

## Costory follow-up

Prefer `datePreset: "LAST_3_MONTHS"` to match the template. Fill slug / currency from `get_context`. Shared scope = `[BQ]` above.

| Ask | CEL add-on | Split |
|-----|------------|-------|
| Leftover on-demand vs slots | `(cos_sku.contains("Analysis") && !cos_sku.contains("Slots"))` vs `cos_sku.contains("Edition")` | two series, `aggBy: Month` |
| Storage model | `contains("Logical")` vs `contains("Physical")` | then split Physical by `cos_sku` (Active vs Long-Term) |
| Storage by dataset | Logical **or** Physical | `groupBy: cos_resource_id` |
| Who ran jobs | Analysis only | `groupBy: user_email` — **only** if `search` found that dimension |

Do not invent `user_email` / `bq_referenced_tables` / label fields.

**Optional levers** (no skeleton unless asked): partition/cluster hot tables from Jobs; `QueryUsagePerUserPerDay` quotas; alerts on stable ingest vs full-refresh; streaming → batch.

## Safety

- Never hardcode a dashboard id; do not `create_dashboard` unless they ask to clone
- Do not forecast physical vs logical from invoice SKUs
- Do not recommend physical on high `churn` or frequent `MERGE`/`UPDATE`/full-refresh
- No `ALTER` without confirm; mention the **14-day** lock and **24h** billing lag
- Leftover `Analysis` is a missed reservation, not “slots billed twice”
- Do not claim slot dollars split by query unless reallocation is enabled
- Do not put LOAD/COPY/EXTRACT on the QUERY reservation
- `publish_virtual_dimension` needs preview + confirm
- US list prices ≠ EU; last 1–2 billing days can be incomplete

## Related

- `query` — leftover / storage / dataset / user slices
- `dashboards` — clone or extend the template
- `virtual-dimensions` — team / product from job or dataset labels
- `recipes` → `gcp-spend-cud` — Flexible / spend CUDs, not BQ SKUs
- `recipes` → `explain-period-change` — one-shot bill jump on a BQ-scoped DIGEST
- Docs: [dbt + BigQuery visibility](https://docs.costory.io/use-cases/dbt_bigquery_visibility/increase_visibility)
