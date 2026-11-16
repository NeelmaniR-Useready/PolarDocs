# Lot History Records: Scrap, Split, Comments, and Manual Interactions KB

---

## Part 1: Scrap Transactions and the MDS Command
*Findings from the October 2026 A14 lot-history export*

### Purpose and Scope
It focuses on how wafer scrap and MDS activity appear in lot-history records. It is intended as a practical analysis reference, not as a formal definition of LotTrack business rules.

**Evidence boundary:** Findings labeled “confirmed” were directly observed in the reviewed records. Business interpretations labeled “likely” or “requires validation” should be confirmed with Lot owners, process engineering, or the application’s reference tables.

### Key Findings at a Glance

| Record pattern | Working interpretation | Status |
| :--- | :--- | :--- |
| `TRANS = ADHOCSCRAP` | Primary indicator that wafer quantity was scrapped | Confirmed |
| `COMMAND = MDS` on reviewed `ADHOCSCRAP` rows | MDS was the command associated with the reviewed scrap transactions | Confirmed for reviewed records; complete-file exception test recommended |
| `HIST_REC = SCRAP: <Qty>:<Code>` | Contains scrap quantity and scrap reason code | Confirmed |
| Nearby `COMMENT` records | May provide narrative context for the scrap event | Confirmed; not the primary scrap indicator |

### Scrap Transactions

#### Primary transaction indicator
The most reliable observed indicator of wafer scrap is:
`TRANS = ADHOCSCRAP`

A text search for the word "scrap" may also return comments, e-mails, or other narrative history. Those records can add context, but should not be counted as scrap transactions unless an `ADHOCSCRAP` record supports the event.

#### Scrap quantity and reason-code format
For confirmed `ADHOCSCRAP` rows, `HIST_REC` followed this pattern:
`SCRAP: <Quantity>:<ReasonCode>`

| Observed HIST_REC | Working interpretation |
| :--- | :--- |
| `SCRAP: 1:SC0066` | 1 wafer scrapped; reason code SC0066 |
| `SCRAP: 10:SC9999` | 10 wafers scrapped; reason code SC9999 |
| `SCRAP: 25:SC0039` | 25 wafers scrapped; reason code SC0039 |
| `SCRAP: 8:GIVE` | 8 wafers scrapped; reason code GIVE |

#### Example raw records
```text
6514A4Q9,10/14/2026 04:04:06,1,ADHOCSCRAP,94966,080,PART_INK,080.PART_INK,KLA8935-302,HHAGOS,SCRAP: 1:SC0066,SP,MDS,Y,E,1,False,...
6514A4V6,10/09/2026 10:42:22,1,ADHOCSCRAP,79905,...,METR_ETCH,...,ZIZZOZ,SCRAP: 8:GIVE,SP,MDS,Y,E,1,False,...
6514A4Z4,10/21/2026 12:25:48,1,ADHOCSCRAP,50328,...,LOW_TEMP,...,MRL323.2,MOHUSSEIN,SCRAP: 10:SC9999,SP,MDS,Y,E,1,False,...
```
*Ellipses indicate columns not needed for this illustration. Analysts should retain the complete source row when producing audit evidence.*

### The MDS Command
Every `ADHOCSCRAP` transaction reviewed during this analysis showed:
`COMMAND = MDS`

Nearby `COMMENT` rows also included phrases such as “MDS Scrap” and “MDS Particle Scrap.” This supports a strong working association between MDS and recorded scrap activity in the reviewed history.

**Important limitation:** The reviewed examples do not by themselves prove that MDS is universally required for all scrap transactions. The definitive exception test is `TRANS = ADHOCSCRAP` and `COMMAND <> MDS` across the complete export.

### Recommended Scrap-Analysis Logic

| Step | Rule |
| :--- | :--- |
| 1 | Filter records to `TRANS = ADHOCSCRAP`. |
| 2 | Parse `HIST_REC` into scrap quantity and reason code. |
| 3 | Retain `LOT`, `DATE_TIME`, `OPER`, `OPERDESC`, `MACHINE`, `USERNAME`, `COMMAND`, `HIST_REC`, and source filename. |
| 4 | Validate whether `COMMAND = MDS`; report exceptions separately. |
| 5 | Join nearby `COMMENT` rows by lot and timestamp for narrative context. |
| 6 | Deduplicate only when the same scrap business event is repeated in history; preserve the original `ADHOCSCRAP` transaction for traceability. |

### Data-Quality and Interpretation Controls
* Do not count `COMMENT` rows as scrap transactions without a supporting `ADHOCSCRAP` record.
* Do not assume a quantity of 25 always means a full-lot scrap; confirm the lot quantity and surrounding history.
* Separate confirmed record content from operational interpretation, especially for scrap-code meanings and the role of MDS.
* Retain filename and source-row information when producing traceable analysis results.

### Practical Field Reference

| Field | Use in analysis |
| :--- | :--- |
| `LOT` | Lot associated with the history row |
| `DATE_TIME` | Event timestamp |
| `TRANS` | Transaction type; `ADHOCSCRAP` is the primary scrap indicator |
| `OPER` / `OPERDESC` | Operation number and description at the time of the event |
| `MACHINE` | Equipment associated with the event, when populated |
| `USERNAME` / `EMPID` | User identity recorded for the event |
| `HIST_REC` | Transaction details, quantity, reason code, or descriptive text |
| `HISTCODE` | Compact history classification |
| `COMMAND` | Command associated with the record; `MDS` appeared on every reviewed `ADHOCSCRAP` transaction |
| `IS_DUPLICATE` | Indicator useful when filtering repeated or inherited scrap history |

### Conclusion
The October lot-history review established a repeatable way to identify wafer scrap and distinguish it from related narrative comments. `ADHOCSCRAP` is the primary observed transaction indicator, and `HIST_REC` carries the scrap quantity and reason code. `MDS` was associated with every reviewed `ADHOCSCRAP` transaction; however, whether `MDS` is universally required remains subject to validation through a complete-file exception test for `ADHOCSCRAP` records where `COMMAND` is not `MDS`.

---

## Part 2: Lot History SPLT Command
*Split-event structure, HISTORDER sequencing, part changes, and route changes*

### Purpose and evidence base
This document records what was learned about Lot History split activity generated by the `SPLT` command. The focus is on cross-part transfers, where the first four digits of the lot number change, and on reconstructing the complete split event from multiple history rows.

### Core finding: one SPLT event spans multiple rows
A split is not represented by only the `TRANS = SPLIT LOT` row. At a common split timestamp, the `SPLT` command writes multiple related rows. `HISTORDER` preserves the order of those component records and should be used when reconstructing the transaction.

* **Event key:** `LOT + DATE_TIME + COMMAND = SPLT`
* **Sequence key:** `HISTORDER`
* **Primary components:** `SPLIT LOT`, `PART NUM`, `ROUTE`, and supporting attribute/comment rows

| Component | What HIST_REC contains | Role |
| :--- | :--- | :--- |
| `SPLIT LOT` | `QTY: <qty> FROM:<source lot> TO:<target lot>` | Identifies quantity and source-to-target relationship |
| `PART NUM` | `OLD PART:<part>` and `NEW PART:<part>` | Defines the part conversion |
| `ROUTE` | `NEW ROUTE:<route> OLD:<route>` | Defines the route conversion when present |
| Other `SPLT` rows | Split code or lot-attribute information | Additional transaction context |

### HISTORDER handling rule
* Group rows by destination `LOT`, the complete `DATE_TIME` value, and `COMMAND = SPLT`.
* Sort the grouped rows by numeric `HISTORDER` before interpreting the event.
* Treat the grouped rows as one business transaction, not independent events.
* Retain all rows even when `TRANS` differs, because `PART NUM` and `ROUTE` rows complete the meaning of `SPLIT LOT`.
* Use `IS_DUPLICATE` to distinguish inherited child-lot history from the originating transaction when the full export is available.

### Worked example: 5319A002 to 5294A002
The split transaction occurred at `10/04/2026 00:20:43`. The supporting extract shows the part and route records at `10/04/2026 00:20`. These rows belong to the same `SPLT` event and should be displayed in `HISTORDER` sequence from the original export.

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 25 FROM:5319A002 TO:5294A002` |
| `PART NUM` | `SPLT` | `OLD PART:C_PV106A1V15` |
| `PART NUM` | `SPLT` | `NEW PART:C_PV106A2V15` |
| `ROUTE` | `SPLT` | `NEW ROUTE:Q1L1-05A OLD:Q1L2-01A` |

**Subsequent route activity:** At `10/04/2026 00:28`, a separate `ROUTE` record with `COMMAND = LRTE` changed the route from `Q1L1-05A` back to `Q1L2-01A`. This later `LRTE` event must not be merged into the `00:20` `SPLT` transaction.

### Consolidated cross-part split results

| Source | Target | Qty | Old part | New part | Old route | New route |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 5319A002 | 5294A002 | 25 | C_PV106A1V15 | C_PV106A2V15 | Q1L2-01A | Q1L1-05A |
| 5319A003 | 5294A003 | 13 | C_PV106A1V15 | C_PV106A2V15 | Q1L2-01A | Q1L1-05A |
| 5319A004 | 5294A004 | 12 | C_PV106A1V15 | C_PV106A2V15 | Q1L2-01A | Q1L1-05A |
| 5866A002 | 5776A003 | 3 | C_MG6042AP | C_MG6062AP | Not present | Not present |
| 5866A003 | 5937A003 | 9 | C_MG6042AP | C_MG6058AP | Not present | Not present |
| 6558A002 | 6187A001 | 4 | C_MG6033AP | C_MG6078AP | Not present | Not present |
| 6587A003 | 7190A002 | 25 | C_MG6027BP | C_MG6057AP | SBH1-01B | SBH1-03B |
| 6587A004 | 7909A003 | 25 | C_MG6027BP | C_MG6059AP | SBH1-01B | SBH1-03B |

*“Not present” means no `TRANS = ROUTE` row was found for that destination lot in `TransferredLotsHistory.csv`. It does not establish that the route remained unchanged.*

### Supporting SPLT data by destination lot

#### 5319A002 $\rightarrow$ 5294A002
**Quantity:** 25 | **Split timestamp:** 10/04/2026 00:20:43

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 25 FROM:5319A002 TO:5294A002` |
| `PART NUM` | `SPLT` | `NEW PART:C_PV106A2V15` |
| `PART NUM` | `SPLT` | `OLD PART:C_PV106A1V15` |
| `ROUTE` | `SPLT` | `NEW ROUTE:Q1L1-05A OLD:Q1L2-01A` |

#### 5319A003 $\rightarrow$ 5294A003
**Quantity:** 13 | **Split timestamp:** 10/06/2026 17:32:41

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 13 FROM:5319A003 TO:5294A003` |
| `PART NUM` | `SPLT` | `NEW PART:C_PV106A2V15` |
| `PART NUM` | `SPLT` | `OLD PART:C_PV106A1V15` |
| `ROUTE` | `SPLT` | `NEW ROUTE:Q1L1-05A OLD:Q1L2-01A` |

#### 5319A004 $\rightarrow$ 5294A004
**Quantity:** 12 | **Split timestamp:** 10/21/2026 13:18:07

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 12 FROM:5319A004 TO:5294A004` |
| `PART NUM` | `SPLT` | `NEW PART:C_PV106A2V15` |
| `PART NUM` | `SPLT` | `OLD PART:C_PV106A1V15` |
| `ROUTE` | `SPLT` | `NEW ROUTE:Q1L1-05A OLD:Q1L2-01A` |

#### 5866A002 $\rightarrow$ 5776A003
**Quantity:** 3 | **Split timestamp:** 10/22/2026 14:03:50

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 3 FROM:5866A002 TO:5776A003` |
| `PART NUM` | `SPLT` | `NEW PART:C_MG6062AP` |
| `PART NUM` | `SPLT` | `OLD PART:C_MG6042AP` |

#### 5866A003 $\rightarrow$ 5937A003
**Quantity:** 9 | **Split timestamp:** 10/22/2026 14:07:58

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 9 FROM:5866A003 TO:5937A003` |
| `PART NUM` | `SPLT` | `OLD PART:C_MG6042AP` |
| `PART NUM` | `SPLT` | `NEW PART:C_MG6058AP` |

#### 6558A002 $\rightarrow$ 6187A001
**Quantity:** 4 | **Split timestamp:** 10/19/2026 12:30:18

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 4 FROM:6558A002 TO:6187A001` |
| `PART NUM` | `SPLT` | `OLD PART:C_MG6033AP` |
| `PART NUM` | `SPLT` | `NEW PART:C_MG6078AP` |

#### 6587A003 $\rightarrow$ 7190A002
**Quantity:** 25 | **Split timestamp:** 10/21/2026 12:08:04

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 25 FROM:6587A003 TO:7190A002` |
| `PART NUM` | `SPLT` | `OLD PART:C_MG6027BP` |
| `PART NUM` | `SPLT` | `NEW PART:C_MG6057AP` |
| `ROUTE` | `SPLT` | `NEW ROUTE:SBH1-03B OLD:SBH1-01B` |

#### 6587A004 $\rightarrow$ 7909A003
**Quantity:** 25 | **Split timestamp:** 10/21/2026 12:17:13

| TRANS | COMMAND | HIST_REC |
| :--- | :--- | :--- |
| `SPLIT LOT` | `SPLT` | `QTY: 25 FROM:6587A004 TO:7909A003` |
| `PART NUM` | `SPLT` | `NEW PART:C_MG6059AP` |
| `PART NUM` | `SPLT` | `OLD PART:C_MG6027BP` |
| `ROUTE` | `SPLT` | `NEW ROUTE:SBH1-03B OLD:SBH1-01B` |

### Interpretation rules
* A suffix-only target generally represents a child-lot split within the same base identifier.
* When the first four digits change, the observed `SPLT` rows also define a destination part and, in some cases, a destination route. This supports classifying the event as a cross-part transfer rather than only a child-lot suffix split.
* `PART NUM` and `ROUTE` values must be taken from the rows that share the `SPLT` event timestamp; later `LRTE` or other commands are separate events.
* A missing `ROUTE` row in the supplied extract should be reported as “not present in extract,” not interpreted as “no route change.”

### Recommended output schema for analysis

| Output field | Source |
| :--- | :--- |
| Event timestamp | `DATE_TIME` from the `SPLIT LOT`/`SPLT` event |
| History sequence | `HISTORDER` from original export |
| Source lot / target lot / quantity | Parsed from `SPLIT LOT.HIST_REC` |
| Old part / new part | Parsed from `PART NUM.HIST_REC` |
| Old route / new route | Parsed from `ROUTE.HIST_REC` |
| Operation | `OPER` and `OPERDESC` |
| Command | `COMMAND`, normally `SPLT` for event components |
| Duplicate status | `IS_DUPLICATE` |
| Evidence | Original filename and source row |

### Conclusion
The appropriate unit of analysis is the complete `SPLT` event, not the individual `SPLIT LOT` row. The event should be reconstructed by grouping the same lot, complete timestamp, and `SPLT` command, then ordering the component rows by `HISTORDER`. This approach captures quantity movement, source and target lots, part conversion, route conversion, and related split metadata without separating records that belong to the same transaction.

### Why lots are split at Polar
The reviewed Lot History records show that `SPLT` is used for more than one type of material movement. The categories below combine confirmed record patterns with working operational interpretations. The exact business reason for an individual split should be established from the complete `HISTORDER` sequence, supporting comments, part and route records, and the applicable process or engineering documentation.

| Observed pattern | Working purpose | Evidence and interpretation control |
| :--- | :--- | :--- |
| Source and target retain the same base lot identifier; target adds a suffix | Conventional child-lot split | Confirmed genealogy pattern. The records establish that a quantity moved to a child lot; the specific engineering or production reason requires supporting records. |
| `COMMAND` or supporting history indicates `REWK` | Rework isolation | Confirmed `REWK`-associated split pattern. This can separate identified material for rework while preserving genealogy; the precise rework cause is not defined by `SPLIT LOT` alone. |
| Small-quantity split, often one wafer, at inspection or metrology-related operations | Engineering, inspection, or characterization activity | Observed pattern only. The split quantity and operation are explicit, but the purpose must be validated from comments, hold notes, or engineering documentation. |
| First four digits change and `SPLT` rows include `PART NUM` and sometimes `ROUTE` changes | Cross-part transfer or product conversion | Confirmed for the reviewed transfer examples. `PART NUM` identifies old/new part values; `ROUTE` identifies old/new route values when present. |
| Child lot later becomes the source of another `SPLT` event | Multi-level material genealogy | Confirmed genealogy pattern. `HISTORDER` and timestamps are required to reconstruct each event without merging separate transactions. |

### Evidence-backed examples

| Category | Example | What the records explicitly establish |
| :--- | :--- | :--- |
| Child-lot genealogy | `6918A003` $\rightarrow$ `6918A003A` $\rightarrow$ `6918A003B` | Quantities were transferred through two sequential `SPLT` events. |
| Rework-associated split | `6515A0Y3` $\rightarrow$ `6515A0Y3A` | The split row used `REWK` in the command/history context. |
| Cross-part transfer | `5319A002` $\rightarrow$ `5294A002` | Quantity 25 moved; part changed from `C_PV106A1V15` to `C_PV106A2V15`; `SPLT` route changed from `Q1L2-01A` to `Q1L1-05A`. |
| Cross-part transfer | `6587A003` $\rightarrow$ `7190A002` | Quantity 25 moved; part changed from `C_MG6027BP` to `C_MG6057AP`; route changed from `SBH1-01B` to `SBH1-03B`. |
| Cross-part transfer | `5866A002` $\rightarrow$ `5776A003` | Quantity 3 moved; part changed from `C_MG6042AP` to `C_MG6062AP`; no `ROUTE` row was present in the supplied extract. |

### Interpretation principle
A `SPLIT LOT` row does not, by itself, explain why the lot was split. The business event is the complete `SPLT` command group: `SPLIT LOT`, `PART NUM`, `ROUTE`, and any supporting split-code, attribute, comment, or hold records in `HISTORDER` sequence. Analysts should report the explicit data first and label the operational purpose as confirmed only when the supporting records state it.

---

## Part 3: Atypical Comments in Lot History
*Definition, classification rules, review controls, and reporting guidance*

### Purpose
This document defines a repeatable method for identifying and reporting atypical comments in Lot History. The method is intended for cross-lot analysis, exception review, and investigation support. It separates objective automated indicators from reviewer judgment so that results are reproducible and defensible.

### Working definition
An atypical comment is a `COMMENT` record that indicates, supports, or is closely associated with an operational exception, material disposition, genealogy change, process deviation, manual intervention, or unusual condition. A comment may also be atypical because its normalized text is statistically rare compared with routine comments in the selected comparison population.

**Classification principle:** A comment is atypical when at least one defined rule is met. Reviewer judgment may supplement a rule-based result, but should not silently replace the rules.

### Classification hierarchy

| Tier | Classification basis | Confidence | Recommended use |
| :--- | :--- | :--- | :--- |
| 1 | Explicit exception code or exception-associated transaction | Highest | Automatically classify as atypical |
| 2 | Predefined exception keyword or phrase | High | Automatically classify, subject to exclusions and context |
| 3 | Temporal association with a confirmed exception event | High to moderate | Classify when the event linkage rule is satisfied |
| 4 | Statistical rarity within a declared comparison population | Moderate | Flag for review; rarity alone does not establish severity |
| 5 | Documented reviewer judgment | Variable | Use only with a reason code and reviewer note |

### Tier 1: exception codes and transactions
A comment should be classified as atypical when it is part of the same reconstructed event as a confirmed exception or disposition transaction. Link records using `LOT`, complete `DATE_TIME`, `COMMAND`, and `HISTORDER` where available.

| Associated record or indicator | Interpretation |
| :--- | :--- |
| `TRANS = ADHOCSCRAP` | Confirmed scrap activity |
| `ON HOLD CODE` / `OFF HOLD CODE` | Hold placement or release activity |
| `TRANS = SPLIT LOT` | Material genealogy or lot transfer activity |
| `TRANS = PART NUM` with `COMMAND = SPLT` | Part-number change within a split event |
| `TRANS = ROUTE` with `COMMAND = SPLT` | Route change within a split event |
| `REWK` context | Rework-associated activity |
| `REPOSITION` or manual override context | Manual movement correction or intervention; review required |

**Important:** The associated transaction establishes that the comment is relevant to an exception event. The comment text may explain the event, but the `COMMENT` row should not replace the underlying transaction as the authoritative evidence.

### Tier 2: keyword and phrase rules
Use a controlled keyword dictionary rather than an unrestricted text search. Apply the rules case-insensitively to normalized comment text, and preserve the matched phrase for auditability.

| Category | Suggested seed terms | Typical reason for review |
| :--- | :--- | :--- |
| Scrap / disposition | scrap, scrapped, full-lot scrap, wafer loss | Material loss or disposition |
| Hold / release | hold, held, release hold, off hold | Processing stopped or resumed |
| Rework / repeat processing | rework, re-run, rerun, repeat, redo | Material diverted for additional processing |
| Split / genealogy | split, child lot, transfer lot | Genealogy or identity changed |
| Process exception | deviation, exception, out of spec, OOS, limits | Process condition outside routine flow |
| Equipment / alarm | alarm, equipment down, tool down, abort | Equipment or execution interruption |
| Defect / contamination | defect, particle, contamination, damage | Material-quality concern |
| Manual intervention | manual, override, reposition, correction | Nonroutine operator or system action |
| Investigation / engineering | investigate, engineering review, disposition review | Further technical review required |
| Cancellation / failure | cancel, failed, failure, error | Attempted action did not complete normally |

#### Keyword controls
* Match whole words or approved phrases where possible. For example, “hold” should not match an unrelated longer word.
* Maintain category-specific exclusions for known routine system messages.
* Record the matched term, category, and dictionary version.
* Treat keywords as indicators, not proof of root cause or severity.
* Review abbreviations and local codes with LotTrack and process owners before adding them to production rules.

### Tier 3: event-association rule
A `COMMENT` row may be atypical even when the text does not contain an exception keyword. Classify the comment by association when it belongs to, or occurs immediately around, a confirmed exception event.

**Preferred event link:**
`LOT + exact DATE_TIME + COMMAND`
Then order component rows by `HISTORDER`

When exact event keys are unavailable, use a declared temporal window only as a secondary rule. The window must be chosen and documented for the analysis; it should not be silently assumed. Comments outside the window remain unlinked unless another explicit identifier connects them.

### Tier 4: statistical rarity
Rarity can identify comments that the rule dictionary does not recognize. Normalize comments before frequency analysis by applying consistent case, trimming whitespace, and optionally replacing transaction-specific identifiers such as lot numbers, wafer IDs, tool IDs, timestamps, and quantities with tokens.

**Example normalization:**
`"Hold lot 6514A491 for Anthony"` $\rightarrow$ `"hold lot <LOT> for <NAME>"`

Recommended rarity measures are listed below. Thresholds should be calibrated against a representative baseline rather than treated as universal Polar standards.

| Measure | Suggested starting rule | Caution |
| :--- | :--- | :--- |
| Exact normalized-text frequency | Flag text appearing fewer than 5 times in the comparison population | Large populations can still produce many one-off routine comments |
| Relative frequency | Flag text representing less than $0.1\%$ of comments | Threshold depends heavily on population size |
| Operation-specific rarity | Flag text rare within `OPER` or `OPERDESC` | Requires enough comments at the operation |
| User-specific or tool-specific rarity | Compare only when the analytical question requires it | Can create misleading results when sample sizes are small |

Rarity is not severity. A rare comment may be harmless, while a common phrase such as “MDS Scrap” may be operationally significant. Statistical rarity should normally produce a review flag rather than an automatic severity rating.

### Tier 5: reviewer judgment
Reviewer judgment is appropriate when context indicates a nonroutine condition that automated rules did not identify. Every judgment-based classification should include a standardized reason and a brief evidence note.

| Reviewer reason code | Use |
| :--- | :--- |
| `ATYP-CONTEXT` | Context suggests an exception despite neutral wording |
| `ATYP-SEQUENCE` | Comment is unusual because of its position in the event sequence |
| `ATYP-GENEALOGY` | Comment affects interpretation across source or descendant lots |
| `ATYP-DATAQUALITY` | Comment or related records indicate inconsistent or incomplete history |
| `NOT-ATYP-ROUTINE` | Rule hit reviewed and determined to be routine |

### Routine comments and exclusions
Routine system-generated messages should not be classified as atypical solely because the text is uncommon. Examples observed in the reviewed Lot History data include movement application messages, checksum verification, location messages, and ordinary quantity/status updates.

| Routine-message example | Default treatment |
| :--- | :--- |
| `MoveLots.Main.RepositionLot` | Routine system message unless associated with a confirmed exception or unusual sequence |
| `MoveLots.Main.Cancel` | Review only when the cancellation is material to the question or repeats abnormally |
| `CheckSum Verified: Wafer <n>` | Routine verification |
| `Located on Shelf ...` | Routine location evidence |
| `QTY: <n> Lot is Being Moved-In` | Routine movement evidence unless the sequence fails or is canceled |

### Recommended output fields

| Field | Purpose |
| :--- | :--- |
| `LOT` | Lot associated with the comment |
| `DATE_TIME` | Comment timestamp |
| `HISTORDER` | Sequence within the event |
| `OPER` / `OPERDESC` | Operation context |
| `MACHINE` | Equipment context when populated |
| `USERNAME` / `EMPID` | Recorded user context |
| `COMMAND` | Command context |
| `HIST_REC` | Original comment text |
| Matched rule | Keyword, event, rarity, or reviewer rule |
| Atypical category | Scrap, hold, rework, split, process exception, equipment, quality, manual intervention, or other |
| Confidence | High, moderate, or reviewer-assigned |
| Linked event ID | Identifier for the reconstructed business event |
| Reviewer disposition | Confirmed, routine, indeterminate, or escalated |
| Evidence reference | Source filename and row |

### Cross-lot and genealogy handling
* Link comments to related source and child lots only through explicit genealogy records such as `SPLIT LOT FROM/TO` relationships.
* Do not copy a source-lot comment to every descendant as a new comment event. Preserve the original event and expose it through the genealogy relationship.
* Exclude duplicated inherited rows from event counts when `IS_DUPLICATE` indicates copied history, but retain them for traceability.
* When reporting subsequent outcomes, require that the later lot is the same lot or an explicitly confirmed descendant.

---

## Part 4: Manual Interactions in Lot History
*Commands, comments, Move Status Control, ADJ OPER, and analytical rules*

### Purpose
This document defines how manual interactions should be identified in Lot History. It distinguishes direct manual commands, comment-supported interactions, Move Status Control, operation adjustments, route adjustments, and weaker candidates that require contextual review. The methodology is intended to measure human intervention without treating every `COMMENT` or routine manufacturing event as a manual touch.

### Core definitions

| Term | Definition |
| :--- | :--- |
| Direct manual action | A documented command whose purpose is to alter lot hold state, move status, operation, route, genealogy, rework status, recovery status, or material disposition. |
| Move Status Control | A deliberate adjustment to the lot movement state, such as queue, dispatched, transition, or another execution-state value. `RPOS`/`REPOSITION` and `NEW=/OLD=` status evidence are primary indicators. |
| Operation adjustment | A deliberate change to the lot operation. `LOPR` is the initiating command; `HISTCODE OP` / `HISTTRANS ADJ OPER` records the operation-adjustment effect. |
| Route adjustment | A deliberate route change initiated by `LRTE`. A route change can also generate an `ADJ OPER` effect when the current operation must be aligned to the new route. |
| Comment-supported manual interaction | A `COMMENT` that explicitly describes a person-directed instruction, decision, justification, coordination step, or disposition and can be linked to a lot event. |
| Manual-interaction candidate | A command or comment that may reflect human activity but does not, by itself, prove intentional manual intervention. |
| Manual interaction event | One reconstructed business event after grouping related rows by `LOT`, complete `DATE_TIME`, `COMMAND`, and `HISTORDER` and excluding inherited duplicates from event counts. |

### Confirmed direct manual actions

| Category | Command | Manual function | Supporting evidence |
| :--- | :--- | :--- | :--- |
| Hold and status control | `HOLD` | Places or manages a lot hold | `COMMENT`; `ON HOLD CODE`; `OFF HOLD CODE` |
| Move Status Control | `RPOS` | Adjusts queue, dispatched, transition, or related move status | `TRANS = REPOSITION`; `NEW=/OLD=` status values; reposition comments |
| Operation Control | `LOPR` | Changes the lot operation | `HISTCODE = OP`; `HISTTRANS = ADJ OPER` |
| Route Control | `LRTE` | Changes the lot route and may align the current operation | `ROUTE` records; possible `OP` / `ADJ OPER` effect |
| Genealogy Management | `SPLT` | Splits material into a child or target lot | `SPLIT LOT`; `PART NUM`; `ROUTE`; split attributes |
| Genealogy Management | `LMRG` | Merges eligible lots or quantities | `MERGE LOTS`; `MERGE QTY` |
| Rework Management | `REWK` | Moves a full or partial lot through rework | Route-leave and split-related effects |
| Material Recovery | `RPUR` | Repurposes previously scrapped material for approved use | `CREATE LOT`; `REPURP LOT`; `ROUTE`; owner records |
| Material Disposition | `MDS` | Records material disposition, including wafer scrap | `ADHOCSCRAP` and associated `COMMENT` rows |

### Move Status Control
Move Status Control is distinct from changing the lot operation or route. It changes the lot’s execution or movement state while the manufacturing operation can remain the same.

* **Primary command:** `RPOS`
* **Primary transaction evidence:** `REPOSITION`
* **Detailed evidence:** `NEW=<new move status> OLD=<prior move status>`

| Evidence pattern | Interpretation | Manual classification |
| :--- | :--- | :--- |
| `COMMAND = RPOS` and `TRANS = REPOSITION` | A reposition action changed or requested a lot movement state | Confirmed manual action |
| `NEW= D :: DISPATCHED OLD= T` | The move status changed from T to dispatched | Confirmed when generated by `RPOS`; otherwise review command context |
| `NEW= Q :: IN QUEUE OLD= D` | The move status changed from dispatched to in queue | Confirmed when generated by `RPOS`; otherwise review command context |
| `MoveLots.Main.RepositionLot` comment | Application comment associated with repositioning | Supporting evidence, not a separate manual event |
| `MoveLots.Main.Cancel` comment | Application cancellation text | Candidate only unless linked to a confirmed manual command or explicit user action |

**Exclusion rule:** Ordinary `DISP`, `MVIN`, and `MOUT` records are routine production-flow events by default. Their presence alone does not establish Move Status Control or a manual interaction.

### Operation Control and ADJ OPER
Operation Control changes the lot’s assigned manufacturing operation. The `LOPR` command is the direct manual command, while `OP` / `ADJ OPER` is the detailed history effect that records the operation adjustment.

`COMMAND = LOPR` | `HISTCODE = OP` | `HISTTRANS = ADJ OPER`

| Record | Role |
| :--- | :--- |
| `LOPR` | Initiates Change Lot Operation |
| `OP` | History code associated with the operation adjustment |
| `ADJ OPER` | History transaction describing the adjusted lot operation |
| `OPER` / `OPERDESC` | Identifies the resulting operation context |
| `COMMENT` | May document the reason, instruction, approval, or correction |

**Counting rule:** Count the `LOPR` command and its `ADJ OPER` row as one reconstructed manual event, not two independent manual touches.

### Route Control and related ADJ OPER effects
`LRTE` is the direct manual command for changing a lot route. A route change can write both `ROUTE` history and `OP` / `ADJ OPER` history when the new route requires the current operation to be adjusted.

* **Command:** `COMMAND = LRTE`
* **Possible effects:** `HISTCODE = RT` / `TRANS = ROUTE` and `HISTCODE = OP` / `HISTTRANS = ADJ OPER`

**Interpretation rule:** When `ADJ OPER` is generated by `LRTE`, classify the event primarily as Route Control with an associated operation-adjustment effect. Do not double-count it as a separate `LOPR` event unless a distinct `LOPR` command exists.

### Comment-supported manual interactions

| Comment category | Examples of evidence | Recommended confidence |
| :--- | :--- | :--- |
| Disposition narrative | “MDS Scrap,” “Particle Scrap,” “FULL-LOT SCRAP MAIL SENT” | High when linked to `MDS`/`ADHOCSCRAP` |
| Hold justification | “Hold for possible split,” “Hold for Anthony to split lot” | High when linked to `HOLD` or hold-code records |
| Move-status explanation | Comment explaining manual reposition, queue correction, dispatch correction, or canceled movement | High when linked to `RPOS`; candidate otherwise |
| Operation-adjustment explanation | Comment explaining why the lot was moved to another operation | High when linked to `LOPR` / `ADJ OPER` |
| Route-adjustment explanation | Comment explaining route reassignment or rerouting | High when linked to `LRTE` / `ROUTE` |
| Split or merge coordination | “Completed splits,” “Split lot process completed,” “Running split lot” | High when linked to `SPLT`/`LMRG`; moderate otherwise |
| Engineering or rework instruction | Engineering review, rerun, repeat, rework, or directed processing | High when linked to `REWK` or an explicit manual event; moderate otherwise |
| Investigation activity | Request for investigation, review, or disposition | Moderate unless linked to a confirmed exception |

### Evidence levels

| Level | Evidence | Classification |
| :--- | :--- | :--- |
| 1 | Confirmed manual command: `HOLD`, `RPOS`, `LOPR`, `LRTE`, `SPLT`, `LMRG`, `REWK`, `RPUR`, or `MDS` | Confirmed direct manual action |
| 2 | Detailed effect linked to confirmed command, including `REPOSITION` or `ADJ OPER` | Confirmed component of the manual event |
| 3 | Explicit human-action comment linked through `LOT`, complete `DATE_TIME`, `COMMAND`, and `HISTORDER` | Confirmed comment-supported interaction |
| 4 | Controlled manual-action phrase or context-dependent command with incomplete event linkage | Manual-interaction candidate |
| 5 | Documented reviewer judgment with reason and evidence | Reviewer-classified; confidence stated separately |

### Candidate manual actions

| Category | Command / record | Why context is required |
| :--- | :--- | :--- |
| Dispatching | `DISP` | Dispatch may be automated or manually initiated. |
| Routine movement | `MVIN` / `MOUT` | Move-in and move-out normally represent production execution, not manual intervention. |
| Lot administration | `BANK` | Requires the specific transaction and context. |
| Attribute management | `SETP` / `SETE` | Flag updates may be manual or workflow-driven. |
| Application comments | `MoveLots.Main.Cancel` / `RepositionLot` | System text can describe application execution rather than an additional human decision. |
| Generic `COMMENT` | `COMMENT` without explicit manual wording or a linked manual command | A `COMMENT` row alone is not sufficient evidence. |

### Event-linking and counting rules
* Group related rows by `LOT`, complete `DATE_TIME`, and `COMMAND`; order them using numeric `HISTORDER`.
* Treat a command and its detailed effects as one business event. Examples: `LOPR + ADJ OPER`; `LRTE + ROUTE + related ADJ OPER`; `RPOS + REPOSITION + NEW=/OLD= status rows`.
* Use `IS_DUPLICATE` to exclude inherited or copied history from event counts while retaining it for traceability.
* Across lots, link only through explicit genealogy such as `SPLIT LOT FROM/TO` or merge relationships.
* Do not classify `DISP`, `MVIN`, or `MOUT` as manual solely because `USERNAME` is populated; require command, comment, or sequence evidence of intervention.

### Recommended analytical categories

| Reporting category | Included commands and evidence |
| :--- | :--- |
| Hold and Status Control | `HOLD` and linked hold/release comments |
| Move Status Control | `RPOS`; `REPOSITION`; `NEW=/OLD=` movement-status effects |
| Operation Control | `LOPR`; `OP` / `ADJ OPER`; linked operation-adjustment comments |
| Route Control | `LRTE`; `ROUTE`; related `OP` / `ADJ OPER` effects |
| Genealogy Management | `SPLT`, `LMRG`, and linked split/merge comments |
| Rework Management | `REWK` and linked rework instructions |
| Material Disposition | `MDS`, `ADHOCSCRAP`, and linked disposition comments |
| Material Recovery | `RPUR` and linked recovery/engineering-use comments |
| Comment-only manual interaction | Explicit manual instruction or decision without a confirmed manual command |
| Candidate manual interaction | Context-dependent command, controlled phrase, or reviewer flag requiring confirmation |

### Recommended output fields

| Field | Purpose |
| :--- | :--- |
| `ManualEventID` | Stable identifier for the reconstructed event |
| `LOT` / `RootLineage` | Current lot and genealogy-level identity |
| `DATE_TIME` / `HISTORDER` | Chronology and within-event sequence |
| `COMMAND` / `TRANS` / `HISTCODE` | Initiating command and detailed effects |
| `OldMoveStatus` / `NewMoveStatus` | Parsed from `NEW=/OLD=` movement-status records |
| `OldOperation` / `NewOperation` | Operation change associated with `LOPR` / `ADJ OPER` |
| `OldRoute` / `NewRoute` | Route change associated with `LRTE` / `ROUTE` |
| `ManualCategory` | Hold, move status, operation, route, genealogy, rework, disposition, recovery, or comment-only |
| `EvidenceLevel` | Evidence Level 1 through 5 |
| `CommentText` / `MatchedRule` | Original text and classification rule |
| `IS_DUPLICATE` | Duplicate-history control |
| `EvidenceReference` | Source file and source row |

### Percentage methodology

| Metric | Numerator | Denominator |
| :--- | :--- | :--- |
| Manual-event rate | Distinct reconstructed confirmed manual events | All eligible reconstructed lot events |
| Move-status-control rate | Distinct `RPOS`-based Move Status Control events | All eligible reconstructed lot events or all manual events, explicitly stated |
| Operation-adjustment rate | Distinct `LOPR` events, counted once even when multiple `ADJ OPER` rows exist | All eligible reconstructed lot events or all manual events, explicitly stated |
| Route-control rate | Distinct `LRTE` events, including associated `ROUTE` and `ADJ OPER` effects as one event | All eligible reconstructed lot events or all manual events, explicitly stated |
| Manual-interaction lot rate | Distinct eligible lots with at least one confirmed or comment-supported manual interaction | All eligible lots with observable Lot History |

### Recommended reporting language
> A manual interaction was counted when the event was generated by a confirmed manual command (`HOLD`, `RPOS`, `LOPR`, `LRTE`, `SPLT`, `LMRG`, `REWK`, `RPUR`, or `MDS`), or when an explicit human-action comment was linked to the event. `RPOS`/`REPOSITION` and `NEW=/OLD=` movement-state records were classified as Move Status Control. `LOPR` with `OP` / `ADJ OPER` was classified as Operation Control. `LRTE` was classified as Route Control; any associated `ADJ OPER` row was treated as an effect of the same `LRTE` event rather than a separate manual touch. `DISP`, `MVIN`, and `MOUT` were treated as routine flow unless additional evidence established manual intervention.

### Conclusion
Manual-interaction analysis should distinguish movement-state adjustments, operation changes, and route changes. `RPOS` represents Move Status Control; `LOPR` represents Operation Control and is supported by `OP` / `ADJ OPER`; `LRTE` represents Route Control and may also generate an `ADJ OPER` effect. Grouping each command with its detailed history rows prevents double counting and provides a more accurate view of human intervention in the lot lifecycle.