# AmazonEC2 +10% vs trailing 7 days (alert)

**When:** *"warn me if AmazonEC2 increases more than 10% vs the trailing 7 days"*, *"EC2 spike alert"*, *"notify if EC2 WoW jumps"* — standing cost alert, not a dashboard.
**Audience:** FinOps / platform on-call for compute spend spikes.
**Outcome:** an alert that fires when the current 7-day EC2 sum is **>10%** above the previous 7-day sum, delivered to a confirmed channel.

## Tool sequence

1. `get_context` → currency
2. Confirm scope: all AmazonEC2 vs further CEL (account / env) → `[FILTER_CEL]` (default below)
3. Confirm channel type → `list_available_destinations` → `[DESTINATION_ID]` / emails
4. Confirm dedup (default once per week) + threshold (default 10%)
5. `preview_alert` with skeleton → show `firingDays` / `notificationsCount`; tune if noisy
6. Explicit user confirm → `create_alert` — include returned URL in the reply

## Payload skeleton

```json
{
  "name": "AmazonEC2 +10% vs prior 7 days",
  "datePreset": "TRAILING_90_DAYS",
  "aggBy": "Day",
  "queries": [{
    "type": "cost",
    "name": "a",
    "alias": "AmazonEC2 cost",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "filterCel": "cos_service_name in [\"AmazonEC2\"]"
  }],
  "condition": "rollingSum(a, 7, DAY) > timeShift(rollingSum(a, 7, DAY), 7, DAY) * 1.1",
  "dedup": { "kind": "CALENDAR", "calendarUnit": "WEEK" },
  "notificationChannel": "[SLACK|TEAMS|EMAIL]",
  "slackChannelId": "[DESTINATION_ID if SLACK]",
  "teamsChannelId": "[DESTINATION_ID if TEAMS]",
  "emails": ["[EMAIL if EMAIL]"]
}
```

**`preview_alert` (same queries/condition/dedup, no channel):**

```json
{
  "queries": [{
    "type": "cost",
    "name": "a",
    "metricId": "cost",
    "currency": "[CURRENCY]",
    "filterCel": "cos_service_name in [\"AmazonEC2\"]"
  }],
  "condition": "rollingSum(a, 7, DAY) > timeShift(rollingSum(a, 7, DAY), 7, DAY) * 1.1",
  "dedup": { "kind": "CALENDAR", "calendarUnit": "WEEK" },
  "lookbackDays": 45
}
```

Frozen: service filter `cos_service_name in ["AmazonEC2"]` (extend CEL only if user scopes further); condition = **current 7-day rolling sum > prior 7-day rolling sum × 1.1**; `aggBy: "Day"`; lookback preset `TRAILING_90_DAYS` for create; preview `lookbackDays: 45`. Threshold `1.1` ↔ 10% — change only if the user asks (e.g. `1.2` for 20%).

## Confirm before build

1. All EC2 vs narrower scope (env / account)
2. Threshold still 10%?
3. Channel type → concrete destination
4. Dedup WEEK vs ROLLING N days
5. Preview fire count acceptable → then create

## Gotchas

- MCP is **create-only** for alerts — edits happen in the UI via the returned URL.
- Condition math runs daily in BigQuery; `datePreset` is the query lookback, **not** an evaluation cadence — use `dedup` for re-notification frequency.
- Guard interpretation: this is **7-day sum vs prior 7-day sum**, not "today vs 7-day average". If they meant the latter, switch condition to `a > (rollingSum(a, 7, DAY) / 7) * 1.1` and re-preview.

**Brief:** *"Alert: AmazonEC2 7-day sum > prior 7-day × 1.1, dedup [WEEK|…], to [channel] (previewed)."*

**→ Hand off:** no separate mechanics skill — execute with `preview_alert` → `create_alert` after confirm. Use `query` only to sanity-check EC2 baseline if preview looks wrong.
