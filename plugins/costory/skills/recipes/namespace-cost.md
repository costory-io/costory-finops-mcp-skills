# Cost per namespace

**When:** *"cost per namespace"*, *"which K8s namespaces are most expensive?"*, *"track cluster spend per team's namespace"*, *"who's driving our EKS/GKE bill?"* — platform showback on shared clusters.
**Audience:** infra / platform / DevOps (weekly standup or infra review).
**Outcome:** weekly per-namespace cost with ~3 months of trend + last-week movers (up *and* down).

## Tool sequence

1. `get_context` → currency
2. Confirm weekday + channel type → `list_available_destinations`
3. Confirm `SCHEDULED` → `create_report`

## Payload skeleton

```json
{
  "visibility": "PRIVATE",
  "schedule": {
    "mode": "SCHEDULED",
    "period": "WEEKLY",
    "weekday": "[WEEKDAY]",
    "firstRunAt": "[ISO_FIRST_RUN]"
  },
  "reportContext": {
    "datePreset": "LAST_WEEK",
    "groupBy": "cos_namespace_reallocated",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "conditionsCel": "cos_namespace_reallocated != null"
  },
  "widgets": [
    {
      "type": "GRAPH_SNAPSHOT",
      "title": "Namespace cost — trailing weeks",
      "queries": [{
        "type": "cost",
        "name": "a",
        "alias": "Cost by namespace",
        "chartType": "LINE"
      }],
      "datePreset": "TRAILING_14_WEEKS",
      "aggBy": "Week"
    },
    {
      "type": "TOP_FLOP",
      "title": "Last week namespace movers",
      "queries": [{ "type": "cost", "name": "a", "alias": "Cost by namespace" }],
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

Frozen: `groupBy` = `cos_namespace_reallocated`; `conditionsCel` = `cos_namespace_reallocated != null`; GRAPH `TRAILING_14_WEEKS` / `aggBy: Week`; TOP_FLOP `topN`/`flopN` **10**; shared `LAST_WEEK`; cadence WEEKLY.

## Confirm before build

1. Scope = reallocated-only is OK (unallocated tracking is a separate ask)
2. Weekday for delivery
3. Channel type → destination

## Gotchas

- CEL null-exclusion is `!= null` — **not** `is_null`, **not** the string `"null"`.
- Prefer `cos_namespace_reallocated` over raw namespace labels when the org uses reallocation.
- Weekly beats monthly here — K8s workloads change fast.

**Brief:** *"Weekly namespace cost: 14-week trend + last-week top/flop 10, scoped to reallocated namespaces, every [weekday] to [channel]."*

**→ Hand off to `reports` (Schedule).**
