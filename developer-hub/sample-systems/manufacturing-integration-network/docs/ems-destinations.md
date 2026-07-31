# EMS Destinations

All asynchronous exchange runs over a single **TIBCO EMS** broker. The broker and every destination
are modelled as `Resource` entities; each destination `dependsOn` the broker, and integration apps
`dependsOn` the destinations they use.

## manufacturing-ems-server (broker)

The TIBCO Enterprise Message Service server hosting all topics and queues. TLS enabled. Every
destination below lives on this broker.

## Queues (point-to-point)

Queues carry inbound-from-SAP or outbound-to-SAP/carrier traffic with a single logical consumer.

| Queue | Carries | Producer | Consumer |
|---|---|---|---|
| `q.sales-order.inbound` | `sales-order-msg` | SAP S/4HANA | order-intake-bw6 |
| `q.material-master.sync` | `material-master-msg` | material-master-sync-bw6 | (subscribers below) |
| `q.shipment.dispatch` | `shipment-dispatch-msg` | shipment-dispatcher-bw6 | external carriers |
| `q.purchase-order.outbound` | `purchase-order-msg` | procurement-bridge-bw6 | SAP Ariba |

## Topics (publish / subscribe)

Topics fan a single event out to multiple subscribers — this is where the cross-team dependencies
live (see [Architecture & Flows](architecture.md#cross-team-ripple-points)).

| Topic | Carries | Producer | Subscribers |
|---|---|---|---|
| `t.production-order.created` | `production-order-msg` | order-intake-bw6 | production-orchestrator-bw6 |
| `t.goods-movement.posted` | `goods-movement-msg` | production-orchestrator-bw6 | inventory-updater-flogo, quality-gateway-flogo |
| `t.inventory.updated` | `inventory-update-msg` | inventory-updater-flogo | shipment-dispatcher-bw6, procurement-bridge-bw6 |
| `t.quality-inspection.result` | `quality-inspection-msg` | quality-gateway-flogo | (reporting / downstream) |

## Naming convention

- `q.<area>.<direction>` for queues, `t.<area>.<event>` for topics.
- The catalog entity `name` is kebab-cased (`ems-q-sales-order-inbound`) because catalog names can't
  contain dots; the real EMS destination string is kept in the entity `title`
  (`q.sales-order.inbound`).
