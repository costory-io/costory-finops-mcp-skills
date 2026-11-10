# Untagged coverage

**When:** *"how much of our cost is untagged?"*, *"tag coverage?"*, *"which services are missing team/env/cost-center?"*, *"we can't allocate half the bill — where are the gaps?"* — allocation hygiene, not absolute spend.
**Audience:** FinOps lead / platform team running a tagging campaign.
**Outcome:** a **coverage ratio** over time plus the services dragging it down — a number the team can move.

## Tool sequence

1. `get_context` → currency (+ popular groupBys as hints only)
2. **Always ask** which tag/label dimension defines coverage (team, env, cost-center, …) — even if they already named one in chat. Then `search` `type: ["dimensions"]` to resolve the exact CEL field → `[TAG_FIELD]`. Do not guess; do not proceed until the user confirms the field.
3. Confirm what is legitimately un-taggable → `[SCOPE_CEL]` (default exclude marketplace: `cos_marketplace_purchase == false`)
4. `query` with formula skeleton below — eyeball both legs + ratio on a known period (`datePreset`, not hand-picked dates)
5. Confirm cadence + channel type → `list_available_destinations`
6. Confirm `SCHEDULED` → `create_report` reusing the **same** formula (shared fields in `reportContext`; hand off to `reports` for inheritance)

## Payload skeleton — validate in `query` first

```json
{
  "datePreset": "[LAST_MONTH|TRAILING_30_DAYS]",
  "aggBy": "Period",
  "queries": [
    {
      "type": "cost",
      "name": "a",
      "alias": "Tagged",
      "metricId": "cost",
      "currency": "[CURRENCY]",
      "groupBy": "cos_service_name",
      "filterCel": "([SCOPE_CEL]) && ([TAG_FIELD] != null)"
    },
    {
      "type": "cost",
      "name": "b",
      "alias": "All in scope",
      "metricId": "cost",
      "currency": "[CURRENCY]",
      "groupBy": "cos_service_name",
      "filterCel": "[SCOPE_CEL]"
    },
    {
      "type": "formula",
      "name": "c",
      "alias": "Tag coverage",
      "formula": "a / b"
    }
  ]
}
```

Adjust tagged leg if "tagged" means a specific value (`[TAG_FIELD] == "[EXPECTED]"`) rather than non-null.

## Payload skeleton — report (after formula looks sane)

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "[WEEKLY|MONTHLY]",
    "weekday": "[WEEKDAY if WEEKLY]",
    "firstRunAt": "[ISO_FIRST_RUN]"
  },
  "reportContext": {
    "datePreset": "[LAST_WEEK|LAST_MONTH]",
    "groupBy": "cos_service_name",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "[SCOPE_CEL]"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Tag coverage over time",
      "queries": [
        {
          "type": "cost",
          "name": "a",
          "alias": "Tagged",
          "filterCel": "[TAG_FIELD] != null",
          "chartType": "LINE"
        },
        {
          "type": "cost",
          "name": "b",
          "alias": "All in scope"
        },
        {
          "type": "formula",
          "name": "c",
          "alias": "Tag coverage",
          "formula": "a / b"
        }
      ],
      "aggBy": "[Week|Month]"
    },
    {
      "type": "TOP_FLOP",
      "title": "Coverage movers by service",
      "queries": [
        {
          "type": "cost",
          "name": "a",
          "alias": "Tagged",
          "filterCel": "[TAG_FIELD] != null"
        },
        {
          "type": "cost",
          "name": "b",
          "alias": "All in scope"
        },
        {
          "type": "formula",
          "name": "c",
          "alias": "Tag coverage",
          "formula": "a / b"
        }
      ],
      "aggBy": "Period",
      "topN": 10,
      "flopN": 10
    }
  ],
  "destinations": [{
    "destinationType": "[SLACK|TEAMS|EMAIL]",
    "channelId": "[DESTINATION_ID]"
  }]
}
```

Frozen: metric is formula `a / b` (not raw `cost`); `groupBy` = `cos_service_name`; scope on `reportContext.conditionsCel`; tagged-only predicate on query `filterCel`. Cadence usually MONTHLY (WEEKLY during an active tagging push).

## Confirm before build

1. **Which dimension defines coverage** — always ask; resolve CEL via `search`, then confirm `[TAG_FIELD]` (+ null vs specific value)
2. Scope exclusions (marketplace, shared, etc.) → `[SCOPE_CEL]`
3. Cadence + channel
4. Whether TOP_FLOP ranks by coverage ratio, untagged $, or absolute tagged $
5. Formula validated in `query` (both legs sane, ratio believable)

## Gotchas

- Measure a **ratio**, not "we spent $2M".
- Don't include structurally un-taggable spend in the denominator without calling it out — it permanently caps the ratio.
- Null labels: CEL `== null` / `!= null`, never `is_null` or `"null"`.

**Brief:** *"Coverage = [tag field] present / in-scope cost (excl. […]), by service: trend + top/flop, [cadence] to [channel] — formula validated in query first."*

**→ Hand off to `query` (validate formula) → `reports` (Schedule).**
