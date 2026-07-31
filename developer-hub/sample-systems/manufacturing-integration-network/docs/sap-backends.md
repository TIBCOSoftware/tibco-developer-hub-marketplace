# SAP Backends

The five SAP systems are the systems of record. They are modelled as `Resource` entities
(`type: sap-system`) and are **external** to the integration system — integration apps reach them via
`dependsOn`. None of them talk to each other directly; all cross-system exchange goes through EMS.

| SAP system | Role | Used by (apps) | Related messages |
|---|---|---|---|
| **SAP S/4HANA** | ERP core — sales orders, material master, finance | order-intake-bw6, material-master-sync-bw6 | `sales-order-msg`, `material-master-msg` |
| **SAP MES** | Manufacturing Execution — shop-floor production & quality | production-orchestrator-bw6, quality-gateway-flogo | `goods-movement-msg`, `quality-inspection-msg` |
| **SAP EWM** | Extended Warehouse Management — inventory & shipping | inventory-updater-flogo, shipment-dispatcher-bw6 | `inventory-update-msg`, `shipment-dispatch-msg` |
| **SAP Ariba** | Procurement / supplier network | procurement-bridge-bw6 | `purchase-order-msg` |
| **SAP PLM** | Product Lifecycle Management — engineering/material data | material-master-sync-bw6 | `material-master-msg` |

## Details

### SAP S/4HANA (ERP)
The origin of demand. Sales orders flow out as IDoc **ORDERS05** onto `q.sales-order.inbound`. Also
the master record for material data, mirrored by `material-master-sync-bw6`.

### SAP MES (Manufacturing Execution)
Runs the shop floor. Receives production orders (via `production-orchestrator-bw6`), confirms
operations and stock movements, and is the system inspections are recorded in.

### SAP EWM (Extended Warehouse Management)
Owns physical stock and outbound logistics. Goods movements update EWM; shippable orders trigger
outbound deliveries and dispatch advices.

### SAP Ariba (Procurement)
Supplier-facing procurement. Receives replenishment purchase orders (IDoc **PORDCR05**) when
inventory falls below the reorder point.

### SAP PLM (Product Lifecycle Management)
Engineering source for product/material definitions. Paired with S/4HANA by the master-data sync to
keep a single canonical material record across the landscape.
