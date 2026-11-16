# LotHistV Semantic Routing and Query Guidance

**Purpose:** Knowledge-base documentation for a Microsoft Copilot agent querying `[TrainingVision].[LotHistV]`  
**Prepared:** 27 October 2026  
**Source material:** View definition and metadata supplied in the request; *Lot History Copilot Validation Question and Answer Guide*, dated 12 October 2026.

---

## 1. Purpose and design principle

This document describes the meaning, lineage, and appropriate use of the fields exposed by `TrainingVision.LotHistV`, then maps request categories to the evidence needed to answer them. It is intended to help an agent interpret a user's intent and build a suitable query at runtime.

This is not a question-and-answer catalogue. Do not store a fixed list of test questions, expected answers, or one SQL statement per question. Instead:

1. Classify the user's intent into one or more semantic categories in Section 6.
2. Extract the lot, tool/machine, operation, event/code, date range, metric, and any user-defined threshold from the request.
3. Select the relevant columns using the category map and the field definitions in Sections 4–5.
4. Apply event definitions from an approved code dictionary or SME rules. Do not assume that a code's meaning is obvious from its spelling.
5. Query the data, preserve transaction-level evidence, calculate only when the event definitions support the calculation, and state limitations clearly.

The column-to-category mapping is intentionally explicit because it describes the current schema. The natural-language questions and SQL predicates must remain dynamic.

---

## 2. Source object and lineage

### 2.1 View definition

```sql
CREATE VIEW [TrainingVision].[LotHistV]
AS
SELECT
    H.LOT,
    H.DATE_TIME,
    H.HISTORDER,
    C.HISTTRANS AS TRANS,
    H.OPER,
    H.MASK_LVL,
    H.OPERDESC,
    CONCAT(CAST(H.MASK_LVL AS VARCHAR(50)), '.', H.OPERDESC) AS OPERLONGDESC,
    H.MACHINE,
    H.USERNAME,
    TrainingVision.PF_LOTHIST_HISTORY3(H.HISTCODE, H.COMMENT) AS HIST_REC,
    H.HISTCODE,
    H.COMMAND,
    C.SHORTREPORT,
    C.VIEWFLAG,
    CASE
        WHEN H.EMPID IN ('00000', '99999', 'SYSTM') THEN 0
        ELSE 1
    END AS Is_Person,
    H.IS_DUPLICATE,
    H.EMPID
FROM [Polar_Lakehouse_POC].[VisionProd].[dbo].[LOTHIST] H
INNER JOIN [Polar_Lakehouse_POC].[VisionProd].[dbo].[HISTCODES] C
    ON C.HISTCODE = H.HISTCODE
WHERE C.VIEWFLAG IN ('E', 'I');
```

### 2.2 Source and transformation rules

| Item | Definition | Agent implications |
| :--- | :--- | :--- |
| Primary object | `[TrainingVision].[LotHistV]` | Use the connected view as the source for this documentation unless the deployment explicitly authorizes other objects. |
| History source | `[Polar_Lakehouse_POC].[VisionProd].[dbo].[LOTHIST] (H)` | Supplies the lot, timestamp/order, operation, machine, user, history code, command, employee ID, duplicate flag, and raw comment used by the parsing function. |
| Code lookup | `[Polar_Lakehouse_POC].[VisionProd].[dbo].[HISTCODES] (C)` | Joined by `H.HISTCODE = C.HISTCODE`; supplies `TRANS` (`C.HISTTRANS`), `SHORTREPORT`, and `VIEWFLAG`. |
| Narrative transformation | `TrainingVision.PF_LOTHIST_HISTORY3(H.HISTCODE, H.COMMENT)` | Produces `HIST_REC`, a derived narrative field. It is not the raw `H.COMMENT` column. The exposed metadata reports `HIST_REC` as `varchar(45)`, so text may be abbreviated or insufficient for an exact-comment requirement. |
| Operation description | `CONCAT(CAST(H.MASK_LVL AS VARCHAR(50)), '.', H.OPERDESC)` | Produces `OPERLONGDESC`, joining the mask level and operation description. |
| Person indicator | `CASE WHEN EMPID IN ('00000','99999','SYSTM') THEN 0 ELSE 1 END` | `Is_Person` is a rule-based indicator from `EMPID`, not definitive proof that an event was manually initiated. |
| View-level filter | `C.VIEWFLAG IN ('E','I')` | The view intentionally excludes source records with other `VIEWFLAG` values. Do not describe the view as the complete raw history without acknowledging this filter. |

### 2.3 Row-grain and history interpretation

Treat each returned row as a history record exposed by the view, not as a unique lot or a unique timestamp. Multiple records can belong to the same lot and share both `DATE_TIME` and `HISTORDER`, while describing different aspects of an event. Do not deduplicate only by `LOT`, timestamp, or operation.

The join assumes the `HISTCODES` match is suitable for the view's intended grain. If the code table can contain multiple rows for one `HISTCODE`, validate that join cardinality before trusting row counts.

---

## 3. Data boundaries the agent must remember

* `LotHistV` has no explicit `STATUS` column. Current state can only be inferred if documented event/status semantics support it.
* There are no dedicated columns for hold state/reason, alarm state, scrap reason/quantity, move status, queue state, process duration, manual reposition old/new values, or parent/child genealogy.
* Some of those values may be represented in `TRANS`, `COMMAND`, `HISTCODE`, or the derived `HIST_REC`, but the supplied metadata does not define the complete code vocabulary or guarantee that each value is present there.
* `MACHINE` identifies the machine recorded on a history row. By itself, it does not prove that a lot was processed on the machine rather than moved, queued, commented on, or otherwise associated with it.
* `HIST_REC` is derived and exposed as `varchar(45)`. Raw `H.COMMENT` is not selected as a separate output column.
* A first or latest event in the connected extract is only the first or latest available qualifying record. It does not prove that the extract contains the lot's entire lifetime.
* Future dispatch/scheduling, operator intent or thoughts, and predictions are not supported by this historical view alone.

---

## 4. Column catalogue

The types and lengths below are the view metadata supplied for this object. For many string expressions, the reported maximum length is 8000; do not interpret that as a business rule or as proof that all values contain 8000 characters.

| Column | Type and nullability | Source / derivation | Semantic use and cautions |
| :--- | :--- | :--- | :--- |
| 1. LOT | `varchar(8000)`<br>Nullable: Yes | `H.LOT` | Lot identifier. Use for lot-scoped history and distinct-lot aggregations. Preserve the identifier as text. |
| 2. DATE_TIME | `datetime2` (precision 26, scale 6; storage length 8)<br>Nullable: Yes | `H.DATE_TIME` | Event timestamp. Use for date filters, chronological ordering, and elapsed-time calculations when start/end event rules are defined. |
| 3. HISTORDER | `smallint`<br>Nullable: Yes | `H.HISTORDER` | Ordering value associated with history records. Use with `DATE_TIME`; ties may remain. |
| 4. TRANS | `varchar(8000)`<br>Nullable: Yes | `C.HISTTRANS` | Transaction/event label from the history-code lookup. Do not confuse it with a raw transaction column from `LOTHIST`, and do not assume all labels are fully defined by this document. |
| 5. OPER | `int`<br>Nullable: Yes | `H.OPER` | Operation identifier. Use with descriptions when grouping or tracing route progression. A numeric identifier alone does not explain the operation's business meaning. |
| 6. MASK_LVL | `varchar(8000)`<br>Nullable: Yes | `H.MASK_LVL` | Mask/level value recorded on the history row. The supplied metadata does not define the manufacturing semantics of each value. |
| 7. OPERDESC | `varchar(8000)`<br>Nullable: Yes | `H.OPERDESC` | Operation description. Useful for readable output; use `OPER` alongside it for grouping to avoid relying only on text. |
| 8. OPERLONGDESC | `varchar(8000)`<br>Nullable: No | `CONCAT(CAST(H.MASK_LVL AS VARCHAR(50)), '.', H.OPERDESC)` | Combined level and operation description. Derived display value; it is not a separate source field. |
| 9. MACHINE | `varchar(8000)`<br>Nullable: Yes | `H.MACHINE` | Recorded machine/tool association. For claims about processing, require a qualifying processing-event definition rather than filtering on machine alone. |
| 10. USERNAME | `varchar(8000)`<br>Nullable: Yes | `H.USERNAME` | User name recorded with the event. May be generic/system-oriented; use with `EMPID` and `Is_Person` and avoid assuming identity or intent. |
| 11. HIST_REC | `varchar(45)`<br>Nullable: Yes | `TrainingVision.PF_LOTHIST_HISTORY3(H.HISTCODE, H.COMMENT)` | Derived history text/narrative. Search for documented reasons, quantities, comments, status changes, wafer references, or genealogy clues. Text may be shortened; it is not the raw comment. |
| 12. HISTCODE | `varchar(8000)`<br>Nullable: Yes | `H.HISTCODE` | Source event/history code. Resolve its meaning using an approved code dictionary. |
| 13. COMMAND | `varchar(8000)`<br>Nullable: Yes | `H.COMMAND` | MES command recorded on the row. The field name or an individual value does not establish its complete semantics; use the command dictionary. |
| 14. SHORTREPORT | `varchar(8000)`<br>Nullable: Yes | `C.SHORTREPORT` | Reporting-related flag from `HISTCODES`. Its detailed business meaning and allowed values should come from the code/reference definition. |
| 15. VIEWFLAG | `varchar(8000)`<br>Nullable: Yes | `C.VIEWFLAG` | Source visibility/filter classification. The view retains only values E and I; do not infer more detailed meanings without a documented definition. |
| 16. Is_Person | `int`<br>Nullable: No | Derived from `H.EMPID`: 0 for 00000, 99999, SYSTM; otherwise 1 | Indicates whether `EMPID` falls outside the three listed special values. It is a proxy, not a universal manual-vs-automated event classifier. |
| 17. IS_DUPLICATE | `bit`<br>Nullable: Yes | `H.IS_DUPLICATE` | Source duplicate indicator. Decide how it affects a metric explicitly; do not silently drop or double-count rows. Confirm business semantics and the approved deduplication key. |
| 18. EMPID | `varchar(8000)`<br>Nullable: Yes | `H.EMPID` | Employee/system identifier recorded with the history row. Interpret with `USERNAME`, `Is_Person`, and event semantics. |

### 4.1 Useful field groupings

| Evidence need | Core columns | What they establish | What they do not establish alone |
| :--- | :--- | :--- | :--- |
| Identify and sequence records | `LOT`, `DATE_TIME`, `HISTORDER` | Lot scope and available ordering | Complete lifetime coverage or a unique order when keys tie |
| Classify an event | `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC` | Event labels, codes, and narrative evidence | Definitive hold/scrap/rework meanings without a code dictionary |
| Place event in route/equipment context | `OPER`, `MASK_LVL`, `OPERDESC`, `OPERLONGDESC`, `MACHINE` | Operation and recorded machine context | That the machine performed processing for every row |
| Understand actor/source | `USERNAME`, `EMPID`, `Is_Person` | Recorded user/employee and derived person flag | Motive, identity certainty, or manual action solely from the flag |
| Assess reporting/duplicate handling | `SHORTREPORT`, `VIEWFLAG`, `IS_DUPLICATE` | Flags supplied by the source view | Unprovided business semantics or a universal deduplication rule |

---

## 5. General query and interpretation rules

### 5.1 Identify the output the user is asking for

Distinguish the requested result type before querying:
* **Record retrieval:** return the supporting history rows.
* **Latest available event:** identify the most recent event record(s), preserving ties.
* **Current-state inference:** derive a state only where an approved mapping from history events to states exists.
* **Count:** state whether the unit is history records/events or distinct lots.
* **Metric:** define numerator, denominator, event qualification, units, time window, and any exclusions.
* **Ranking/trend:** use a consistent definition across the whole comparison and show the underlying population.
* **Narrative summary:** summarize source-supported chronology and keep interpretation distinct from observed fields.

### 5.2 Chronology and latest-record handling

Use `DATE_TIME` and `HISTORDER` together for ordering. If two or more records tie on both values, preserve the tied records or report the ambiguity; this view exposes no additional guaranteed unique tie-breaker. Do not select an arbitrary row and present it as the definitive latest state.

For full-history retrieval, return chronological records. For a latest-event analysis, rank records within each lot by descending `DATE_TIME` and `HISTORDER` and retain all rows tied for the top rank unless an approved key resolves the tie.

### 5.3 Event counts and duplicate handling

* Use an event/record count when the requested unit is recorded history events.
* Use a distinct `LOT` count when the requested unit is affected lots.
* State the exact event filter and date boundaries.
* Inspect `IS_DUPLICATE`; do not assume that filtering it is always appropriate. If the approved meaning and key are not documented, disclose the ambiguity or present both raw and deduplicated counts when material.
* Do not collapse records merely because they share `LOT`, `DATE_TIME`, `HISTORDER`, `OPER`, or `COMMAND`.

### 5.4 Time windows and elapsed durations

Apply date filters to `DATE_TIME`. Resolve relative periods (for example, a recent shift, week, or month) using an explicit date range and the agreed timezone. For timestamp intervals, prefer a half-open interval (`DATE_TIME >= @StartDate AND DATE_TIME < @EndDate`) to avoid boundary double-counting across adjacent windows.

Elapsed time, queue time, dwell time, and cycle time require agreed start and end event definitions. `DATE_TIME` supplies timestamps; it does not identify which rows mark entry, release, completion, or queue exit.

### 5.5 Code and narrative interpretation

Treat `HISTCODE`, `COMMAND`, `TRANS`, and values in `HIST_REC` as evidence, not automatically self-defining business logic. Use an approved data dictionary or SME-confirmed mapping for hold, release, equipment alarm, scrap, processing, rework, reposition, split/merge, and other domain classifications. If the mapping is unavailable, explain what matching records show without asserting an unsupported event classification.

### 5.6 SQL generation safety and efficiency

* Query only `[TrainingVision].[LotHistV]` unless the connected environment explicitly grants and documents another source.
* Select only the fields needed for the intent and evidence; avoid `SELECT *` in normal answers.
* Use parameters for lot IDs, codes, machines, operations, and date boundaries. Never interpolate raw user text into SQL syntax.
* Apply the lot/date/event filters as early as possible; use aggregation only after the event-selection rule is defined.
* Return a small evidence sample for aggregates and retain traceable source rows for audit.
* Do not invent column names such as `STATUS`, `HOLD_REASON`, `SCRAP_QTY`, `QUEUE_STATUS`, `OLD_STATUS`, `NEW_STATUS`, `PARENT_LOT`, or `CHILD_LOT`; those columns are not exposed by this view.

---

## 6. Intent category to evidence mapping

Classify the intent conceptually. A request can span more than one category (for example, a tool exposure followed by a hold), so combine the appropriate mappings rather than forcing every request into one bucket.

### Lot identity, start/end, latest event, or full timeline
* **Signals / concepts to recognize:** Lot lookup, current/latest, start, end, full history, chronological account
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `OPER`, `OPERDESC`, `OPERLONGDESC`, `MACHINE`, `USERNAME`, `EMPID`, `Is_Person`, `IS_DUPLICATE`
* **Derivation pattern:** Filter by lot; order events; use the first/last qualifying available record for start/end claims; return supporting event rows.
* **Guardrails:** No explicit status field; latest available row is not automatically the current state. Mention history-window limits.

### Hold, release, equipment alarm, or stated cause
* **Signals / concepts to recognize:** Hold/release, alarm, EQALRM, reason, recovery
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `OPER`, `OPERLONGDESC`, `MACHINE`, `USERNAME`, `EMPID`, `Is_Person`, `IS_DUPLICATE`
* **Derivation pattern:** Find events using an approved event/code mapping; summarize documented reason and sequence; aggregate only qualifying events.
* **Guardrails:** This schema supplies no dedicated hold/alarm flag or hold-reason field. The supplied metadata does not define EQALRM semantics.

### Scrap and quantity loss
* **Signals / concepts to recognize:** Scrap, discarded material, quantity, reason, scrap trends
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `OPER`, `OPERLONGDESC`, `MACHINE`, `IS_DUPLICATE`
* **Derivation pattern:** Identify scrap events from a validated mapping; extract quantity only if the narrative or an approved source encodes it consistently; aggregate with stated units.
* **Guardrails:** No explicit scrap quantity or scrap-reason column. Do not sum arbitrary numbers from `HIST_REC` without validating meaning and units.

### Tool/machine exposure and processing
* **Signals / concepts to recognize:** Tool/machine visits, route operation, processing evidence, throughput
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `MACHINE`, `OPER`, `OPERDESC`, `OPERLONGDESC`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `IS_DUPLICATE`
* **Derivation pattern:** Filter by machine/operation and a validated event family; distinguish merely recorded on a machine from qualifying processing; count distinct lots or events as requested.
* **Guardrails:** `MACHINE` alone is not proof of processing. Define “visited” separately from “processed.”

### Cycle time, queue, dwell, and delay
* **Signals / concepts to recognize:** Duration, wait, queue, stuck time, movement delay, longest/average elapsed time
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `OPER`, `OPERDESC`, `OPERLONGDESC`, `MACHINE`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`
* **Derivation pattern:** Pair validated start and end events; calculate elapsed time; document inclusion/exclusion of holds, rework, and non-processing time.
* **Guardrails:** No dedicated queue entry/exit or duration fields. Do not calculate a duration until the event-boundary definitions are known.

### Repeat visits and rework
* **Signals / concepts to recognize:** Repeated operation, revisit, rework, repeat processing, rework rate
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `OPER`, `OPERDESC`, `OPERLONGDESC`, `MACHINE`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `IS_DUPLICATE`
* **Derivation pattern:** Sequence qualifying operation events per lot; identify repeated operation visits; classify rework only by an approved rework rule; calculate rates with explicit numerator and denominator.
* **Guardrails:** A repeated operation is not, by itself, proof of rework.

### Current processing/waiting state and manual reposition
* **Signals / concepts to recognize:** Processing, waiting, move status, reposition, old/new status, interrupted flow
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `OPER`, `OPERLONGDESC`, `MACHINE`, `USERNAME`, `EMPID`, `Is_Person`, `IS_DUPLICATE`
* **Derivation pattern:** Read the latest valid state-changing sequence only when the status mapping is documented; look for old/new status text only if encoded in `HIST_REC`; distinguish automated from manual events using approved rules.
* **Guardrails:** There is no `STATUS`, `OLD_STATUS`, or `NEW_STATUS` output column. Labels such as P and Waiting require a reference mapping. `Is_Person` alone is insufficient to prove manual action.

### Manual intervention and actor activity
* **Signals / concepts to recognize:** Manual action, operator involvement, override, comment, release, adjustment
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `USERNAME`, `EMPID`, `Is_Person`, `OPERLONGDESC`, `MACHINE`
* **Derivation pattern:** Select rows meeting a validated manual-event rule; show event sequence, actor fields, and recorded narrative.
* **Guardrails:** Do not infer what an operator intended or thought. A user name or person flag is not enough to classify the event as manual.

### Comments, atypical events, wafer mentions, and narrative search
* **Signals / concepts to recognize:** Comment, unusual/abnormal text, keyword, wafer ID, documented reason
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `HIST_REC`, `TRANS`, `HISTCODE`, `COMMAND`, `OPER`, `OPERLONGDESC`, `MACHINE`, `USERNAME`, `EMPID`
* **Derivation pattern:** Search the available narrative for supplied terms or identifiers; return the matching record and context; define the rule used to label a comment “atypical.”
* **Guardrails:** Raw `COMMENT` is not exposed. `HIST_REC` is only `varchar(45)` and may not contain the complete original text.

### Split, merge, parent/child, and genealogy
* **Signals / concepts to recognize:** Split/merge, parent lot, child lot, ancestry, genealogy tree
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `OPERLONGDESC`, `MACHINE`, `USERNAME`
* **Derivation pattern:** Extract parent/child links only if an approved event mapping or narrative format explicitly records them; preserve supporting source events.
* **Guardrails:** No dedicated parent/child columns or guaranteed genealogy relation. Never invent links from similar lot IDs alone.

### Cross-lot trends, bottlenecks, and process-area comparisons
* **Signals / concepts to recognize:** Ranked operations/tools, most common reasons, queue depth/time, high-volume area, trend
* **Primary columns to inspect:** Category-specific event fields plus `LOT`, `DATE_TIME`, `OPER`, `OPERDESC`, `OPERLONGDESC`, `MACHINE`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `IS_DUPLICATE`
* **Derivation pattern:** Define the event population, period, grouping, measure, ranking, and whether the metric counts events or lots; map operation to process area only when a documented mapping exists.
* **Guardrails:** No explicit queue-depth, process-area, or business-reason dimension. A ranking is only as valid as its classification rule.

### Cross-event sequence or cohort correlation
* **Signals / concepts to recognize:** Visited one operation then later had another event; subsequent hold; event order across lots
* **Primary columns to inspect:** `LOT`, `DATE_TIME`, `HISTORDER`, `OPER`, `OPERLONGDESC`, `MACHINE`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`
* **Derivation pattern:** Match qualifying events for the same lot, then enforce temporal order based on timestamp/order; clearly define “later/subsequently” and the observation window.
* **Guardrails:** Do not treat mere coexistence as causality. Define cohort window and sequence rules.

### Event family lookup or special code counts
* **Signals / concepts to recognize:** A named command, history code, transaction family, or domain token
* **Primary columns to inspect:** `COMMAND`, `HISTCODE`, `TRANS`, `HIST_REC`, `LOT`, `DATE_TIME`, `HISTORDER`, `OPERLONGDESC`, `MACHINE`
* **Derivation pattern:** Use the exact code or token requested, plus an approved definition when one exists; return each included event type and count basis.
* **Guardrails:** An unfamiliar code is opaque until the code dictionary or SME defines it. A code substring may not equal a business event.

### Evidence, definitions, filters, and confidence
* **Signals / concepts to recognize:** Supporting records, why an answer was returned, filters, confidence, assumptions
* **Primary columns to inspect:** Relevant event fields plus `SHORTREPORT`, `VIEWFLAG`, `IS_DUPLICATE`, `USERNAME`, `EMPID`, `Is_Person` when relevant
* **Derivation pattern:** Present the selected rows, date bounds, filters, counting rule, code definitions, assumptions, and whether the result is observed or calculated.
* **Guardrails:** Always disclose ambiguous codes, tied records, truncated narrative risk, excluded view flags, or incomplete coverage when material.

### Unsupported scope or missing records
* **Signals / concepts to recognize:** Future schedules, prediction, unrecorded intent, nonexistent lot, earlier period outside extract
* **Primary columns to inspect:** Use `LOT` for existence checks; use relevant fields only if some supporting event exists
* **Derivation pattern:** Verify source presence and coverage; state when the view does not support the requested fact.
* **Guardrails:** Do not invent rows, future schedules, predictions, motives, or pre-extract history.

---

## 7. Runtime routing workflow for the Copilot agent

Use this as a repeatable reasoning workflow, not as a set of question-specific branches.

1. **Parse the intent.** Identify whether the user wants retrieval, latest-event/state, a count, a duration, a ranking, a cross-event correlation, a narrative, or an explanation of evidence. Allow multiple categories.
2. **Resolve the parameters.** Capture lot IDs, machine/tool names, operation IDs or descriptions, event codes/terms, date range, units, threshold, cohort, and desired granularity. Ask a concise follow-up only when a missing parameter materially changes the result; otherwise state a reasonable, explicit assumption.
3. **Choose fields from the category map.** Include identifying/time fields and the event-classification evidence, plus the fields required for the requested output. Do not choose columns based on memorized question wording.
4. **Resolve event semantics.** Use the approved code dictionary and SME-approved definitions. If a definition is not available, query observable codes/text but qualify the result instead of silently treating it as a known event.
5. **Build a parameterized query.** Use `[TrainingVision].[LotHistV]`, apply the source-level view restriction implicitly through the view, filter by the requested lot/date/machine/operation, and select only necessary fields.
6. **Preserve the grain.** Keep individual rows when event sequence, narrative, actor, or code differences matter. Choose event count versus distinct-lot count deliberately. Apply duplicate policy only when defined.
7. **Calculate transparently.** Show the start/end event rule for durations, the numerator and denominator for rates, the period and population for rankings, and the units for quantities.
8. **Validate output against evidence.** Ensure each conclusion is supported by returned rows and that records match the requested time/event scope. Check for timestamp/order ties and null or truncated fields.
9. **Respond with evidence and boundaries.** Separate directly observed facts from calculated values and interpretation. Mention unsupported fields, unavailable definitions, data-window limitations, and assumptions that materially affect the answer.

---

## 8. Recommended evidence returned with an answer

Include the fields that support the specific result. For a lot-level chronology or event investigation, the usual evidence set is:

`LOT`, `DATE_TIME`, `HISTORDER`, `TRANS`, `HISTCODE`, `COMMAND`, `OPER`, `OPERDESC` or `OPERLONGDESC`, `MACHINE`, `HIST_REC`, `USERNAME`, `EMPID`, `Is_Person`, `IS_DUPLICATE`.

Also report, when applicable:
* The date/time window and timezone used.
* The event/code definition and filter applied.
* Whether a number represents event records, distinct lots, or a calculated quantity.
* The duplicate-handling rule.
* The start and end event definitions for durations.
* The operation-to-process-area mapping or other reference mapping used.
* Any ties, nulls, unavailable fields, truncated narrative, or extract-boundary limitations.

Do not expose unnecessary personal or system identifiers in summaries. Include actor fields when they are relevant to the user's question or needed for audit evidence.

---

## 9. Business definitions and reference data needed for full coverage

The following mappings are not included in the supplied view metadata. To make advanced answers reliable, maintain these as a separate, approved data dictionary or business rules document; do not bury them in a collection of hardcoded questions.

| Required definition | Why it is needed | Related fields |
| :--- | :--- | :--- |
| `HISTCODE` catalogue and descriptions | Converts technical event codes into approved event families | `HISTCODE`, `TRANS`, `HIST_REC` |
| `COMMAND` catalogue and whether each command is system- or operator-generated | Supports command-level interpretation and manual/automated classification | `COMMAND`, `EMPID`, `USERNAME`, `Is_Person` |
| `HISTTRANS` / `TRANS` meanings | Defines transaction labels and event groupings | `TRANS`, `HISTCODE` |
| Hold/release and equipment-alarm rules, including EQALRM | Identifies qualifying hold/alarm events and documented recovery | `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC`, `MACHINE` |
| Processing/Waiting/move-status vocabulary, including P | Makes current-state and reposition analysis defensible | `HIST_REC`, `COMMAND`, `HISTCODE`, `TRANS` |
| Manual reposition signature and old-to-new status extraction | Distinguishes manual changes from normal automated sequences | `HIST_REC`, `COMMAND`, `EMPID`, `USERNAME`, `Is_Person` |
| Scrap event rule, quantity format, and units | Enables scrap counts and quantity totals | `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC` |
| Queue entry/exit and cycle-time boundaries | Enables elapsed-time and bottleneck calculations | `DATE_TIME`, `HISTORDER`, `OPER`, `TRANS`, `HISTCODE`, `COMMAND` |
| Rework classification rule | Separates true rework from normal repeated visits | `LOT`, `OPER`, `DATE_TIME`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC` |
| Split/merge and genealogy record format | Enables parent/child relationship extraction | `LOT`, `TRANS`, `HISTCODE`, `COMMAND`, `HIST_REC` |
| Operation-to-process-area mapping | Enables process-area rankings | `OPER`, `OPERDESC`, `OPERLONGDESC` |
| `IS_DUPLICATE` meaning and approved deduplication key | Makes record/event counts repeatable | `IS_DUPLICATE`, `LOT`, `DATE_TIME`, `HISTORDER`, event identifiers available in other sources |
| `SHORTREPORT` and `VIEWFLAG` business definitions | Clarifies report eligibility and visibility classification | `SHORTREPORT`, `VIEWFLAG` |
| Data retention/extract coverage and timezone | Defines what “first,” “latest,” “yesterday,” and “complete” mean | `DATE_TIME`, source documentation |
| Full-comment access, if exact comments are required | Enables exact quote/evidence beyond the 45-character derived field | Raw `H.COMMENT` or an approved comment source; not currently exposed as a view column |

---

## 10. Sample-row interpretation for row-grain awareness

The supplied sample has two records for lot `6053A0G8` with the same `DATE_TIME` (`2026-09-02T12:24:35.33`) and `HISTORDER` (`1`). The records differ in their history code, transaction label, narrative, and reporting/visibility flags:

| Field | Record A | Record B |
| :--- | :--- | :--- |
| COMMAND | DISP | DISP |
| HISTCODE | DL | CM |
| TRANS | DISPATCH | COMMENT |
| HIST_REC | QTY: 25 | Dispatched by Local Rules. |
| SHORTREPORT | N | Y |
| VIEWFLAG | I | E |
| IS_DUPLICATE | false | false |
| EMPID / USERNAME | SYSTM / SYSTEM | SYSTM / SYSTEM |
| Is_Person | 0 | 0 |

**Interpretation rule:** these are two distinct history records, not one row that should be collapsed because the timestamp and order match. The narrative `QTY: 25` may be a quantity clue, but its business meaning and units must be validated before using it in a scrap or production quantity calculation. This sample alone does not define all meanings of `DL`, `CM`, or `DISP`.

---

## 11. Response quality and validation standard

A response should be assessed by whether the result agrees with the source records and whether the supporting evidence is complete.

* **Pass:** correct result, complete supporting evidence, and no unsupported claims.
* **Partial:** directionally correct but incomplete, ambiguous, or missing material evidence.
* **Fail:** incorrect, unsupported, fabricated, or unable to answer an in-scope request that the connected data can answer.

For every calculated or interpreted answer, make the query scope and calculation basis inspectable. If the source cannot support the requested fact, the correct behavior is to say what is missing rather than guess.

---

## 12. Agent instruction block

The following can be adapted into the Copilot agent's main instructions. Keep it alongside this semantic guide; do not replace it with a test-question list.

> For requests about manufacturing lot history, infer the user's intent category and parameters, then use the LotHistV Semantic Routing and Query Guidance document to select the evidence columns and required business definitions. Do not match requests against a hardcoded list of example questions or expected answers. Build parameterized queries against `[TrainingVision].[LotHistV]` using the fields that fit the inferred intent. Treat `TRANS`, `HISTCODE`, `COMMAND`, and `HIST_REC` as event evidence whose business meaning must be confirmed by an approved dictionary or SME rule. Use `DATE_TIME` and `HISTORDER` for chronology and preserve tied records. Distinguish event counts from distinct-lot counts, and apply duplicate handling only under a documented rule. Do not claim a current status, hold reason, scrap quantity, queue state, rework, genealogy link, or processing event unless the available fields and validated definitions support it. `HIST_REC` is derived and only 45 characters wide in the view metadata; raw `COMMENT` is not exposed. The view filters to `VIEWFLAG` values E and I. Provide supporting transaction evidence, state filters and calculation rules, distinguish observed facts from derived conclusions, and identify unavailable fields or data coverage limits. Never invent records, future schedules, operator intent, or predictions from this historical view.

---

## 13. Quick design checklist

Before enabling the agent for production use, verify that:
* The agent's connected SQL tool can read `[TrainingVision].[LotHistV]` and that its metadata matches this document.
* `HISTCODE`, `COMMAND`, `TRANS`, `SHORTREPORT`, `VIEWFLAG`, and `IS_DUPLICATE` definitions have an approved reference source.
* Hold/alarm, scrap, processing-state, reposition, queue/cycle, rework, and genealogy rules are defined by an SME where those use cases are in scope.
* The agent is configured to use parameterized SQL and to return evidence rows for results.
* Ties, duplicate handling, date windows, timezone, null values, and limited history coverage have been tested.
* The source view's `VIEWFLAG IN ('E','I')` filter is disclosed wherever completeness matters.
* Validation uses representative records and independently verified results, not only example prompts.