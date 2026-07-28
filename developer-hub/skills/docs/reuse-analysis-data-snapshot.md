# Catalog data snapshot — reuse-or-build analyses (2026-07-22)

Shared provenance for the six `reuse-*.md` reports. All data fetched from the live Developer Hub
via the **dev-hub MCP server** (`POST /api/mcp-actions/v1/catalog`, JSON-RPC `tools/call`,
tool `catalog.query-catalog-entities`) — no REST fallback was needed.

## Queries run

1. `{"query":{"kind":"API"},"fields":["kind","metadata.name","metadata.description","spec.type","spec.owner","spec.lifecycle","spec.definition","relations"]}` → 16 API entities
2. `{"query":{"kind":{"$in":["Resource","Component"]}},"fields":[...same minus definition]}` → 46 entities

## Entities relied on (sap-integration-hub domain)

### Message contracts (kind API, `spec.type: ems-message`)

| API | Format | Transport | Owner | Producer | Consumers |
|---|---|---|---|---|---|
| `inventory-update-msg` | JSON Schema (closed) | topic `t.inventory.updated` | logistics-team | `inventory-updater-flogo` | `shipment-dispatcher-bw6`, `procurement-bridge-bw6` |
| `shipment-dispatch-msg` | XSD DESADV01 | queue `q.shipment.dispatch` | logistics-team | `shipment-dispatcher-bw6` | none internal (external carrier/3PL) |
| `material-master-msg` | XSD MATMAS05 | queue `q.material-master.sync` | master-data-team | `material-master-sync-bw6` | `inventory-updater-flogo`, `order-intake-bw6` |
| `quality-inspection-msg` | JSON Schema (closed) | topic `t.quality-inspection.result` | production-team | `quality-gateway-flogo` | none yet |
| `production-order-msg` | JSON Schema (closed) | topic `t.production-order.created` | manufacturing-order-management-team | `order-intake-bw6` | `production-orchestrator-bw6` |
| `goods-movement-msg` | JSON Schema (closed) | topic `t.goods-movement.posted` | production-team | `production-orchestrator-bw6` | `inventory-updater-flogo`, `quality-gateway-flogo` |

### Field evidence used by the reports

- `inventory-update-msg`: `materialNumber*`, `plant*`, `storageLocation`, `availableQuantity*`, `reservedQuantity`, `unit*`, `reorderPoint`, `belowReorderPoint`, `asOf*` (* = required; `additionalProperties: false`)
- `shipment-dispatch-msg` (XSD sequence): `ShipmentNumber`, `DeliveryNumber`, `Carrier`, `TrackingNumber` (minOccurs=0), `ShipFromPlant`, `ShipToAddress`, `PlannedGoodsIssueDate`, `Packages`; attr `idocType=DESADV01`
- `material-master-msg` (XSD sequence): `MaterialNumber`, `MaterialType`, `Description`, `BaseUnit`, `MaterialGroup?`, `GrossWeight?`, `WeightUnit?`, `Plants?` (PlantCode, ProcurementType?, SafetyStock?), `ChangeIndicator`; attr `idocType=MATMAS05`
- `quality-inspection-msg`: `inspectionLot*`, `productionOrderRef` (**optional**), `materialNumber*`, `batch`, `plant*`, `decision*` (ACCEPTED/REJECTED/REWORK), `defectCount`, `inspector`, `inspectionDate*`, `characteristics[]`
- `production-order-msg`: `productionOrderNumber*`, `salesOrderRef`, `materialNumber*`, `quantity*`, `unit*`, `plant*`, `workCenter`, `scheduledStartDate*`, `scheduledEndDate`, `priority`, `components[]`
- `goods-movement-msg`: `movementId*`, `movementType*` (101/261/311/601), `productionOrderRef`, `materialNumber*`, `quantity*`, `unit*`, `plant*`, `storageLocation`, `batch`, `postingDate*`, `postedBy`

### Supporting resources

- Topics (subscribable): `ems-t-inventory-updated`, `ems-t-quality-inspection-result`, `ems-t-production-order-created`, `ems-t-goods-movement-posted` — all owned by manufacturing-integration-platform-team
- Queues (point-to-point): `ems-q-shipment-dispatch`, `ems-q-material-master-sync`, `ems-q-sales-order-inbound`, `ems-q-purchase-order-outbound`
- Broker: `manufacturing-ems-server`
- Systems of record (no registered API contract): `sap-ewm` (logistics-team), `sap-s4hana`, `sap-plm` (master-data-team), `sap-mes` (production-team)

Also present in the catalog but out of scope (different domains): car-order-system entities (`car-*`), banking/payments entities (`pain001-*`, `pacs008-*`, `fraud-*`, `ledger-*`, `payment-*`).

No request/response (openapi) API exists in the manufacturing domain — the only `openapi` entities are in the car-order domain. This grounds the BUILD verdict in `reuse-stock-on-demand.md`.
