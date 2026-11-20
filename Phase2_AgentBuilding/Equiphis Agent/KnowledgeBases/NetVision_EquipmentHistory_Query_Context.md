# NetVision Equipment History — Query Context Guide

## Purpose

Use this guide as semantic context when querying the Fabric table:

`[MES_Analytics].[TrainingVision_EQUIPHIS4]`

The guide describes how to identify the relevant fields and query patterns for equipment-history analysis.

The agent should infer which fields are needed from the user's intent, select an appropriate time range and equipment scope, and build a query that retrieves source events needed to answer the request. Prefer retrieving the underlying records first; calculate durations, counts, rankings, and timelines from those records only when the required definitions and data are available.

---

## 1. Source table and field inventory

### Equipment identity and classification

Use these fields to identify equipment, constrain a query to a tool, or group/compare equipment:

- `MACHINE` — equipment identifier/name. Use as the primary human-readable equipment filter and output identifier.
- `MachineId` — numeric machine identifier. Use when a stable numeric ID is needed, when grouping/joining is supported, or to disambiguate machines with similar names.
- `MACHINE_DESCRIPTION` — descriptive text about the machine or a component/chamber, when populated.
- `MACHINE_GROUP` — equipment group/category.
- `MACHINE_TYPE` — equipment type/family.
- `MACHINE_KIND` — equipment kind/classification.

For equipment-type comparisons, use `MACHINE_TYPE`, `MACHINE_KIND`, and/or `MACHINE_GROUP` as the grouping dimensions, as appropriate to the requested comparison. Do not assume that any one of these fields is a canonical equipment-family definition without confirmation.

### Event time and event ordering

- `DATE_TIME` — datetime field associated with the equipment-history record. It is a candidate event timestamp for chronology and interval calculations.
- `DATE_TIME_RUN` — datetime field also present on each record. Its precise business meaning is not defined in the supplied metadata. Inspect it alongside `DATE_TIME`; do not assume they are interchangeable or that one is always the authoritative event time.
- `LINEORDER` — integer field that may provide ordering/context. Its business meaning is not documented. Use it as a secondary ordering field only if record evidence or SME documentation supports that interpretation.

For timelines, retrieve both timestamp fields initially when relevant. Order by the confirmed event-time field, with a secondary ordering key only where justified. If the event-time semantics or ordering are ambiguous, disclose the ambiguity rather than silently choosing.

### Status/event descriptors

- `STATUS1_CODE`, `STATUS1_NAME` — first status code/name pair.
- `STATUS2_CODE`, `STATUS2_NAME` — second status code/name pair.

These are the primary status-related fields to inspect for status transitions and event context. Retrieve code and name together. Use status names when populated and codes when names are absent, but do not invent a code-to-status mapping. The metadata does not define which pair represents the prior state, resulting state, primary status, secondary status, or transition semantics.

### Preventive-maintenance context

- `PM_CODE`
- `PM_NAME`

Inspect these fields when identifying PM-related history. Retrieve them with timestamps, equipment identity, status fields, and comments to understand the recorded PM event. The table metadata does not define PM code meanings or the rules for determining PM start, completion, or duration.

### Repair-related context

- `REPAIR1_CODE`, `REPAIR1_NAME`
- `REPAIR2_CODE`, `REPAIR2_NAME`
- `REPAIR3_CODE`, `REPAIR3_NAME`
- `REPAIRCODE`

Inspect repair-related fields when researching repair/event context or correlating recorded repair information with status history. The supplied schema reports the six `REPAIR1/2/3_CODE` and `REPAIR1/2/3_NAME` columns as `int` (including the columns suffixed `_NAME`); preserve their reported types and do not treat them as descriptive text without checking actual data. `REPAIRCODE` is reported as `varchar`. The meaning and relationship among these fields are not documented.

### Comments, user, and record classification

- `COMMENTS` — free-text event/comment content. Search and retrieve this field for incident descriptions, recorded symptoms, context, and similar historical text.
- `USERNAME` — username associated with the record, if populated.
- `EMPID` — employee identifier, if populated.
- `COMMENTTYPE` — comment/event type classification; values and their meanings are not documented.
- `IGNORE_RECORD` — field that appears to indicate record-ignore status, but its semantics and valid values are not documented. Do not automatically exclude rows based on it unless an approved rule is supplied.
- `MACHINE_PRIORITY` — priority-like field; data type is `varchar`, and meaning is not documented. Do not assume it is numeric or a severity ranking without validation.

---

## 2. Intent-to-data retrieval patterns

Use the patterns below to determine which field groups should be retrieved. These are **semantic retrieval instructions**, not fixed query templates. Combine field groups as needed; retain source timestamps and evidence columns in the result.

### A. Latest/current state and state at a point in time

When the intent is to establish the equipment state at the present/latest recorded point or at a requested historical time:

- Scope by `MACHINE` and/or `MachineId`.
- Retrieve `DATE_TIME`, `DATE_TIME_RUN`, `STATUS1_CODE`, `STATUS1_NAME`, `STATUS2_CODE`, `STATUS2_NAME`.
- Include `COMMENTS`, `COMMENTTYPE`, `PM_CODE`, `PM_NAME`, repair-related fields when they may explain the event.
- Use the validated event-time and status-transition semantics to select the applicable record and reconstruct the state.
- For a historical point-in-time state, retrieve the event(s) immediately before and after the requested time when needed to establish the state interval.

**Important:** The table's metadata alone does not identify a definitive "current status" rule, status-transition direction, or authoritative timestamp.

### B. Chronological history, incident reconstruction, and before/while/after analysis

When the intent is a history or narrative for a machine and time window:

- Filter by equipment identifier and the requested time range.
- Retrieve both timestamp fields (until event-time semantics are confirmed), status code/name pairs, `COMMENTS`, `COMMENTTYPE`, PM fields, repair fields, `USERNAME`, `EMPID`, and equipment descriptors.
- Sort chronologically using the validated event timestamp; use `LINEORDER` only if its ordering semantics are established.
- Include the events around the target transition, not only the transition row, to provide context.
- For an incident timeline, include records from the beginning of the relevant interval through recovery/return-to-state, if those boundaries can be established from status semantics.

Do not infer causation from temporal order or comments alone. Report what is recorded and distinguish documented causes from possible interpretations.

### C. Down events, recovery, downtime, and interval duration

When the intent concerns downtime, down transitions, recovery, duration, or repeated failure intervals:

- Retrieve equipment identity, timestamp(s), both status code/name pairs, comments, event classification, and repair/PM context.
- Establish the qualifying start and end transitions using an approved mapping of status values and transition rules.
- Pair transitions into continuous intervals before counting incidents; avoid counting multiple records within the same continuous interval as separate incidents.
- Calculate interval duration from validated timestamps only after start/end rules are confirmed.
- Preserve each interval's start/end event records so calculations can be explained and audited.

**Unresolved definitions:** This table does not specify which statuses count as Down, unavailable, Production Up, scheduled downtime, unscheduled downtime, or failure; nor does it define incident-merging rules, treatment of missing transitions, or duration boundaries. Do not invent these definitions.

### D. Availability, MTTR, MTBF, counts, and rankings

When the intent is a performance metric or comparison across machines:

- Retrieve machine identity and classification fields (`MACHINE`, `MachineId`, `MACHINE_GROUP`, `MACHINE_TYPE`, `MACHINE_KIND`).
- Retrieve event timestamps, status code/name pairs, and contextual comments/classification/PM/repair fields.
- Build the qualifying event/interval set according to documented business rules.
- Aggregate only after intervals/events have been deduplicated and classified.
- Retain date range, event count, interval count, total durations, and supporting records alongside metric values.
- For rankings, group by the intended equipment identifier and sort by the requested calculated metric.

The source table does not supply metric formulas, eligible status definitions, scheduled/unscheduled classifications, or policies for partial intervals and boundary conditions. A calculation must state its formula and assumptions; if the business definition is missing, ask for it or label the result as a provisional calculation.

### E. PM history and performance after PM

When the intent concerns PM history or events occurring after a PM:

- Retrieve `PM_CODE`, `PM_NAME`, both timestamp fields, status code/name pairs, `COMMENTS`, `COMMENTTYPE`, machine identity, and repair fields.
- Determine PM event/interval boundaries using documented PM semantics.
- To examine subsequent events, retrieve history after the validated PM completion time for the requested time window.
- Link records by machine and chronology; do not assume a later Down event was caused by PM.

The table metadata does not define PM completion or the PM interval, and does not establish causal relationships.

### F. Repeated problems, similar events, and patterns

When the intent is to find recurrence on the same machine or compare similar equipment:

- Retrieve machine identity and type/group/kind fields.
- Search `COMMENTS` for the user-provided symptom/problem terms when applicable.
- Inspect status pairs, `COMMENTTYPE`, PM and repair fields, and timestamps.
- Compare records using observable attributes (matching text, event/status sequence, classifications, or documented codes).
- Return the occurrence dates, machine identifiers, associated status/event context, and interval duration if duration can be validly calculated.
- Explain the basis for similarity. Do not label similar symptoms as the same root cause unless the source explicitly establishes that.

For cross-equipment comparisons, select the peer group using the relevant machine classification fields, but verify that the selected group is appropriate before treating machines as comparable.

### G. Recorded cause or explanation

When asked what caused an event or what the history says about it:

- Retrieve the relevant incident records and the surrounding chronological context.
- Prioritize `COMMENTS`, status fields, event classification, PM and repair fields.
- Treat explicit documented cause as a recorded fact only when the source text clearly states it.
- Otherwise summarize recorded events, comments, and observed sequence, and state that the cause is not established in the available history.

---

## 3. Query construction and evidence requirements

1. **Identify the equipment scope.** Prefer exact `MACHINE`; use `MachineId` to disambiguate or support grouping when available.
2. **Identify the requested time window.** Apply explicit date/time boundaries. If the user uses a relative period, resolve it using the current date/time and state the resulting range.
3. **Retrieve relevant source evidence.** Do not query only aggregate values if the answer requires event timestamps, explanations, or auditability.
4. **Select appropriate columns.** Include the field groups described above for the intent; retrieve status codes and names together and comments when explanation is required.
5. **Use validated semantics.** Do not assume status, event, PM, repair, comment-type, ignore-record, timestamp, or ordering definitions that are not documented.
6. **Calculate transparently.** Explain formulas, interval boundaries, classification criteria, and any assumptions. Avoid aggregating duplicate or same-incident records incorrectly.
7. **Show supporting evidence.** Answers should include the records, timestamps, status transitions, comments, or intervals supporting the result when available.
8. **Represent uncertainty.** Separate recorded facts, calculated results, observed patterns, interpretations, and information unavailable from this table.
9. **Respect scope.** This history is for understanding what was recorded, when it occurred, how events relate chronologically, and what patterns are present. Do not provide step-by-step repair, maintenance, calibration, adjustment, tuning, rebuilding, or physical troubleshooting instructions.

---

## 4. Schema reference (28 columns)

| Column                  | Type          | Query role / notes                                                                |
| ----------------------- | ------------- | --------------------------------------------------------------------------------- |
| `DATE_TIME_RUN`       | `datetime2` | Timestamp field; exact meaning unconfirmed. Inspect alongside`DATE_TIME`.       |
| `EMPID`               | `varchar`   | Employee identifier; may be useful for record attribution if populated.           |
| `MACHINE`             | `varchar`   | Human-readable machine/equipment identifier; primary equipment filter.            |
| `MACHINE_DESCRIPTION` | `varchar`   | Machine/component descriptive context.                                            |
| `MACHINE_GROUP`       | `varchar`   | Grouping/classification for equipment analysis.                                   |
| `MACHINE_TYPE`        | `varchar`   | Equipment type/family classification.                                             |
| `MACHINE_KIND`        | `varchar`   | Equipment kind/classification.                                                    |
| `DATE_TIME`           | `datetime2` | Timestamp field; candidate event time, semantics to validate.                     |
| `STATUS1_CODE`        | `varchar`   | First status code; interpret only with validated mapping.                         |
| `STATUS1_NAME`        | `varchar`   | First status name; semantics/role to validate.                                    |
| `STATUS2_CODE`        | `varchar`   | Second status code; interpret only with validated mapping.                        |
| `STATUS2_NAME`        | `varchar`   | Second status name; semantics/role to validate.                                   |
| `PM_CODE`             | `varchar`   | PM-related code.                                                                  |
| `PM_NAME`             | `varchar`   | PM-related name/label.                                                            |
| `REPAIR1_CODE`        | `int`       | Repair-related code slot 1.                                                       |
| `REPAIR1_NAME`        | `int`       | Repair-related name slot 1; schema reports`int`, not text.                      |
| `REPAIR2_CODE`        | `int`       | Repair-related code slot 2.                                                       |
| `REPAIR2_NAME`        | `int`       | Repair-related name slot 2; schema reports`int`, not text.                      |
| `REPAIR3_CODE`        | `int`       | Repair-related code slot 3.                                                       |
| `REPAIR3_NAME`        | `int`       | Repair-related name slot 3; schema reports`int`, not text.                      |
| `IGNORE_RECORD`       | `varchar`   | Ignore-related flag/marker; meaning and values unconfirmed.                       |
| `COMMENTS`            | `varchar`   | Free-text event context, symptoms, notes, or recorded explanation.                |
| `USERNAME`            | `varchar`   | Username associated with the record, if present.                                  |
| `COMMENTTYPE`         | `varchar`   | Comment/event type category; value meanings unconfirmed.                          |
| `LINEORDER`           | `int`       | Possible ordering/context field; meaning unconfirmed.                             |
| `MACHINE_PRIORITY`    | `varchar`   | Priority-like field; meaning and numeric interpretation unconfirmed.              |
| `REPAIRCODE`          | `varchar`   | Repair-related code; meaning and relationship to other repair fields unconfirmed. |
| `MachineId`           | `int`       | Numeric machine identifier; use for disambiguation/grouping where appropriate.    |

---

## 5. Known metadata limitations and SME items needed

Before enabling reliable status-based calculations, obtain definitions or examples for:

- Which field(s) are the authoritative event timestamp, and the difference between `DATE_TIME` and `DATE_TIME_RUN`.
- Meaning and transition roles of `STATUS1_CODE/NAME` and `STATUS2_CODE/NAME`; valid code-to-name/status mapping.
- The exact definition of `Down`, `Production Up`, other unavailable states, and qualifying failure/recovery transitions.
- Rules for continuous downtime incident grouping, duplicate/repeated event handling, missing start/end events, and time-window boundaries.
- Scheduled versus unscheduled downtime classification and how it is represented.
- Availability, MTTR, MTBF, and other metric formulas and inclusion/exclusion rules.
- PM code/name definitions, PM start/completion rules, and how PM duration is determined.
- Repair field definitions and relationships among `REPAIR1/2/3_CODE`, corresponding `*_NAME` fields, and `REPAIRCODE`.
- `COMMENTTYPE` value definitions; how `IGNORE_RECORD` should be applied; meaning of `MACHINE_PRIORITY` and `LINEORDER`.
- Rules for defining comparable equipment groups for cross-tool analysis.

Until these are supplied, the agent may retrieve and summarize raw records, compare recorded values, and perform clearly labeled exploratory calculations only where the assumptions are stated. It must not present unconfirmed classifications or calculations as authoritative.

---

## 6. Evidence-oriented response structure

For event-history answers, prefer this structure when applicable:

- **Scope:** equipment identifier and time range.
- **Result:** concise answer or calculated metric.
- **Evidence:** source timestamps, status codes/names, comments, and record context.
- **Calculation:** formula, interval boundaries, and classification assumptions (for calculated results).
- **Interpretation / pattern:** clearly labeled observations, not asserted root cause.
- **Unknowns:** information that cannot be established from the records or supplied definitions.

Keep the response concise for simple lookups and expand into a chronological timeline when the request requires incident reconstruction.
