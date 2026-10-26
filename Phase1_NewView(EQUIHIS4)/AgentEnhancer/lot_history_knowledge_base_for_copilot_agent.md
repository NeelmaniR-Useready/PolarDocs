# Lot History Analysis: Comprehensive Knowledge Base for Copilot Agent

**Purpose:** This document is the definitive, comprehensive reference guide for interpreting Lot History data. It is explicitly designed to address and resolve the information gaps identified in `Lot_History_Copilot_Open_Information_Gaps.docx`. 

## 0. Open Information Gaps Resolution Mapping
This knowledge base directly addresses the open items required for reliable agent validation:
*   **Scrap analysis (Tests 8–10, 33):** Addressed in Section 1 (defines transaction codes, quantities, reason mappings, and validation logic).
*   **Genealogy and splits (Tests 28–30):** Addressed in Section 2 (defines split representation, parent/child linkages, HISTORDER sequencing, and event reconstruction).
*   **Cross-lot analysis (Tests 34–35):** Addressed in Section 3 (defines 'visited' vs. 'processed' based on operation state matrix).
*   **Atypical comments (Test 36):** Addressed in Section 4 (defines the 5-tier classification hierarchy, exact keyword rules, and statistical rarity metrics).
*   **Manual interactions (Tests 26–27, 39–40):** Addressed in Section 5 (defines manual vs. automated events, Move Status Control, old/new status fields, and calculation methodologies).
*   **Wafer callouts (Test 37):** While full wafer identification structures require further system validation, wafer-level references are observed in specific routine messages (e.g., `CheckSum Verified: Wafer <n>`) and manual comments.

---

## 1. Scrap Analysis and the MDS Command

This section dictates how wafer scrap and Material Disposition System (MDS) activity appear and should be processed.

### 1.1 Primary Transaction Indicators
*   **Primary Indicator:** `TRANS = ADHOCSCRAP` is the definitive transaction indicating wafer quantity was scrapped.
*   **Command Association:** `COMMAND = MDS` (Material Disposition System). In reviewed records, every `ADHOCSCRAP` transaction was accompanied by the `MDS` command. *Agent Validation Note:* A complete-file exception test should be run for `TRANS = ADHOCSCRAP` where `COMMAND <> MDS` to verify universal application.

### 1.2 Scrap Quantity and Reason-Code Format
The `HIST_REC` field contains the exact quantity and reason code, formatted as: `SCRAP: <Quantity>:<ReasonCode>`.
*   *Working Interpretations:*
    *   `SCRAP: 1:SC0066` → 1 wafer scrapped; reason code SC0066.
    *   `SCRAP: 10:SC9999` → 10 wafers scrapped; reason code SC9999.
    *   `SCRAP: 25:SC0039` → 25 wafers scrapped; reason code SC0039.
    *   `SCRAP: 8:GIVE` → 8 wafers scrapped; reason code GIVE.

### 1.3 Recommended Scrap-Analysis Logic
1.  **Filter:** Filter records to `TRANS = ADHOCSCRAP`.
2.  **Parse:** Parse `HIST_REC` into scrap quantity and reason code.
3.  **Retain Fields:** `LOT`, `DATE_TIME`, `OPER`, `OPERDESC`, `MACHINE`, `USERNAME`, `COMMAND`, `HIST_REC`, and source filename.
4.  **Validate:** Check whether `COMMAND = MDS`; report exceptions separately.
5.  **Contextualize:** Join nearby `COMMENT` rows (e.g., "MDS Particle Scrap") by lot and timestamp for narrative context. *Do not count `COMMENT` rows as independent scrap events.*
6.  **Deduplicate:** Only deduplicate when the identical scrap business event is repeated in history; preserve the original `ADHOCSCRAP` transaction. Do not assume a quantity of 25 always means full-lot scrap without confirming lot quantity.

### 1.4 Practical Field Reference for Scrap
| Field | Use in analysis |
| :--- | :--- |
| `LOT` | Lot associated with the history row. |
| `DATE_TIME` | Event timestamp. |
| `TRANS` | Transaction type; `ADHOCSCRAP` is the primary scrap indicator. |
| `OPER`/`OPERDESC` | Operation number and description at the time of the event. |
| `MACHINE` | Equipment associated with the event, when populated. |
| `USERNAME`/`EMPID` | User identity recorded for the event. |
| `HIST_REC` | Transaction details, quantity, reason code, or descriptive text. |
| `HISTCODE` | Compact history classification. |
| `COMMAND` | Command associated with the record (usually `MDS`). |
| `IS_DUPLICATE` | Indicator useful when filtering repeated or inherited scrap history. |

---

## 2. Lot Genealogy and Split Commands (SPLT)

This section details how to reconstruct complete split events, identifying parent/child relationships and cross-part transfers.

### 2.1 Core Finding: SPLT Events Span Multiple Rows
A split is **not** represented by a single `TRANS = SPLIT LOT` row. At a common split timestamp, the `SPLT` command writes multiple related rows. `HISTORDER` preserves the sequence and must be used for reconstruction.
*   **Event key:** `LOT` + `DATE_TIME` + `COMMAND = SPLT`
*   **Sequence key:** `HISTORDER`

### 2.2 Component Rows of a SPLT Event
| Component | What HIST_REC contains | Role |
| :--- | :--- | :--- |
| `SPLIT LOT` | `QTY: <qty> FROM:<source lot> TO:<target lot>` | Identifies quantity and source-to-target relationship. |
| `PART NUM` | `OLD PART:<part>` and `NEW PART:<part>` | Defines the part conversion. |
| `ROUTE` | `NEW ROUTE:<route> OLD:<route>` | Defines the route conversion when present. |
| Other `SPLT` rows | Split code or lot-attribute information | Additional transaction context. |

### 2.3 Interpretation Rules & Worked Example
*   **Child-lot split:** A suffix-only target (e.g., 6918A003 → 6918A003A) generally represents a split within the same base identifier.
*   **Cross-part transfer:** When the first four digits change, `SPLT` rows will define a destination part and sometimes route.
*   **Rule:** `PART NUM` and `ROUTE` values must be taken from rows sharing the exact `SPLT` event timestamp. Later `LRTE` commands are separate events.

**Worked Example: 5319A002 to 5294A002 (Cross-part transfer)**
*Quantity: 25 | Split timestamp: 10/04/2026 00:20:43*
*   `SPLIT LOT` | `SPLT` | `QTY: 25 FROM:5319A002 TO:5294A002`
*   `PART NUM` | `SPLT` | `OLD PART:C_PV106A1V15`
*   `PART NUM` | `SPLT` | `NEW PART:C_PV106A2V15`
*   `ROUTE` | `SPLT` | `NEW ROUTE:Q1L1-05A OLD:Q1L2-01A`

### 2.4 Why Lots Are Split (Observed Patterns)
| Observed pattern | Working purpose | Evidence and interpretation control |
| :--- | :--- | :--- |
| Suffix added to target | Conventional child-lot split | Confirmed genealogy pattern (qty moved). |
| `COMMAND` or history indicates `REWK` | Rework isolation | Confirmed `REWK`-associated pattern. |
| Small-qty split (e.g., 1 wafer) at metrology | Engineering/inspection | Observed pattern. Requires validation from comments. |
| First four digits change + `PART NUM`/`ROUTE` | Cross-part transfer | Confirmed for reviewed transfer examples. |
| Child lot becomes source of another `SPLT` | Multi-level genealogy | Confirmed pattern. Requires timestamp isolation. |

### 2.5 Recommended Output Schema for Split Analysis
| Output field | Source |
| :--- | :--- |
| Event timestamp | `DATE_TIME` from the SPLIT LOT/SPLT event |
| History sequence | `HISTORDER` from original export |
| Source / Target / Quantity | Parsed from `SPLIT LOT.HIST_REC` |
| Old part / New part | Parsed from `PART_NUM.HIST_REC` |
| Old route / New route | Parsed from `ROUTE.HIST_REC` |
| Operation | `OPER` and `OPERDESC` |
| Command | `COMMAND` (normally `SPLT`) |
| Duplicate status | `IS_DUPLICATE` |

---

## 3. Cross-Lot Analysis (Visited vs. Processed)

A strict distinction must be made between a lot being physically present at a location versus actively undergoing manufacturing processing.
*   **Visited:** A lot visited an operation when history contains credible evidence it was present, queued, dispatched, located at, or otherwise associated with that operation.
*   **Processed:** A lot was processed *only* when there is affirmative evidence that processing occurred (not merely present).

**State Evaluation Matrix:**

| State | Visited? | Processed? |
| :--- | :--- | :--- |
| Queue | Yes | No |
| Dispatch | Yes | No |
| Hold | Yes | No |
| Reposition | Yes | No |
| Move-In | Yes | Possibly |
| Running | Yes | Yes (in progress) |
| Move-Out | Yes | Yes |
| Scrap | Outcome | N/A |
| Split | Genealogy Event| N/A |

---

## 4. Manual Interactions

This defines how to measure intentional human intervention, distinguishing direct commands, adjustments, and comment-supported actions from routine system execution.

### 4.1 Confirmed Direct Manual Actions
| Category | Command | Manual function | Supporting evidence |
| :--- | :--- | :--- | :--- |
| Hold and status control | `HOLD` | Places/manages a lot hold | `COMMENT`; `ON HOLD CODE`; `OFF HOLD CODE` |
| Move Status Control | `RPOS` | Adjusts queue/dispatched status | `TRANS = REPOSITION`; `NEW=/OLD=` values |
| Operation Control | `LOPR` | Changes the lot operation | `HISTCODE = OP`; `HISTTRANS = ADJ OPER` |
| Route Control | `LRTE` | Changes the lot route | `ROUTE` records; possible `ADJ OPER` effect |
| Genealogy Management | `SPLT` | Splits material | `SPLIT LOT`; `PART NUM`; `ROUTE` |
| Genealogy Management | `LMRG` | Merges lots/quantities | `MERGE LOTS`; `MERGE QTY` |
| Rework Management | `REWK` | Moves lot through rework | Route-leave/split effects |
| Material Recovery | `RPUR` | Repurposes scrapped material | `CREATE LOT`; `REPURP LOT` |
| Material Disposition | `MDS` | Records disposition/scrap | `ADHOCSCRAP`; `COMMENT` rows |

### 4.2 Move Status Control, Operation Control, and Route Control
*   **Move Status (`RPOS`):** Adjusts execution state without changing the manufacturing operation.
    *   *Evidence:* `COMMAND = RPOS`, `TRANS = REPOSITION`.
    *   *Details:* Look for `NEW=D OLD=T` (Transition to Dispatched) or `NEW=Q OLD=D` (Dispatched to Queue).
*   **Operation Control (`LOPR`):** Changes assigned operation. Supported by `HISTCODE = OP` and `HISTTRANS = ADJ OPER`. Count `LOPR` + `ADJ OPER` as ONE event.
*   **Route Control (`LRTE`):** Changes assigned route. If a route change requires an operation adjustment, `ADJ OPER` is an effect of `LRTE` and should NOT be double-counted as a separate `LOPR` event.

### 4.3 Evidence Levels and Candidate Actions
*   **Level 1:** Confirmed manual command (`HOLD`, `RPOS`, `LOPR`, `LRTE`, `SPLT`, `LMRG`, `REWK`, `RPUR`, `MDS`).
*   **Level 2:** Detailed effect linked to confirmed command (`REPOSITION`, `ADJ OPER`).
*   **Level 3:** Explicit human-action comment linked by `LOT`+`DATE_TIME`+`COMMAND`+`HISTORDER`.
*   **Level 4:** Candidate requiring context.
*   **Level 5:** Documented reviewer judgment.
*   *Exclusion Rule:* Routine movement (`MVIN`, `MOUT`, `DISP`) are routine production flow events. The presence of a `USERNAME` alone does not make them manual interactions.

### 4.4 Percentage Methodology
*   **Manual-event rate:** (Distinct reconstructed confirmed manual events) / (All eligible reconstructed lot events)
*   **Move-status-control rate:** (Distinct `RPOS` events) / (All eligible lot events)
*   **Operation-adjustment rate:** (Distinct `LOPR` events, counted once) / (All eligible lot events)
*   **Route-control rate:** (Distinct `LRTE` events, including `ROUTE` and `ADJ OPER` effects) / (All eligible lot events)

**Recommended Reporting Language:**
> "A manual interaction was counted when the event was generated by a confirmed manual command (HOLD, RPOS, LOPR, LRTE, SPLT, LMRG, REWK, RPUR, or MDS), or when an explicit human-action comment was linked to the event. RPOS/REPOSITION and NEW=/OLD= movement-state records were classified as Move Status Control. LOPR with OP / ADJ OPER was classified as Operation Control. LRTE was classified as Route Control... DISP, MVIN, and MOUT were treated as routine flow unless additional evidence established manual intervention."

---

## 5. Atypical Comments Analysis

An atypical comment indicates an operational exception, disposition, genealogy change, process deviation, or unusual condition. Rarity alone does not dictate severity.

### 5.1 The 5-Tier Classification Hierarchy
1.  **Tier 1 (Explicit exception code/transaction):** Highest confidence. Automatically classify if the `COMMENT` shares the exact `LOT`, `DATE_TIME`, and `COMMAND` with an exception transaction (`ADHOCSCRAP`, `ON HOLD CODE`, `SPLIT LOT`, `REWK`, `REPOSITION`).
2.  **Tier 2 (Predefined keywords):** High confidence. Applied case-insensitively to normalized text.
    *   *Scrap:* scrap, scrapped, full-lot scrap, wafer loss
    *   *Hold:* hold, held, release hold, off hold
    *   *Rework:* rework, re-run, rerun, repeat, redo
    *   *Process/Equipment:* deviation, exception, out of spec, OOS, limits, alarm, equipment down, tool down, abort
    *   *Defect:* defect, particle, contamination, damage
    *   *Manual:* manual, override, reposition, correction
    *   *Investigation:* investigate, engineering review, disposition review
    *   *Failure:* cancel, failed, failure, error
3.  **Tier 3 (Temporal association):** Moderate/High confidence. Belongs to the immediate temporal window around a confirmed exception.
4.  **Tier 4 (Statistical rarity):** Moderate confidence. Text appearing < 5 times in population, or < 0.1% frequency. Must normalize first (e.g., `"Hold lot 6514A491 for Anthony"` → `"hold lot <LOT> for <NAME>"`).
5.  **Tier 5 (Reviewer judgment):** Variable. Must use reason codes: `ATYP-CONTEXT`, `ATYP-SEQUENCE`, `ATYP-GENEALOGY`, `ATYP-DATAQUALITY`, `NOT-ATYP-ROUTINE`.

### 5.2 Routine Comments and Exclusions
System-generated messages must not be classified as atypical solely based on rarity. Default to routine:
*   `MoveLots.Main.RepositionLot`
*   `MoveLots.Main.Cancel`
*   `CheckSum Verified: Wafer <n>`
*   `Located on Shelf ...`
*   `QTY: <n> Lot is Being Moved-In`

*Note on cross-lot comment inheritance: Do not copy a source-lot comment to every descendant. Preserve the original event; use `IS_DUPLICATE` to exclude inherited rows from aggregate event counts while retaining them for traceability.*