# Data Lineage — index & sample questions

Four worked examples of the `/data-lineage` skill, all generated from one run against the live
`sap-integration-hub` catalog on 2026-07-29. Shared provenance:
[lineage-data-snapshot.md](./lineage-data-snapshot.md).

## The reports

| # | Question asked | Subject | Verdict in one line | Report |
|---|---|---|---|---|
| 1 | *Where does `materialNumber` come from and which systems end up seeing it?* | field | Enters from S/4HANA by two doors, crosses 5 teams, leaves into Ariba **and** EWM — under **5 different spellings** | [lineage-materialnumber.md](./lineage-materialnumber.md) |
| 2 | *Trace the full flow of the inventory update message, upstream and downstream* | message | 4 hops from origin, 2 from sink; a **decision boundary** — only 2 of 9 fields travel further | [lineage-inventory-update-msg.md](./lineage-inventory-update-msg.md) |
| 3 | *Does the quantity on a sales order survive to the purchase order?* | field | **No.** It changes meaning 4 times; the delta→level aggregation is invisible to the catalog | [lineage-quantity.md](./lineage-quantity.md) |
| 4 | *Where does `PlannedGoodsIssueDate` on the dispatch IDoc come from?* | field | **Nowhere traceable.** Required field, zero upstream carriers, single team, one hop | [lineage-planned-gi-date.md](./lineage-planned-gi-date.md) |

## Sample questions you can ask

The skill triggers on provenance/flow phrasing. Any of these work:

**Field lineage** — the audit and governance workhorse
- "Where does `materialNumber` come from and where does it end up?"
- "Trace the `quantity` field through the system."
- "Which systems of record ever see the customer's requested delivery date?"
- "Does data from SAP S/4HANA reach SAP Ariba? Show me the path."
- "Where does `PlannedGoodsIssueDate` on the shipment IDoc originate?"
- "Which fields on `inventory-update-msg` have no upstream source?"

**Message lineage** — the architecture view
- "Show me the end-to-end lineage of `inventory-update-msg`."
- "What is upstream of `shipment-dispatch-msg`?"
- "How does a sales order become a purchase order?"
- "Which apps and EMS destinations does a goods movement pass through?"
- "Give me the data flow for the whole `sap-integration-hub` system."

**Governance / audit framing**
- "Which teams handle material data on its way to the supplier network?"
- "Where in the flow does data cross a team boundary?"
- "Which transformations can't be verified from the catalog?"
- "Is any field broadcast on a topic where consumers aren't registered?"

**Scoping variants** — all understood
- "…upstream only" / "…downstream only" / "just the provenance"
- "…and include the team hand-offs"
- "…write it to `reports/my-lineage.md`"

## Cross-cutting patterns found

These recur across all four reports and are the general lessons about this landscape.

**1. Naming convention flips at every JSON↔XSD boundary.** Four contracts are JSON Schema (native
events), four are XSD (SAP IDocs). Every crossing is a hand-written mapping in a BW6 or Flogo
process. `materialNumber` alone has 5 spellings. Nothing in the catalog declares the equivalences.

**2. Field names lie about semantics.** `quantity` is a delta in `goods-movement-msg` and a level in
`inventory-update-msg`; `Items.Quantity` means *ordered* on a sales order and *to buy* on a purchase
order. Any purely name-based lineage tool — including the skill's own `lineage.py` — will call these
carries. Reading the definitions is not optional.

**3. The most business-critical values are the least traceable.** `availableQuantity`,
`belowReorderPoint`, and `PlannedGoodsIssueDate` all originate inside an app. The catalog proves
they have no message-borne source and stops there. **Contract-level lineage is honest about its
own limits — that boundary is a finding, not a gap in the report.**

**4. Topics widen exposure invisibly.** Four of the eight contracts ride topics. A new subscriber
needs no producer change and no catalog entry, so any consumer list is a lower bound. Three
contracts show **zero** registered consumers — almost certainly incomplete catalog data rather than
dead destinations.

**5. Lineage and impact analysis agree where they overlap.** The single-team, single-hop finding for
`PlannedGoodsIssueDate` matches the blast-radius conclusion in
an `/impact-analysis` run on `shipment-dispatch-msg`, reached independently from the change-risk
side.

## Related skills

| Skill | Question | Example output |
|---|---|---|
| `/reuse-or-build` | Where can I **get** this data? | [reuse-analysis-index.md](./reuse-analysis-index.md) |
| `/impact-analysis` | What **breaks** if I change this? | [car-information-api-impact-analysis.md](./car-information-api-impact-analysis.md) |
| `/data-lineage` | Where does this data **come from / go to**? | this index |
