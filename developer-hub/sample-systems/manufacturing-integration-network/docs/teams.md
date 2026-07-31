# Teams & Ownership

Six `Group` entities own the apps, destinations and contracts in the network. Ownership is what the
impact-analysis coordination/notify list is built from — when a shared message changes, the owning
teams of all impacted consumers must be involved.

| Team | Owns | Key contracts |
|---|---|---|
| **Integration Platform** | TIBCO EMS broker + all destinations | (infrastructure) |
| **Order Management** | order-intake-bw6 | `sales-order-msg`, `production-order-msg` |
| **Production** | production-orchestrator-bw6, quality-gateway-flogo | `goods-movement-msg`, `quality-inspection-msg` |
| **Logistics** | inventory-updater-flogo, shipment-dispatcher-bw6 | `inventory-update-msg`, `shipment-dispatch-msg` |
| **Procurement** | procurement-bridge-bw6 | `purchase-order-msg` |
| **Master Data** | material-master-sync-bw6, SAP S/4HANA & PLM resources | `material-master-msg` |

## Coordination boundaries

The contracts that cross a team boundary are the ones that need change review beyond a single team:

- **`material-master-msg`** (Master Data) is consumed by **Order Management** and **Logistics** — any
  change to the material record affects order validation and inventory.
- **`goods-movement-msg`** (Production) is consumed by **Logistics** (inventory) and stays within
  Production (quality) — a production-side change ripples into logistics.
- **`inventory-update-msg`** (Logistics) is consumed by **Procurement** as well as Logistics — an
  inventory-shape change can break automatic replenishment.

When changing any of the above, treat the owning teams of every `apiConsumedBy` component as required
reviewers. The impact-analysis workflow generates this notify list automatically from the catalog.
