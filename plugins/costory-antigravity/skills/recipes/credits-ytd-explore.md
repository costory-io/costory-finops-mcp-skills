# Credits received YTD (explore)

**When:** *"how much credits did we receive this year?"*, *"YTD promotional credits"*, *"total credit amount year to date"* — one-shot credit total, not a runway report.
**Audience:** FinOps / finance closing or planning.
**Outcome:** YTD credit (and optionally discount) dollars, with a charge-category breakdown so the total is explainable.

## Tool sequence

1. `get_context` → currency
2. `search` `{ type: ["dimensions"], query: "credit" }` (also try `"charge category"`) → discover real `cos_charge_category` values and any credit-related fields → `[CREDIT_CEL]`
3. Confirm with the user: **credits only** vs credits + discounts / charge-category mix
4. `query` totals with skeleton A, then breakdown with skeleton B
5. Optional: monthly trend (`aggBy: "Month"`) if they ask "when did credits land?"

## Payload skeleton

**A — YTD credit total:**

```json
{
  "datePreset": "YTD",
  "aggBy": "Period",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "Credits YTD",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "filterCel": "[CREDIT_CEL]"
  }]
}
```

**B — YTD by charge category:**

```json
{
  "datePreset": "YTD",
  "aggBy": "Period",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "Credits by charge category",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "groupBy": "cos_charge_category",
    "filterCel": "[CREDIT_CEL]",
    "chartType": "BAR"
  }]
}
```

Frozen: period `YTD`; breakdown `groupBy` = `cos_charge_category`. **`[CREDIT_CEL]` is not frozen** — resolve from `search` + user confirm (typical shape: `cos_charge_category in ["Credit", …]` with org-specific labels). Credit lines are often **negative** — report the signed total and say so.

## Confirm before build

1. Credits-only CEL vs broader discount / charge-category mix
2. Calendar YTD vs last invoice year (if they mean fiscal/invoice, use explicit `from`/`to` or `LAST_12_MONTHS`)
3. Whole org vs provider / account scope

## Gotchas

- Cousin of `provider-credits` (monthly **scheduled runway** report). This card is **explore YTD total**.
- Never invent charge-category string values — always discover via `search`.
- Negative credit amounts are expected; don't flip the sign unless the user asks for "credits received" as a positive magnitude (then state you took `abs`).

**Brief:** *"YTD credits explore with [CREDIT_CEL]: total + charge-category breakdown[, monthly trend]."*

**→ Hand off to `query`**. For a standing monthly runway report → `provider-credits`.
