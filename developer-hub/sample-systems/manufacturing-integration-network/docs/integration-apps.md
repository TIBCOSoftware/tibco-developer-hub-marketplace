# Integration Apps

The seven TIBCO integration applications that move and transform messages. Each is a catalog
`Component` (`type: service`) that declares its contract edges (`providesApis` / `consumesApis`) and
its infrastructure edges (`dependsOn` the EMS broker, EMS destinations, and SAP backends).

---

## order-intake-bw6

- **Tech:** TIBCO BusinessWorks 6 (BW6) · **Team:** Order Management
- **What it does:** Receives inbound customer sales orders from SAP S/4HANA, validates each line
  against current material master data, and emits a production order to start manufacturing.
- **Consumes:** `sales-order-msg`, `material-master-msg`
- **Produces:** `production-order-msg`
- **Depends on:** SAP S/4HANA · EMS broker · `q.sales-order.inbound` · `t.production-order.created`

## production-orchestrator-bw6

- **Tech:** TIBCO BusinessWorks 6 (BW6) · **Team:** Production
- **What it does:** Orchestrates the production order through SAP MES (release, confirm operations)
  and publishes a goods-movement event for every stock posting (component issue, GR of finished goods).
- **Consumes:** `production-order-msg`
- **Produces:** `goods-movement-msg`
- **Depends on:** SAP MES · EMS broker · `t.production-order.created` · `t.goods-movement.posted`

## material-master-sync-bw6

- **Tech:** BW6 · **Team:** Master Data
- **What it does:** Keeps material master data consistent between SAP S/4HANA and SAP PLM and
  republishes the canonical material record for downstream consumers. The master-data publisher in
  the network.
- **Consumes:** —
- **Produces:** `material-master-msg`
- **Depends on:** SAP S/4HANA · SAP PLM · EMS broker · `q.material-master.sync`

## inventory-updater-flogo

- **Tech:** TIBCO Flogo · **Team:** Logistics
- **What it does:** Applies goods movements to stock in SAP EWM, recalculates availability against the
  reorder point, and publishes an inventory update. Joins goods-movement events with material master
  attributes.
- **Consumes:** `goods-movement-msg`, `material-master-msg`
- **Produces:** `inventory-update-msg`
- **Depends on:** SAP EWM · EMS broker · `t.goods-movement.posted` · `t.inventory.updated`

## quality-gateway-flogo

- **Tech:** Flogo · **Team:** Production
- **What it does:** Triggers and records quality inspections in SAP MES for relevant goods movements
  and publishes the inspection result (accept / reject / rework).
- **Consumes:** `goods-movement-msg`
- **Produces:** `quality-inspection-msg`
- **Depends on:** SAP MES · EMS broker · `t.goods-movement.posted` · `t.quality-inspection.result`

## shipment-dispatcher-bw6

- **Tech:** BW6 · **Team:** Logistics
- **What it does:** On inventory updates that make an order shippable, releases the outbound delivery
  in SAP EWM and emits a dispatch advice to carriers.
- **Consumes:** `inventory-update-msg`
- **Produces:** `shipment-dispatch-msg`
- **Depends on:** SAP EWM · EMS broker · `t.inventory.updated` · `q.shipment.dispatch`

## procurement-bridge-bw6

- **Tech:** BW6 · **Team:** Procurement
- **What it does:** Watches inventory updates and, when stock falls below the reorder point, raises a
  purchase order in SAP Ariba to replenish.
- **Consumes:** `inventory-update-msg`
- **Produces:** `purchase-order-msg`
- **Depends on:** SAP Ariba · EMS broker · `t.inventory.updated` · `q.purchase-order.outbound`
