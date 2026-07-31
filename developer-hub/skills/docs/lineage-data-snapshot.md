# Data snapshot — lineage analyses over `sap-integration-hub`

Shared provenance for [lineage-index.md](./lineage-index.md) and the four reports it links.
Everything below was read from the **live catalog** on 2026-07-29; no value is inferred.

## Access route

Read directly from the **catalog REST API** of the running Developer Hub. No token needed — local
guest mode allows anonymous catalog reads.

```sh
curl -s "http://localhost:7007/api/catalog/entities?filter=spec.system=sap-integration-hub\
&fields=kind,metadata.name,metadata.description,metadata.tags,spec.type,spec.owner,\
spec.lifecycle,spec.definition,relations" -o hub.json          # 29 entities
```

Analysis performed with `.claude/skills/data-lineage/lineage.py`:

```sh
lineage.py graph  hub.json
lineage.py fields hub.json --flat
lineage.py trace  hub.json --field materialNumber
lineage.py trace  hub.json --field PlannedGoodsIssueDate
lineage.py flow   hub.json --api api:default/inventory-update-msg --field materialNumber
lineage.py flow   hub.json --api api:default/goods-movement-msg   --field quantity
lineage.py path   hub.json --from api:default/sales-order-msg --to resource:default/sap-ariba
```

## The directed flow graph

Derived from `providesApi` (writes) / `consumesApi` (reads) on each Component, with `dependsOn`
split into transport (topic/queue/broker) and systems of record.

| Component | Team | Reads | Writes | System of record |
|---|---|---|---|---|
| `material-master-sync-bw6` | master-data | — | `material-master-msg` | sap-s4hana, sap-plm |
| `order-intake-bw6` | order-management | `sales-order-msg`, `material-master-msg` | `production-order-msg` | sap-s4hana |
| `production-orchestrator-bw6` | production | `production-order-msg` | `goods-movement-msg` | sap-mes |
| `quality-gateway-flogo` | production | `goods-movement-msg` | `quality-inspection-msg` | sap-mes |
| `inventory-updater-flogo` | logistics | `goods-movement-msg`, `material-master-msg` | `inventory-update-msg` | sap-ewm |
| `shipment-dispatcher-bw6` | logistics | `inventory-update-msg` | `shipment-dispatch-msg` | sap-ewm |
| `procurement-bridge-bw6` | procurement | `inventory-update-msg` | `purchase-order-msg` | sap-ariba |

All seven components are `spec.lifecycle: production`.

## Contracts and their transport

| Contract | Format | Fields | Producer | Consumers | Transport | Type |
|---|---|---|---|---|---|---|
| `sales-order-msg` | XSD | 21 | **none (external)** | order-intake-bw6 | `q.sales-order.inbound` | queue |
| `material-master-msg` | XSD | 15 | material-master-sync-bw6 | order-intake-bw6, inventory-updater-flogo | `q.material-master.sync` | **queue** |
| `production-order-msg` | JSON Schema | 14 | order-intake-bw6 | production-orchestrator-bw6 | `t.production.order.created` | topic |
| `goods-movement-msg` | JSON Schema | 11 | production-orchestrator-bw6 | inventory-updater-flogo, quality-gateway-flogo | `t.goods.movement.posted` | topic |
| `inventory-update-msg` | JSON Schema | 9 | inventory-updater-flogo | procurement-bridge-bw6, shipment-dispatcher-bw6 | `t.inventory.updated` | topic |
| `quality-inspection-msg` | JSON Schema | 14 | quality-gateway-flogo | **none** | `t.quality.inspection.result` | topic |
| `purchase-order-msg` | XSD | 18 | procurement-bridge-bw6 | **none** | `q.purchase-order.outbound` | queue |
| `shipment-dispatch-msg` | XSD | 20 | shipment-dispatcher-bw6 | **none** | `q.shipment.dispatch` | queue |

All destinations `dependsOn` `resource:default/manufacturing-ems-server` (message-broker), owned by
`group:manufacturing-integration-platform-team`.

## Systems of record

| Resource | Type | Owner | Touched by |
|---|---|---|---|
| `sap-s4hana` | sap-system | master-data-team | material-master-sync-bw6, order-intake-bw6 |
| `sap-plm` | sap-system | master-data-team | material-master-sync-bw6 |
| `sap-mes` | sap-system | production-team | production-orchestrator-bw6, quality-gateway-flogo |
| `sap-ewm` | sap-system | logistics-team | inventory-updater-flogo, shipment-dispatcher-bw6 |
| `sap-ariba` | sap-system | procurement-team | procurement-bridge-bw6 |

## Shared-field inventory

Fields appearing in more than one contract, normalised (case- and separator-insensitive).
Raw spellings in brackets where they differ.

| Normalised field | Contracts | Spellings |
|---|---|---|
| `materialnumber` | **8 (all)** | `materialNumber`, `MaterialNumber`, `Items.MaterialNumber`, `Packages.MaterialNumber`, `components[].materialNumber` |
| `plant` | 7 | `plant`, `Items.Plant`, `Plants.Plant` |
| `unit` | 7 | `unit`, `Items.Unit`, `components[].unit`, `characteristics[].unit` |
| `quantity` | 6 | `quantity`, `Items.Quantity`, `Packages.Quantity`, `components[].quantity` |
| `idoctype` | 4 | `@idocType` (XSD attribute, fixed value) |
| `storagelocation` | 2 | `storageLocation` |
| `productionorderref` | 2 | `productionOrderRef` |
| `batch` | 2 | `batch` |
| `currency` | 2 | `Currency`, `Header.Currency` |
| `netprice` | 2 | `NetPrice`, `Items.NetPrice` |
| `itemnumber` | 2 | `Items.ItemNumber` |
| `description` | 2 | `Description`, `Items.Description` |

## Full field lists

### `inventory-update-msg` — JSON Schema, `additionalProperties: false`

`materialNumber`* `plant`* `storageLocation` `availableQuantity`* `reservedQuantity` (default 0)
`unit`* `reorderPoint` `belowReorderPoint` `asOf`* (date-time) · `$id: urn:manufacturer:ems:inventory-update`

### `goods-movement-msg` — JSON Schema

`movementId`* `movementType`* `productionOrderRef` `materialNumber`* `quantity`* `unit`* `plant`*
`storageLocation` `batch` `postingDate`* `postedBy`

### `production-order-msg` — JSON Schema

`productionOrderNumber`* `salesOrderRef` `materialNumber`* `quantity`* `unit`* `plant`* `workCenter`
`scheduledStartDate`* `scheduledEndDate` `priority` `components[]`{`materialNumber`* `quantity`* `unit`*}

### `quality-inspection-msg` — JSON Schema

`inspectionLot`* `productionOrderRef` `materialNumber`* `batch` `plant`* `decision`* `defectCount`
`inspector` `inspectionDate`* `characteristics[]`{`name`* `result`* `measuredValue` `unit`}

### `shipment-dispatch-msg` — XSD (`DESADV01`)

`ShipmentNumber`* `DeliveryNumber`* `Carrier`* `TrackingNumber` `ShipFromPlant`*
`ShipToAddress`{`Name`* `Street`* `City`* `PostalCode`* `Country`*} `PlannedGoodsIssueDate`* (date)
`Packages`{…`MaterialNumber`* `Quantity`* …} · `@idocType` fixed `DESADV01`

### `material-master-msg` — XSD (`MATMAS`)

`MaterialNumber`* `MaterialType`* `Description`* `BaseUnit`* `MaterialGroup` `GrossWeight`
`WeightUnit` `Plants`{`PlantCode`* `ProcurementType` `SafetyStock`} `ChangeIndicator`*

### `sales-order-msg` — XSD (`ORDERS`)

`Header`{`OrderNumber`* `OrderType`* `SoldToParty`* `ShipToParty`* `PurchaseOrderRef` `OrderDate`*
`RequestedDeliveryDate` `Currency`* `SalesOrg`*}
`Items`{`ItemNumber`* `MaterialNumber`* `Description` `Quantity`* `Unit`* `NetPrice` `Plant`}

### `purchase-order-msg` — XSD (`ORDERS` outbound)

`PurchaseOrderNumber`* `DocumentType`* `Supplier`* `PurchasingOrg`* `PurchasingGroup` `Currency`*
`DocumentDate`* `Items`{`ItemNumber`* `MaterialNumber`* `Quantity`* `Unit`* `NetPrice` `Plant`*
`DeliveryDate`*}

`*` = required.

## Verified reachability paths

```
sales-order-msg → order-intake-bw6 → production-order-msg → production-orchestrator-bw6
  → goods-movement-msg → inventory-updater-flogo → inventory-update-msg
  → procurement-bridge-bw6 → sap-ariba                                  (9 nodes)

sales-order-msg → order-intake-bw6 → production-order-msg → production-orchestrator-bw6
  → goods-movement-msg → inventory-updater-flogo → sap-ewm              (7 nodes)
```

## Limits of this snapshot

- **Contract level only.** No component source was read; no `backstage.io/source-location`
  annotation was present on any of the seven components. Every 🟡 Derived and 🔵 Originates
  classification in the reports is inference from field presence, not a verified mapping.
- **Field matching is name-based.** Semantic equivalence is asserted by the analyst, not proven —
  see the `quantity` grain change documented in [lineage-quantity.md](./lineage-quantity.md).
- **Unregistered consumers are invisible.** Three contracts show zero consumers; topics in
  particular can have subscribers that were never added to the catalog.
