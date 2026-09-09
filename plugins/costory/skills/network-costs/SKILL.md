---
name: network-costs
description: "Use when the user asks about network costs, egress, data transfer, NAT, CloudFront, Cloud CDN, Azure Front Door, load balancers, inter-region or inter-AZ traffic, Direct Connect, Cloud Interconnect, ExpressRoute, Bandwidth, idle public IPs, or unused network resources. Call get_skill with skillId \"network-costs\" before investigating network spend or cloning the [Network] dashboard. Not for Kubernetes namespace showback (recipes namespace-cost) or generic Explorer (query)."
---

# Network costs

**Open [Network] Network costs, walk it, do not rebuild or re-explain it.** The dashboard teaches traffic vs rent, the eight families, credits, idle resources, and what moved. This skill is what is *not* on screen: how to find it, why the scope is SKU-explicit, landmines when cloning, and how to pair Costory cost with telemetry.


## Workflow

1. `get_context` — currency, providers, `externalMetricIntegrations`.
2. `search` `{ type: ["dashboards"], query: "Network" }` → **[Network] Network costs**. Org copies differ. **Never hardcode an id.**
3. `get` it. URL, scope (`conditionsCel`), and every category `filterCel` come from that payload — **never retype CELs from memory.**
4. Walk the dashboard. Query only for what it does not answer.
5. No `create_dashboard` unless they ask to clone. If the org has no network dashboard, build from `dashboards` + the template categories, then hand back the URL.

## What the LLM does not know (this is the real skill)

These are Costory-empirical, not textbook FinOps. That is the information advantage. It is real, and it is why a naive agent will quote the wrong network total.

- `cos_subcategory in ["Network"]` is too wide — measured 5x in an AWS+GCP org because it swallows Bedrock/Claude token SKUs and Pub/Sub. An LLM will use that filter first. **Do not.**
- `cos_category_focus in ["Networking"]` is too narrow — drops EC2-attached egress, NAT, inter-AZ, GCS/PD, Cloud Run.
- Use `contracted_cost` — billed `cost` under-reported one org’s AWS network by **$614k / 3 months** because credits land on transfer lines.
- `aws_product_transfer_type` hard-errors unless ingested. Template omits it.
- AWS grain is `cos_line_item_usage_type`, SKU is `_Data Transfer` — never `groupBy: cos_sku`.
- Open **[Network] Network costs** by name, copy CELs from `get`, never invent them or hardcode an id. Do not paste `[NET]` into every widget (`conditionsCel` AND-chains fail open).

Customers do not want the methodology argument. You need it when someone asks "why isn't this just the Network category?" The dashboard uses an explicit SKU + service + usage-type scope — copy it from `get`.

## Landmines when extending or cloning

- **`aws_product_transfer_type` hard-errors** (`"is not a valid dimension"`) unless ingested. Add per-org only after `search` confirms it.
- Dashboard scope lives on `conditionsCel`. Inherit it; do not AND a copied `[NET]` (or any full network CEL) into every widget `filterCel` — long AND chains fail open.
- Do not report Cloud Armor / Network Intelligence Center as traffic.

## Telemetry via Costory

The invoice is dollars, not bytes. Bytes, cache hit ratio, and attached-but-idle resources come from CloudWatch or Cloud Monitoring.

Confirm an integration in `get_context.externalMetricIntegrations`. Then `list_metrics` `{ includeExternal: true, search: "<term>" }` → `query` `{ type: "externalMetric", provider: "cloudwatch" | "cloudmonitoring", integrationId, metricName, aggregator, groupByFields }`. Pair with a cost series + `formula`. If none is connected, **say so and suggest the user to connect CloudWatch or Cloud Monitoring and stay on the dashboard** — Idle resources is still a real invoice-side answer. Do not invent metric values or a $/GB you cannot compute.

NAT bill vs real throughput (take the NAT `filterCel` from `get`):

```json
{
  "queries": [
    { "type": "cost", "name": "a", "metricId": "contracted_cost", "filterCel": "<NAT CEL from get>" },
    {
      "type": "externalMetric",
      "name": "b",
      "provider": "cloudwatch",
      "integrationId": "<from get_context>",
      "metricName": "AWS/NATGateway/BytesOutToDestination",
      "aggregator": "SUM",
      "groupByFields": ["NatGatewayId"]
    },
    { "type": "formula", "name": "c", "alias": "Cost per GB", "formula": "a / (b / 1e9)" }
  ],
  "datePreset": "LAST_3_MONTHS",
  "aggBy": "Month"
}
```

Same shape for GCP: `provider: "cloudmonitoring"`, e.g. `router.googleapis.com/nat/sent_bytes_count`. You already know the rest of the metric catalog (EC2 `NetworkOut`, ALB `RequestCount`, CloudFront `CacheHitRate` / `BytesDownloaded`, VPN `TunnelState`, DX `ConnectionBpsEgress`, GCP `instance/network/sent_bytes_count` + `remote_region`, `https/request_count` with `cache_result`). CloudFront `CacheHitRate` needs additional metrics enabled (billable).

Dashboard Idle resources = what the *invoice* can prove (unattached IPs, gateway uptime). Telemetry finds the rest: `RequestCount` ≈ 0, NAT bytes ≈ 0, `TunnelState` = 0, interconnect far below port capacity.

## Costory follow-up — dashboard does not answer

Prefer `datePreset: "LAST_3_MONTHS"`. Scope CEL from `get`, never memory.

| Ask | How |
|-----|-----|
| Which team / product owns this egress | `groupBy` a virtual dimension (`virtual_*` `bqName` from `list_virtual_dimensions`) inside the network scope |
| Network cost per K8s namespace | `groupBy: cos_namespace_reallocated` |
| Why did network jump last month | `find_cost_change_factors` with the network scope and `compare` |
| One account or all | `groupBy: cos_sub_account_id`, then `cos_region` inside the mover |
| Egress per customer / per request | cost + `externalMetric` + `formula` — needs an integration |

## Related

- `query` — slices. `dashboards` — clone. `virtual-dimensions` — team/product.
- `recipes` → `explain-period-change` (network-scoped DIGEST). `recipes` → `namespace-cost` (K8s showback, not invoice network).
