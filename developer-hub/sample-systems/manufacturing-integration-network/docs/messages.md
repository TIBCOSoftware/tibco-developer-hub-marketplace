# Message Catalog

Every message is a catalog `API` (`type: ems-message`) whose definition holds the contract schema.
Producers `providesApis` it; consumers `consumesApis` it. Change a schema here and the consumers
listed become direct-impact entities.

> The same schemas are also visible on each API entity's **Definition** tab in the Developer Hub.

| Message | Destination | Format | Producer | Consumers |
|---|---|---|---|---|
| `sales-order-msg` | `q.sales-order.inbound` | XSD (ORDERS05) | SAP S/4HANA (external) | order-intake-bw6 |
| `material-master-msg` | `q.material-master.sync` | XSD (MATMAS05) | material-master-sync-bw6 | order-intake-bw6, inventory-updater-flogo |
| `production-order-msg` | `t.production-order.created` | JSON Schema | order-intake-bw6 | production-orchestrator-bw6 |
| `goods-movement-msg` | `t.goods-movement.posted` | JSON Schema | production-orchestrator-bw6 | inventory-updater-flogo, quality-gateway-flogo |
| `inventory-update-msg` | `t.inventory.updated` | JSON Schema | inventory-updater-flogo | shipment-dispatcher-bw6, procurement-bridge-bw6 |
| `quality-inspection-msg` | `t.quality-inspection.result` | JSON Schema | quality-gateway-flogo | — |
| `shipment-dispatch-msg` | `q.shipment.dispatch` | XSD (DESADV01) | shipment-dispatcher-bw6 | — (external carriers) |
| `purchase-order-msg` | `q.purchase-order.outbound` | XSD (PORDCR05) | procurement-bridge-bw6 | — (SAP Ariba) |

---

## sales-order-msg

**Purpose:** inbound customer sales order from SAP S/4HANA, kicking off the pipeline.
**Destination:** `q.sales-order.inbound` (queue) · **Format:** SAP IDoc **ORDERS05** (XSD).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="urn:manufacturer:ems:sales-order"
            xmlns="urn:manufacturer:ems:sales-order"
            elementFormDefault="qualified">
  <xsd:element name="SalesOrder">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="Header" type="HeaderType"/>
        <xsd:element name="Items" type="ItemsType"/>
      </xsd:sequence>
      <xsd:attribute name="idocType" type="xsd:string" fixed="ORDERS05"/>
    </xsd:complexType>
  </xsd:element>
  <xsd:complexType name="HeaderType">
    <xsd:sequence>
      <xsd:element name="OrderNumber" type="xsd:string"/>
      <xsd:element name="OrderType" type="xsd:string"/>
      <xsd:element name="SoldToParty" type="xsd:string"/>
      <xsd:element name="ShipToParty" type="xsd:string"/>
      <xsd:element name="PurchaseOrderRef" type="xsd:string" minOccurs="0"/>
      <xsd:element name="OrderDate" type="xsd:date"/>
      <xsd:element name="RequestedDeliveryDate" type="xsd:date" minOccurs="0"/>
      <xsd:element name="Currency" type="xsd:string"/>
      <xsd:element name="SalesOrg" type="xsd:string"/>
    </xsd:sequence>
  </xsd:complexType>
  <xsd:complexType name="ItemsType">
    <xsd:sequence>
      <xsd:element name="Item" maxOccurs="unbounded">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="ItemNumber" type="xsd:integer"/>
            <xsd:element name="MaterialNumber" type="xsd:string"/>
            <xsd:element name="Description" type="xsd:string" minOccurs="0"/>
            <xsd:element name="Quantity" type="xsd:decimal"/>
            <xsd:element name="Unit" type="xsd:string"/>
            <xsd:element name="NetPrice" type="xsd:decimal" minOccurs="0"/>
            <xsd:element name="Plant" type="xsd:string" minOccurs="0"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:sequence>
  </xsd:complexType>
</xsd:schema>
```

---

## material-master-msg

**Purpose:** material master record, synchronised between S/4HANA and PLM and shared with consumers.
**Destination:** `q.material-master.sync` (queue) · **Format:** SAP IDoc **MATMAS05** (XSD).
**Cross-team:** owned by Master Data; consumed by Order Management and Logistics.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="urn:manufacturer:ems:material-master"
            xmlns="urn:manufacturer:ems:material-master"
            elementFormDefault="qualified">
  <xsd:element name="MaterialMaster">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="MaterialNumber" type="xsd:string"/>
        <xsd:element name="MaterialType" type="xsd:string"/>
        <xsd:element name="Description" type="xsd:string"/>
        <xsd:element name="BaseUnit" type="xsd:string"/>
        <xsd:element name="MaterialGroup" type="xsd:string" minOccurs="0"/>
        <xsd:element name="GrossWeight" type="xsd:decimal" minOccurs="0"/>
        <xsd:element name="WeightUnit" type="xsd:string" minOccurs="0"/>
        <xsd:element name="Plants" type="PlantsType" minOccurs="0"/>
        <xsd:element name="ChangeIndicator" type="xsd:string"/>
      </xsd:sequence>
      <xsd:attribute name="idocType" type="xsd:string" fixed="MATMAS05"/>
    </xsd:complexType>
  </xsd:element>
  <xsd:complexType name="PlantsType">
    <xsd:sequence>
      <xsd:element name="Plant" maxOccurs="unbounded">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="PlantCode" type="xsd:string"/>
            <xsd:element name="ProcurementType" type="xsd:string" minOccurs="0"/>
            <xsd:element name="SafetyStock" type="xsd:decimal" minOccurs="0"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:sequence>
  </xsd:complexType>
</xsd:schema>
```

---

## production-order-msg

**Purpose:** a production order created from a sales order, to drive the shop floor.
**Destination:** `t.production-order.created` (topic) · **Format:** JSON Schema (Draft-07).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:manufacturer:ems:production-order",
  "title": "ProductionOrder",
  "type": "object",
  "additionalProperties": false,
  "required": ["productionOrderNumber", "materialNumber", "quantity", "unit", "plant", "scheduledStartDate"],
  "properties": {
    "productionOrderNumber": { "type": "string" },
    "salesOrderRef": { "type": "string" },
    "materialNumber": { "type": "string" },
    "quantity": { "type": "number", "minimum": 0 },
    "unit": { "type": "string", "examples": ["EA", "KG", "L"] },
    "plant": { "type": "string" },
    "workCenter": { "type": "string" },
    "scheduledStartDate": { "type": "string", "format": "date" },
    "scheduledEndDate": { "type": "string", "format": "date" },
    "priority": { "type": "string", "enum": ["LOW", "NORMAL", "HIGH", "URGENT"], "default": "NORMAL" },
    "components": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["materialNumber", "quantity", "unit"],
        "properties": {
          "materialNumber": { "type": "string" },
          "quantity": { "type": "number", "minimum": 0 },
          "unit": { "type": "string" }
        }
      }
    }
  }
}
```

---

## goods-movement-msg

**Purpose:** a stock movement posted during production (component issue / finished-goods receipt).
**Destination:** `t.goods-movement.posted` (topic) · **Format:** JSON Schema (Draft-07).
**Cross-team:** owned by Production; consumed by Logistics and Production.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:manufacturer:ems:goods-movement",
  "title": "GoodsMovement",
  "type": "object",
  "additionalProperties": false,
  "required": ["movementId", "movementType", "materialNumber", "quantity", "unit", "plant", "postingDate"],
  "properties": {
    "movementId": { "type": "string" },
    "movementType": { "type": "string", "enum": ["101", "261", "311", "601"] },
    "productionOrderRef": { "type": "string" },
    "materialNumber": { "type": "string" },
    "quantity": { "type": "number" },
    "unit": { "type": "string" },
    "plant": { "type": "string" },
    "storageLocation": { "type": "string" },
    "batch": { "type": "string" },
    "postingDate": { "type": "string", "format": "date" },
    "postedBy": { "type": "string" }
  }
}
```

---

## inventory-update-msg

**Purpose:** current available stock for a material at a plant; drives shipping and replenishment.
**Destination:** `t.inventory.updated` (topic) · **Format:** JSON Schema (Draft-07).
**Cross-team:** owned by Logistics; consumed by Logistics and Procurement.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:manufacturer:ems:inventory-update",
  "title": "InventoryUpdate",
  "type": "object",
  "additionalProperties": false,
  "required": ["materialNumber", "plant", "availableQuantity", "unit", "asOf"],
  "properties": {
    "materialNumber": { "type": "string" },
    "plant": { "type": "string" },
    "storageLocation": { "type": "string" },
    "availableQuantity": { "type": "number" },
    "reservedQuantity": { "type": "number", "default": 0 },
    "unit": { "type": "string" },
    "reorderPoint": { "type": "number" },
    "belowReorderPoint": { "type": "boolean" },
    "asOf": { "type": "string", "format": "date-time" }
  }
}
```

---

## quality-inspection-msg

**Purpose:** the outcome of a quality inspection lot.
**Destination:** `t.quality-inspection.result` (topic) · **Format:** JSON Schema (Draft-07).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:manufacturer:ems:quality-inspection",
  "title": "QualityInspectionResult",
  "type": "object",
  "additionalProperties": false,
  "required": ["inspectionLot", "materialNumber", "plant", "decision", "inspectionDate"],
  "properties": {
    "inspectionLot": { "type": "string" },
    "productionOrderRef": { "type": "string" },
    "materialNumber": { "type": "string" },
    "batch": { "type": "string" },
    "plant": { "type": "string" },
    "decision": { "type": "string", "enum": ["ACCEPTED", "REJECTED", "REWORK"] },
    "defectCount": { "type": "integer", "minimum": 0, "default": 0 },
    "inspector": { "type": "string" },
    "inspectionDate": { "type": "string", "format": "date" },
    "characteristics": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["name", "result"],
        "properties": {
          "name": { "type": "string" },
          "result": { "type": "string", "enum": ["PASS", "FAIL"] },
          "measuredValue": { "type": "number" },
          "unit": { "type": "string" }
        }
      }
    }
  }
}
```

---

## shipment-dispatch-msg

**Purpose:** outbound dispatch advice to carriers when a shipment leaves the warehouse.
**Destination:** `q.shipment.dispatch` (queue) · **Format:** SAP IDoc **DESADV01** (XSD).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="urn:manufacturer:ems:shipment-dispatch"
            xmlns="urn:manufacturer:ems:shipment-dispatch"
            elementFormDefault="qualified">
  <xsd:element name="ShipmentDispatch">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="ShipmentNumber" type="xsd:string"/>
        <xsd:element name="DeliveryNumber" type="xsd:string"/>
        <xsd:element name="Carrier" type="xsd:string"/>
        <xsd:element name="TrackingNumber" type="xsd:string" minOccurs="0"/>
        <xsd:element name="ShipFromPlant" type="xsd:string"/>
        <xsd:element name="ShipToAddress" type="AddressType"/>
        <xsd:element name="PlannedGoodsIssueDate" type="xsd:date"/>
        <xsd:element name="Packages" type="PackagesType"/>
      </xsd:sequence>
      <xsd:attribute name="idocType" type="xsd:string" fixed="DESADV01"/>
    </xsd:complexType>
  </xsd:element>
  <xsd:complexType name="AddressType">
    <xsd:sequence>
      <xsd:element name="Name" type="xsd:string"/>
      <xsd:element name="Street" type="xsd:string"/>
      <xsd:element name="City" type="xsd:string"/>
      <xsd:element name="PostalCode" type="xsd:string"/>
      <xsd:element name="Country" type="xsd:string"/>
    </xsd:sequence>
  </xsd:complexType>
  <xsd:complexType name="PackagesType">
    <xsd:sequence>
      <xsd:element name="Package" maxOccurs="unbounded">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="PackageId" type="xsd:string"/>
            <xsd:element name="MaterialNumber" type="xsd:string"/>
            <xsd:element name="Quantity" type="xsd:decimal"/>
            <xsd:element name="Weight" type="xsd:decimal" minOccurs="0"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:sequence>
  </xsd:complexType>
</xsd:schema>
```

---

## purchase-order-msg

**Purpose:** outbound purchase order to SAP Ariba when stock drops below the reorder point.
**Destination:** `q.purchase-order.outbound` (queue) · **Format:** SAP IDoc **PORDCR05** (XSD).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="urn:manufacturer:ems:purchase-order"
            xmlns="urn:manufacturer:ems:purchase-order"
            elementFormDefault="qualified">
  <xsd:element name="PurchaseOrder">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="PurchaseOrderNumber" type="xsd:string"/>
        <xsd:element name="DocumentType" type="xsd:string"/>
        <xsd:element name="Supplier" type="xsd:string"/>
        <xsd:element name="PurchasingOrg" type="xsd:string"/>
        <xsd:element name="PurchasingGroup" type="xsd:string" minOccurs="0"/>
        <xsd:element name="Currency" type="xsd:string"/>
        <xsd:element name="DocumentDate" type="xsd:date"/>
        <xsd:element name="Items" type="ItemsType"/>
      </xsd:sequence>
      <xsd:attribute name="idocType" type="xsd:string" fixed="PORDCR05"/>
    </xsd:complexType>
  </xsd:element>
  <xsd:complexType name="ItemsType">
    <xsd:sequence>
      <xsd:element name="Item" maxOccurs="unbounded">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="ItemNumber" type="xsd:integer"/>
            <xsd:element name="MaterialNumber" type="xsd:string"/>
            <xsd:element name="Quantity" type="xsd:decimal"/>
            <xsd:element name="Unit" type="xsd:string"/>
            <xsd:element name="NetPrice" type="xsd:decimal" minOccurs="0"/>
            <xsd:element name="Plant" type="xsd:string"/>
            <xsd:element name="DeliveryDate" type="xsd:date" minOccurs="0"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:sequence>
  </xsd:complexType>
</xsd:schema>
```
