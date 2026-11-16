# LotHistV Knowledge Base 2

## Definitions

**D1: Processing:** A lot-operation visit is classified as Processing when a valid Move-In (MVIN) establishes entry into processing and no subsequent Move-Out (MOUT) or other event terminating that processing episode is present. MOVESTATUS = P may be used as supporting confirmation of the state but should not, by itself, override the chronological transaction sequence.

**D2: Waiting:** The lot has arrived at or is assigned to an operation but there is no open processing episode established by MVIN.

**D3: Queue Time** is the elapsed time from ARRIVAL (LA) at an operation to the first valid MOVE IN (IN/MVIN) beginning that visit's processing episode. Dispatch (DL) is an intermediate state and does not end queue time. Gross Queue Time includes holds; Net Queue Time subtracts active hold intervals. Pre-dispatch, dispatch-to-move-in, machine-queue, and hold time should be retained as separate components when the history supports them.

**D4: Bottleneck:** An operation exhibiting sustained material accumulation and elevated waiting time relative to comparable operations or its own historical baseline. Bottlenecks should be identified primarily using net queue time, queue depth/queued WIP, and throughput. Gross queue time and hold time are reported separately, and processing time is used to determine whether the constraint is associated with processing capacity or another source of delay. A single long-running lot is an exception, not by itself evidence of an operation-level bottleneck. \*\* You may need an additional table to calculate this (LotStatus) \*\* - this may have to wait.

**D5:**

| Term | Definition |
| :--- | :--- |
| Fab processing finished | Lot has completed its required processing operations |
| At Transfer | Lot has reached the Transfer operation/staging point |
| Lot Complete | Lot Transfer (TRNF) has successfully occurred |
| Total Lot Cycle Time | CREATE LOT $\rightarrow$ Lot Transfer |

**D6: Significant Delay:** A completed operation visit whose net queue time exceeds the 95th percentile of eligible comparable visits to the same operation. Hold time is calculated and reported separately. When an approved Polar absolute delay threshold is exceeded, the visit is also classified as significant regardless of its percentile. Operations with insufficient comparison history are reported with observed duration but are not automatically classified as statistically significant.

**D7:**

| Term | Definition |
| :--- | :--- |
| Fab processing finished | Lot has completed its required processing operations |
| At Transfer | Lot has reached the Transfer operation/staging point |
| Lot Complete | Lot Transfer (TRNF) has successfully occurred |
| Total Lot Cycle Time | CREATE LOT $\rightarrow$ Lot Transfer |

**D8:**
* **Hold Episode Count:** number of transitions from not held $\rightarrow$ held.
* **Hold Code Count:** number of distinct hold-code activations.
* **Hold Duration:** elapsed wall-clock time during which at least one hold was active.

This prevents a lot with multiple hold codes during one uninterrupted stop from appearing as though it was repeatedly placed on hold.
For operation-level analysis, attribute the episode to the operation where the not-held $\rightarrow$ held transition occurred, while retaining all subsequent hold-code changes as details of that same episode.

**D9:** For tool-level analysis, I would define Visited and Processed differently. MVIN is the key event for proving that a lot actually entered a specific tool.

| Classification | Rule |
| :--- | :--- |
| Visited tool | Valid MVIN or MOUT naming that tool |
| Processed on tool | Valid MVIN naming that tool |
| Completed processing on tool | Matching MVIN $\rightarrow$ MOUT for the same lot/tool/visit |
| Currently processing | MVIN with no subsequent matching MOUT or other terminating event |
| Assigned/Dispatched only | DL naming the tool, but no MVIN |
| Not visited | Tool appears only in dispatch/comment/other records without MVIN or MOUT |

**D10: Operation Visit:** One continuous pass by a lot through a specific OPER, beginning with the valid LA / ARRIVAL event that establishes entry to the operation and ending with the event that causes the lot to leave that operation.

```text
Visit start       = LA
Queue start       = LA
Processing start  = MVIN
Processing end    = MOUT
Visit end         = departure/transition from the operation
```

One operation visit = one continuous presence of the lot at the operation, from LA until departure or an equivalent event moves the lot elsewhere. A later return to the same operation begins a new visit.

**D11:** The proportion of eligible lots, wafers, or completed normal-route operation visits that subsequently enter a confirmed rework path through a REWK event. The denominator and unit of analysis must be explicitly stated. Operations beginning with RW\_ are excluded from rankings intended to identify where rework originates, because those operations perform the rework rather than independently indicate its cause. RW\_ operations are analyzed separately for rework execution, queue time, cycle time, and bottleneck performance.
I would define rework rate from the event that sends material into rework, not from visits to RW\_ operations. Then I would exclude RW\_ operations from the primary ranking of operations causing rework, while reporting them separately as rework-execution operations.

**D12: Process Area:** A stable manufacturing grouping used to combine related operations that perform the same broad type of fabrication activity for cross-lot, cycle-time, WIP, bottleneck, and exception analysis. \*\*\* I think we will need to include another table \*\*\* this will be addressed in the future.

---

## References

**R1:**

| MoveStatus | Meaning | Status |
| :--- | :--- | :--- |
| Q | In Queue | Confirmed |
| D | Dispatched | Confirmed |
| T | In Transit | Confirmed by you |
| I | In Machine Queue (IN MACHQUE) | Confirmed from Lot History |
| P | Processing | Provisional, use MVIN $\rightarrow$ MOUT as primary processing evidence |
| M | Wait Moveout | Confirmed |
| A | Unknown | Still needs definition |

**R2:**

| Code | Flag 1 | Description | Supply | Category | Flag 2 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| BANK | Y | Waiting at BANK operation | GOODSUPPLY | BANK | N |
| CUSTAP | Y | WAIT CUSTOMER APPROVAL | GOODSUPPLY | DELAY | N |
| DESIGN | Y | DESIGN ENGINEERING | GOODSUPPLY | HOLD | N |
| EQALRM | Y | MACHINE ALARM | GOODSUPPLY | DELAY | Y |
| LIMITS | Y | OUTSIDE ENG PROC LIMITS | GOODSUPPLY | DELAY | Y |
| LRR | Y | LOT REVIEW REQUIRED | GOODSUPPLY | DELAY | Y |
| MASK | Y | WAITING FOR MASK | GOODSUPPLY | DELAY | Y |
| MISMSK | Y | MISSING MASK | GOODSUPPLY | DELAY | Y |
| NOPROG | Y | NO PROG IN SPEC'D EQUIP | GOODSUPPLY | DELAY | Y |
| NOTEST | Y | WAITING FOR TEST PROG | GOODSUPPLY | DELAY | Y |
| PRCDEV | Y | PROCESS DEVELOPMENT | GOODSUPPLY | DELAY | N |
| PRCENG | Y | PROCESS ENGINEERING | GOODSUPPLY | DELAY | Y |
| PRDCON | Y | PRODUCTION CONTROL | GOODSUPPLY | DELAY | Y |
| PROJCT | Y | PROJECT ENGINEERING | GOODSUPPLY | DELAY | Y |
| SCREEN | Y | PARAMETRIC CRITERIA | GOODSUPPLY | DELAY | Y |
| SNGWAF | Y | WAITING FOR SINGLE WAFER | GOODSUPPLY | DELAY | Y |
| STARTQ | Y | WAITING PROCESS START | GOODSUPPLY | HOLD | N |
| SUBCON | Y | SUB-CONTRACTOR | GOODSUPPLY | SUBCON | Y |
| TENCOR | Y | WAITING FOR TENCOR | GOODSUPPLY | DELAY | Y |
| PRTCLE | Y | Particles out of Limits | GOODSUPPLY | DELAY | Y |
| WAIVER | Y | Wait for WAIVER Approval | GOODSUPPLY | DELAY | N |
| VQL | Y | Visually Questionable Lot | GOODSUPPLY | DELAY | Y |
| MIA | Y | Lot cannot be Located. | GOODSUPPLY | DELAY | Y |
| PRGNOT | Y | Program Not Working | GOODSUPPLY | DELAY | Y |
| PRCTME | Y | Fail Minimum Process Time | GOODSUPPLY | DELAY | Y |
| CHKSUM | Y | Checksum Error | GOODSUPPLY | DELAY | Y |
| INSPCT | Y | Failed Inspection | GOODSUPPLY | DELAY | Y |
| MSKCHK | Y | Mask Failed Inspection | GOODSUPPLY | DELAY | Y |
| AUTOMA | Y | Automation Placed Hold | GOODSUPPLY | DELAY | Y |
| PQL | Y | Parametric Questionable | GOODSUPPLY | DELAY | Y |
| PLOTRW | Y | Partial Lot Rework | GOODSUPPLY | DELAY | Y |
| RWMERG | Y | Merge after Rework | GOODSUPPLY | DELAY | Y |
| WAFSRT | Y | WAFERSORT DATA INCOMPLETE | GOODSUPPLY | DELAY | Y |
| RTECHK | Y | ROUTE CHECK | GOODSUPPLY | DELAY | Y |
| FGALRM | Y | FABGUARD ALARM | GOODSUPPLY | DELAY | Y |
| HOTSPT | Y | Photo - HOTSPOT | GOODSUPPLY | DELAY | Y |
| REJCTS | Y | Photo - Rejected Wafers | GOODSUPPLY | DELAY | Y |
| MRB | Y | MRB needed | GOODSUPPLY | DELAY | Y |
| METRO | Y | Metrology Engineering | GOODSUPPLY | DELAY | Y |
| AUTOER | Y | Automation Error | GOODSUPPLY | DELAY | Y |
| P_ETCH | Y | Etch Process Engineering | GOODSUPPLY | DELAY | Y |
| P_BEOL | Y | BEOL Process Engineering | GOODSUPPLY | DELAY | Y |
| PPhoto | Y | Photo Process Engineering | GOODSUPPLY | DELAY | Y |
| MFGBNK | Y | BANK HOLD FOR MFG LOTS | GOODSUPPLY | DELAY | N |
| DEVBNK | Y | Bank Hold for DEV Lots | GOODSUPPLY | DELAY | N |
| QI | Y | QI or QE Lot Hold | GOODSUPPLY | HOLD | N |
| P_DIFF | Y | DIFF_LPCVD Process Eng | GOODSUPPLY | DELAY | Y |
| P_EPI | Y | EPI_IMPLANT Process Eng | GOODSUPPLY | DELAY | Y |
| APC | Y | APC Error | GOODSUPPLY | DELAY | Y |
| INDENG | Y | IE Scheduling Issue | GOODSUPPLY | DELAY | Y |
| CSTBNK | Y | Customer Bank Hold - Lot | GOODSUPPLY | HOLD | N |
| P_CMP | Y | CMP PROCESS ENGINEERING | GOODSUPPLY | DELAY | Y |
| PARAM | Y | Param Transfer Tech->Eng. | GOODSUPPLY | DELAY | Y |
| P_DEF | Y | DEFCT PROCESS ENGINEERING | GOODSUPPLY | DELAY | Y |
| P_MET | Y | METRO PROCESS ENGINEERING | GOODSUPPLY | DELAY | Y |
| S_BEOL | Y | SHIFT TECH BEOL | GOODSUPPLY | DELAY | Y |
| S_CMP | Y | SHIFT TECH CMP | GOODSUPPLY | DELAY | Y |
| S_DEF | Y | SHIFT TECH DEFECT | GOODSUPPLY | DELAY | Y |
| S_DIFF | Y | SHIFT TECH DIFFUSION | GOODSUPPLY | DELAY | Y |
| S_EPI | Y | SHIFT TECH EPI | GOODSUPPLY | DELAY | Y |
| S_ETCH | Y | SHIFT TECH ETCH | GOODSUPPLY | DELAY | Y |
| S_MET | Y | SHIFT TECH METRO | GOODSUPPLY | DELAY | Y |
| Sphoto | Y | SHIFT TECH Photo | GOODSUPPLY | DELAY | Y |

**R3:** It is primarily a parsing issue, with one important exception:
* SCRAP is not the reason. It is the record marker at the beginning of HIST_REC.
* EVENT is a real value appearing in the reason-code position, although we have not yet established the business definition of that reason code.

**R4:** does not exist

**R5:** No

**R6:** Typo for the operation description – try this question “Show all lots that visited operation METR_ETCH and were later scrapped”

**R7:** LOPR is a legitimate way for a lot to leave an operation, but it is not the normal production-flow exit.
Normal vs. exceptional exit:

| Exit mechanism | Meaning | How I would classify it |
| :--- | :--- | :--- |
| MOUT | Normal completion/exit from the operation | Normal exit |
| LOPR | Change Lot Operation | Manual/administrative exit |
| LRTE | Change Lot Route, potentially changing current operation | Route-driven exit |
| REWK | Diverts lot into a rework path | Rework exit |
| SPLT | Can move material into a child/target lot and potentially change part/route | Genealogy-driven exit |

**R8:** PRCENG-related events primarily from the hold code, not from free-text mentions.

| Record | Count as PRCENG-related? | Treatment |
| :--- | :--- | :--- |
| ON HOLD CODE:PRCENG | Yes | Starts/adds a PRCENG hold condition |
| OFF HOLD CODE:PRCENG | Yes | Releases/removes that PRCENG hold condition |
| COMMENT generated as part of the same PRCENG hold/release transaction | Supporting evidence | Show it, but don't count it as another PRCENG event |
| COMMENT containing PRCENG but no linked hold event | Mention only | Report separately, don't include in primary count |
| Unrelated record occurring while lot is on PRCENG hold | No | Context only |

**R9** – Will need another table

**R10** – No