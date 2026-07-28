# Reuse-or-Build analyses — index (2026-07-22)

Six `/reuse-or-build` decision reports against the live `sap-integration-hub` catalog.

| # | Question | Verdict | Report |
|---|---|---|---|
| 1 | Notify me when stock falls below its reorder point | ✅ **REUSE** — subscribe to `inventory-update-msg` (`belowReorderPoint` already exists) | [reuse-below-reorder-alert.md](./reuse-below-reorder-alert.md) |
| 2 | Carrier + tracking number per outbound delivery | 🟡 **EXTEND delivery** — fields exist in `shipment-dispatch-msg`; queue transport needs a topic bridge | [reuse-carrier-tracking.md](./reuse-carrier-tracking.md) |
| 3 | Reserved vs available stock, on demand | 🔴 **BUILD** — thin query service fed by the existing topic (no request/response API exists) | [reuse-stock-on-demand.md](./reuse-stock-on-demand.md) |
| 4 | Planned goods issue date for a dashboard | 🟡 **EXTEND delivery** — same bridge as #2; make the consumer schema-tolerant for the pending `PlannedGoodsDeliveryDate` | [reuse-planned-gi-date.md](./reuse-planned-gi-date.md) |
| 5 | Material descriptions + base units for a pricing app | 🟡 **REUSE contract** — MATMAS05 matches fully; request a per-consumer delivery from master-data-team | [reuse-material-master.md](./reuse-material-master.md) |
| 6 | Quality results correlated to production orders | ✅ **REUSE** — join two existing topics on `productionOrderRef`; watch the optional-key gap | [reuse-quality-per-order.md](./reuse-quality-per-order.md) |

**Patterns that emerged across all six:**

- **Topic vs queue decides reuse cost.** Every topic-carried contract (#1, #6) was a plain ✅; every queue-carried one (#2, #4, #5) had a perfect field match but needed a delivery change — the platform team's bridge/topic ticket is the recurring unlock.
- **No manufacturing request/response API exists.** Any on-demand need (#3) is a build today; registering the new lookup contract in the catalog turns the *next* such question into a reuse.
- **#2 + #4 share one prerequisite** (`t.shipment.dispatched` bridge) — batch them into a single platform-team request.

Shared provenance: [reuse-analysis-data-snapshot.md](./reuse-analysis-data-snapshot.md)
