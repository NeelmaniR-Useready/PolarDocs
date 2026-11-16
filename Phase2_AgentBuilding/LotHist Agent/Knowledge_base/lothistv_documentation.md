# Lot History Business Description and Related Fields

## Executive Summary

| Field / Concept | Business meaning |
| :--- | :--- |
| **LOT** | The unit of work processed through the fab. A lot contains one or more wafers and is the primary WIP and genealogy object. |
| **OPER** | A specific process step on a route. It answers: where is the lot in the manufacturing flow? |
| **COMMAND / HISTCODE** | The transaction or system action performed against the lot. It answers: what action was taken? The specific history event or attribute recorded by a command, such as MOVE IN, MOVE OUT, ROUTE, COMMENT, or LOT COMPLETE. |
| **HISTORDER** | A field used in lot-history extracts to distinguish or order records that share the same transaction context. The exact database contract should be validated against the authoritative schema. |
| **MASK_LVL** | An operation property associated with the process step. Internal documentation confirms operations have a mask level. It is the masking level that operations are associated with. It can also be thought of as layer. |
| **OPERATION_DESC** | The human-readable operation description. `OPER` should be treated as the identifier; descriptions should be interpreted with route and historical context. |

---

## 1. LOT Lifecycle

### What a LOT Represents
In NetVision, a LOT is the unit of work processed through the fab. It contains one or more wafers, typically 25, and carries the information needed to manage current WIP position, route, operation, machine assignment, dispatch area, masks, priority, sales-order linkage, attributes, and manufacturing history.

### End-to-End Business Lifecycle

| Stage | Business activity | Typical records or outcomes |
| :--- | :--- | :--- |
| **1. Define product flow** | A part is assigned to a route. The route contains an ordered sequence of runcards and operations. | Approved route and operation sequence. |
| **2. Create the lot** | `LotCreate` establishes the lot with part, route, owner, wafer, mask, history, and initial WIP details. For applicable production lots, raw wafer inventory is decremented in Great Plains. | `LTCR` command; `CR`, `PT`, `RT`, `WF` and related history entries. |
| **3. Initial dispatch** | After creation, eligible non-test-wafer lots are automatically dispatched to the first operation when required route-operation-machine and recipe information is available. | Initial WIP and dispatch state. |
| **4. Process through route** | The lot moves through operations using move-in and move-out activity. NetVision validates resources and captures genealogy, EDC, SPC, comments, status, and location history. | `MVIN`/`MOUT` and movement-related history. |
| **5. Handle exceptions** | The lot may be held, released, reworked, split, merged, repositioned, rerouted, or adjusted under business rules. | `HOLD`, `REWK`, `SPLT`, `LMRG`, `LRTE`, `LOPR` and related history. |
| **6. Stage for transfer** | The completed fabrication lot reaches the TRANSFER operation or staging area and must satisfy transfer validation, including applicable inspection and quality requirements. | Transfer eligibility. |
| **7. Complete and transfer** | `LotTransfer` marks the lot completed, records completion data, removes it from active WIP, and may create a finished-goods inventory transaction in Great Plains. | `TRNF` command; `LC` history; closed-to-manufacturing state. |
| **8. Reporting and downstream use** | Completed lot information remains available for reporting, labels, quality, traceability, and genealogy. | Reporting record and retained history. |

---

## 2. Operations and Commands

### `OPER`: The Process Step
An operation is a specific process step that a lot goes through. At minimum, an operation has an operation number, description, mask level, and dispatch area. Operations are ordered within runcards and routes. A lot is placed at the first route operation when created and progresses through the ordered flow.

### `COMMAND`: The Transaction
A command identifies the transaction or applications action that caused the lot-history records to be written. One command can generate multiple `HISTCODE` rows because a single business transaction may record several effects or attributes.

| Command | Plain-language name | Role in lifecycle |
| :--- | :--- | :--- |
| **LTCR** | Lot Create | Creates a fabrication lot and its initial manufacturing records. |
| **MVIN** | Move In | Records entry into processing at the current operation or machine. |
| **MOUT** | Move Out | Records completion or exit from the current processing step and may advance the lot. |
| **DISP** | Dispatch | Assigns or requests a lot for processing. |
| **HOLD** | Hold | Changes or records lot hold status and comments. |
| **SPLT** | Lot Split | Creates a child lot or separates wafers from the original lot. |
| **LMRG** | Lot Merge | Combines eligible lots or quantities. |
| **REWK** | Rework | Routes a full or partial lot through an approved rework path. |
| **LOPR** | Change Lot Operation | Adjusts the lot to another operation under business rules. |
| **LRTE** | Change Lot Route | Changes the route and records associated operation/route effects. |
| **RPOS** | Reposition | Repositions or requests the lot within an operation. |
| **SETP / SETE** | Set Production / Engineering Flags | Updates production- or engineering-controlled lot attributes. |
| **RPUR** | Repurpose Lot | Recreates scrap material for approved engineering use. |
| **TRNF** | Transfer | Completes and closes the lot and performs the final manufacturing-to-inventory handoff when configured. |

> **Interpretation rule:** Use `OPER` to understand process position, `COMMAND` to understand the initiating transaction, and `HISTCODE` to understand each detailed effect recorded by that transaction.

---

## 3. History Codes and Order

### `HISTCODE`
`HISTCODE` identifies the specific event, attribute, or state change written to lot history. The command-to-history mapping shows that the same `HISTCODE` can appear under different commands and that one command commonly produces several history rows.

| Code | History label | Business description |
| :--- | :--- | :--- |
| **CR** | CREATE LOT | Creation of the lot. |
| **IN** | MOVE IN | Lot entered processing. |
| **MV** | MOVE OUT | Lot exited or completed a processing step. |
| **TX** | TRANSITION | Lot is transitioning into move-in. |
| **LA** | ARRIVAL | Lot arrived at an operation. |
| **DL** | DISPATCH | Lot was dispatched. |
| **SK / SA** | SKIP OPER / Auto Skip | Operation was skipped manually or automatically. |
| **RT** | ROUTE | Route associated with or changed for the lot. |
| **OP** | ADJ OPER | Operation adjustment. |
| **SP** | SPLIT LOT | Lot split transaction. |
| **MM / MQ** | MERGE LOTS / MERGE QTY | Lot merge and resulting quantity. |
| **RW** | REWORK | Lot rework. |
| **R1 / R2** | RTE LEAVE / RTE RETURN | Departure from and return to the main route. |
| **HC** | HOLD CODE | Lot hold status or reason. |
| **LS** | CHG STATUS | Lot status change. |
| **CM / CA** | COMMENT / Automation Comment | User or automated explanatory comment. |
| **PT** | PART NUM | Lot part number. |
| **MR** | MASK/REV | Mask number or revision associated with the lot. |
| **OW** | OWNERCODE | Lot owner code. |
| **DD** | DUE DATE | Lot due date. |
| **HT** | HOT STATUS | Hot-lot flag or priority status. |
| **SO** | SALES ORDR | Sales-order association. |
| **QI / QD / QE** | Quality Incident / Description / Event | Quality event and supporting description. |
| **SC** | ADHOCSCRAP | Ad hoc wafer scrap. |
| **UN** | UNSCRAP | Previously scrapped wafers were restored. |
| **LC** | LOTCOMPLET | Lot completed during transfer. |

### `HISTORDER`
Multiple lines were needed for the history entry. This is the order that they were entered.

---

## 4. Other Key Fields

### `MASK_LVL`
This is the Mask level or the layer. Masks are used to define objects/structures. Different layers build different parts of the components.

### `OPER` and `OPERATION_DESC`
`OPER` is the operation identifier or number of a step on a route. `OPERATION_DESC` is the display description and human-readable text associated with an operation in a route. Operation number and description are separate operation properties.

---

## 5. How to Read a Lot-History Row

| Field | Example | Interpretation |
| :--- | :--- | :--- |
| **LOT** | `6053A0K9` | The manufacturing lot whose history is being described. |
| **OPER** | `50325` | The process operation associated with the event. |
| **OPERATION DESC** | `LOW_TEMP` | The human-readable operation description captured in the extract. |
| **COMMAND** | `MOUT` | The transaction was a Move-Out. |
| **HISTCODE** | `MV` | The detailed history event was MOVE OUT. |
| **HISTORDER** | `1` | Order/differentiation within the transaction context; exact semantics require schema confirmation. |
| **MASK LVL** | `026` | Operation mask/process-layer value; exact encoding requires data-dictionary confirmation. |
| **COMMENTS** | `QTY: 25` | Additional business detail recorded with the event. |
