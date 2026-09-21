# Initial Analysis of dbo.LOTHIST and Request for Domain Clarification

## Subject

Initial Analysis of `dbo.LOTHIST` and Assistance Required to Understand Complete Lot Lifecycle

---

Hi Polar,

I have completed an initial analysis of the `dbo.LOTHIST` table and reviewed a sample lot in detail to better understand how lot history is recorded within the system.
My primary objective is to understand the complete lifecycle of a semiconductor manufacturing lot and determine how the information stored in `dbo.LOTHIST` can be used to reconstruct that lifecycle accurately.
As part of my analysis, I selected a lot with multiple transactions and traced its history chronologically.

---

# Lot Selected for Analysis

```text
LOT = T2720W2670
```

The lot was analyzed using:

```sql
SELECT *
FROM dbo.LOTHIST
WHERE LOT = 'T2720W2670'
ORDER BY DATE_TIME;
```

---

# Overall Understanding So Far

Based on the data reviewed, my current understanding is that `dbo.LOTHIST` functions as a historical audit trail for manufacturing lots.
Instead of storing the current state of a lot, it stores all events that occur throughout the lot's lifecycle.
The table appears to record:

- Lot creation
- Product assignments
- Route assignments
- Status changes
- Transfers
- Operation movements
- Quality activities
- Lot attribute changes
- User/operator actions
- Miscellaneous system-generated events
I also observed that a single manufacturing transaction typically creates multiple rows within the table.Rows sharing the same timestamp usually appear to belong to the same transaction/event.
The business meaning of each record seems to be derived from the combination of:

```text
COMMAND
HISTCODE
COMMENT
OPER
OPERDESC
```

rather than from any individual column alone.

---

# Understanding of Table Columns

The following interpretations are based only on observed behavior from the sample lot and should be validated.

---

## LOT

### Observed Understanding

Unique identifier of the manufacturing lot.

Example:

```text
T2720W2670
```

Every record in the sample belongs to the same lot.

---

## DATE_TIME

### Observed Understanding

Timestamp when the transaction was recorded.

Example:

```text
2026-08-18 13:23:05.210
```

Records having the same timestamp appear to belong to the same logical event.

---

## COMMAND

### Observed Understanding

Represents the action performed against the lot.

Observed values:

```text
TWCR
TWKT
TRNF
```

Possible interpretations:

| Command | Assumed Meaning |
|----------|----------------|
| TWCR | Lot Creation / Traveler Creation |
| TWKT | Tracking Transaction |
| TRNF | Transfer Transaction |

These are assumptions and require confirmation.

---

## OPER

### Observed Understanding

Represents an operation or manufacturing process step.

Examples observed:

```text
90100
94941
71000
99999
```

These appear to correspond to process stages within the manufacturing route.

---

## OPERDESC

### Observed Understanding

Human-readable description of the operation.

Examples:

```text
SCRIBE
TW_ROOM
QUAL_CHECK(1)
WET_DIP_E
```

This appears to be the textual description associated with the OPER number.

---

## MACHINE

### Observed Understanding

Likely represents equipment or machine identifiers.

This field was mostly empty in the sample reviewed.

Further clarification is needed regarding:

- When it is populated
- Whether it represents actual equipment
- Whether it references another lookup table

---

## EMPID

### Observed Understanding

Employee identifier of the person initiating the transaction.

Examples:

```text
04518
03414
05234
05852
```

Appears to correspond to personnel IDs.

---

## USERNAME

### Observed Understanding

System user who performed the action.

Examples:

```text
JOHNSONR
VANGA
ALALI
DEMISSIEB
```

Appears to correspond to the operator login.

---

## HISTCODE

### Observed Understanding

This appears to be one of the most important columns in the table.

Based on the sample analyzed, each HISTCODE seems to identify the type of information stored in the row.

Observed values:

```text
CM
CR
PT
RE
WF
KL
RT
DK
EM
EN
```

However, the exact business meaning of each code is currently unknown.

---

## HISTORDER

### Observed Understanding

Appears to determine sequencing within a transaction.

Example:

```text
77
78
79
80
```

It seems to control display or processing order of records generated during the same event.

Unclear whether it contains additional business meaning.

---

## COMMENT

### Observed Understanding

Contains the actual business information associated with the history record.

Examples:

```text
Fab Area: ALL
New Status: INV
METAL_TEST
M_STARTUP_PARTICLE_5
OLD: T_SCRIBE61300000
NEW: T_M_8STACK
```

Many important business events appear to be documented through COMMENT values.

---

## MASK_LVL

### Observed Understanding

Observed values:

```text
777
888
999
```

Potential interpretations:

- Mask level
- Process layer
- Manufacturing stage
- Technology layer

Meaning requires confirmation.

---

## IS_DUPLICATE

### Observed Understanding

Appears to indicate duplicate history records.

Observed value:

```text
0
```

Need confirmation on the meaning of:

```text
0
1
```

---

## LotHistId

### Observed Understanding

Unique identifier for each history row.

Example:

```text
1374089198
```

Appears to be the primary key.

---

## UTCDate_Time

### Observed Understanding

Timezone-aware timestamp.

Example:

```text
2026-08-22 01:10:08.430 -05:00
```

Used to preserve timezone information.

---

# Reconstructed Lifecycle of Lot T2720W2670

The information below represents my interpretation of the lot history.

---

## Event 1 – Lot Creation

Timestamp:

```text
2026-08-11 07:23:48
```

Operation:

```text
90100
SCRIBE
```

Key History Records:

```text
CR = 1 T2720W2670
PT = T_SCRIBE61300000
CM = Fab Area: ALL
RE = RJ
WF = 1 61300000 Rev: C
```

My interpretation:

The lot appears to be initially created or released into operation SCRIBE.

---

## Event 2 – Additional Product/Route Information

Timestamp:

```text
2026-08-11 07:23:49
```

History Records:

```text
KL = 5582T8469 20T_SCRIBE61300000
RT = 8SCRB_3000
```

My interpretation:

Additional routing, product, recipe, mask, traveler, or process information is attached to the lot.

Need clarification on what these values actually represent.

---

## Event 3 – Inventory Transfer

Timestamp:

```text
2026-08-18 13:22:04
```

History Records:

```text
New Status: INV
```

My interpretation:

Lot status changes into Inventory.

Need confirmation of exact meaning of INV.

---

## Event 4 – Qualification Activity

Timestamp:

```text
2026-08-18 13:23:05
```

Operation:

```text
94941
QUAL_CHECK(1)
```

Observed Attribute Additions:

```text
MACHINE = METAL_TEST
EVENT = M_STARTUP_PARTICLE_5
```

System Messages:

```text
Lot attr MACHINE added
Lot attr EVENT added
```

My interpretation:

The lot is being tagged or classified for a qualification-related activity.

Need clarification on:

- METAL_TEST
- M_STARTUP_PARTICLE_5
- How lot attributes are managed

---

## Event 5 – Status and Product Change

Timestamp:

```text
2026-08-22 00:06:27
```

Observed:

```text
New Status: USDEVNT

OLD: T_SCRIBE61300000
NEW: T_M_8STACK
```

My interpretation:

The lot is reassigned to a different product or process flow.

Need clarification regarding:

- USDEVNT
- Product transition rules
- Whether PT represents product, process flow, or traveler

---

## Event 6 – Entry into WET_DIP_E

Timestamp:

```text
2026-08-22 01:10:08
```

Operation:

```text
71000
WET_DIP_E
```

History Records:

```text
KL = 6530TH051 20T_M_8STACK
RT = T_M_8C_STK
```

My interpretation:

The lot proceeds into the next manufacturing step under the reassigned product flow.

Need clarification regarding the significance of KL and RT values.

---

# Current Assumptions Regarding HISTCODE Values

These are assumptions only and should not be considered authoritative.

| HISTCODE | Current Assumption |
|----------|-------------------|
| CM | Comment Message |
| CR | Creation Record |
| PT | Product Type / Process Type |
| RE | Recipe Revision |
| WF | Workflow Definition |
| KL | Product/Route Key Information |
| RT | Route Type |
| DK | Device/Product Key |
| EM | MACHINE Attribute |
| EN | EVENT Attribute |

---

# Critical Questions Required to Fully Understand the Dataset

## 1. Lot Lifecycle

My primary goal is to understand the complete lifecycle of a lot.

Questions:

- What event officially creates a lot?
- How is lot movement represented?
- How can we identify:
  - Lot creation
  - Queueing
  - Processing
  - Transfer
  - Hold
  - Release
  - Rework
  - Product change
  - Route change
  - Completion
  - Scrap
  - Cancellation

- Which fields should be used to reconstruct the complete lot lifecycle?

---

## 2. COMMAND Definitions

Can we obtain the official definitions of all COMMAND values?

Examples observed:

```text
TWCR
TWKT
TRNF
```

Questions:

- What does each command mean?
- Which commands are system-generated?
- Which commands are operator-generated?
- Is there a master lookup table?

---

## 3. HISTCODE Definitions

Can we obtain complete documentation for HISTCODE values?

Questions:

- What is the exact meaning of every HISTCODE?
- Is there a lookup table available?
- Are HISTCODE definitions stored in dbo.HISTCODES?
- Which HISTCODEs indicate:
  - Status changes
  - Product changes
  - Route changes
  - Attribute changes
  - Recipe changes
  - Equipment changes

---

## 4. HISTORDER

Questions:

- What exactly is HISTORDER?
- Is it merely sequence ordering?
- Does it contain business meaning?
- Is it transaction-specific?

---

## 5. OPERATION Data

Questions:

- Is OPER the route step number?
- Is there a route master table?
- Can operations repeat?
- Can a lot revisit an operation?
- How can route progression be traced?

---

## 6. MASK_LVL

Questions:

- What does MASK_LVL represent?
- Why do values such as 777, 888, and 999 occur?
- Is there a lookup/reference table?

---

## 7. Product and Route Information

Questions:

- What do values like the following represent?

```text
T_SCRIBE61300000
T_M_8STACK
T_M_8C_STK
8SCRB_3000
```

- Product?
- Route?
- Traveler?
- Recipe?
- Reticle?
- Process family?

---

## 8. Lot Attributes

Questions:

- Where are lot attributes defined?
- Is there a dedicated lot attribute table?
- How can all possible lot attributes be identified?
- What is the lifecycle of lot attributes?
- How can attribute updates be tracked?

---

## 9. Status Codes

Observed values:

```text
INV
USDEVNT
```

Questions:

- What are all possible statuses?
- What do they mean?
- Is there a status master table?

---

## 10. dbo.HISTCODES

Questions:

- Does this table provide formal definitions of HISTCODE values?
- Can it be used as the primary reference source for interpretation?

---

## 11. dbo.LOTHISTV

Questions:

- What additional logic exists in LOTHISTV?
- Is LOTHISTV the preferred business view?
- Which transformations are applied relative to LOTHIST?

---

# Desired Outcome

The goal of this effort is to build a complete business understanding of lot processing and produce a comprehensive data dictionary describing:

- Every column in LOTHIST
- Every COMMAND value
- Every HISTCODE value
- Lot lifecycle stages
- Product transitions
- Route transitions
- Status changes
- Attribute management
- Operational flow through the manufacturing process

This will enable accurate reporting, analysis, and future development work based on lot history data.

Thank you for your time and assistance. Any documentation, lookup tables, process flow diagrams, reference material, or SME guidance would be greatly appreciated.

Best Regards,

**Neelmani**
