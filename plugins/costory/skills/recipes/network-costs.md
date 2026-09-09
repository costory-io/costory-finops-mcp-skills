# Network costs (dashboard + families)

**When:** *"explain network costs"*, *"egress / data transfer"*, *"NAT bill"*, *"CloudFront vs origin"*, *"inter-region / inter-AZ"*, *"Direct Connect / Interconnect / ExpressRoute"*, *"Azure Bandwidth / Front Door"* — invoice networking across AWS, GCP, Azure.
**Audience:** FinOps / platform watching data transfer, NAT, CDN, and private connectivity.
**Outcome:** they leave on the **[Network] Network costs** dashboard — it teaches traffic vs rent, splits the eight families, and flags idle resources — plus any drill-down it does not cover.

**Not this if:** K8s namespace showback → `namespace-cost`. Whole-bill "what changed last month" → `explain-period-change`. Generic Explorer outside network → `query`.

## Tool sequence

1. `get_skill` with `skillId: "network-costs"`. If the catalog returns unknown skillId, retry `skillId: "plugins/costory/skills/network-costs/SKILL.md"`. **Stop and follow that skill**. Do not invent widgets or CEL from this card.
2. That skill owns dashboard discovery (search by **name**, never an id), teaching, and follow-up queries.

## Confirm before build

None on this card. The `network-costs` skill asks before a dashboard clone or VDIM publish.

## Gotchas

- `cos_subcategory in ["Network"]` is too wide — in one AWS+GCP org it returned **5x** the real number because it swallows Bedrock / Claude token SKUs. `cos_category_focus in ["Networking"]` is too narrow (drops EC2/NAT/GCS transfer).
- **The dashboard does the teaching** (traffic vs rent, families, levers). Open it and walk it; do not re-explain it in chat or rebuild it.
- Bytes are not on the invoice. For real throughput, cache hit ratio, or zombie LB / NAT / VPN hunting, use CloudWatch or Cloud Monitoring via `externalMetric` — the skill lists the metric names.
- AWS `AWSDataTransfer` billed $ is often a credit sink — template is `contracted_cost`.
- AWS SKU is often `_Data Transfer` — decode with `cos_line_item_usage_type`, not SKU contains. `aws_product_transfer_type` only if `search` finds it.
- Never put the dashboard id in a payload — `search` **[Network] Network costs** (often a template).
- Do not paste `[NET]` into every widget `filterCel` — inherit `conditionsCel` (long AND chains fail open).

**Brief:** *"Load skill `network-costs`: teach families and scopes, open [Network] Network costs, then drill egress / NAT / CDN / private path."*

**→ Hand off to `network-costs`.**
