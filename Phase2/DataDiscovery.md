# Polar Semiconductor MES Data Analysis
## Understanding `dbo.LOTHIST`, `dbo.HISTCODES`, and Lot History Interpretation

---

# Objective

Analyze the semiconductor manufacturing history table `dbo.LOTHIST` for Polar Semiconductor and understand:

1. The purpose of each column.
2. The meaning of transaction records.
3. The lifecycle of a lot through the manufacturing process.
4. The likely meaning of `HISTCODE` values.
5. How `dbo.HISTCODES` can be used to fully decode the history table.

---

# Data Investigation Performed

The following query was executed to identify lots and understand transaction history density:

```sql
SELECT LOT, COUNT(LOT) AS LotCount
FROM dbo.LOTHIST
GROUP BY LOT
ORDER BY COUNT(LOT);
```

A single lot was selected for detailed investigation:

```sql
SELECT *
FROM dbo.LOTHIST
WHERE LOT = 'T2720W2670'
ORDER BY DATE_TIME;
```

---

# Sample Lot History Data

Lot:

```text
T2720W2670
```

Returned records:

```text
LOT DATE_TIME COMMAND OPER MACHINE EMPID HISTCODE HISTORDER COMMENT MASK_LVL OPERDESC USERNAME IS_DUPLICATE LotHistId UTCDate_Time

T2720W2670 2026-08-11 07:23:48.820 TWCR 90100      04518 CM 79  Fab Area: ALL                 777 SCRIBE        JOHNSONR 0 1374089198 2026-08-11 07:23:48.820 -05:00
T2720W2670 2026-08-11 07:23:48.820 TWCR 90100      04518 CR 77    1 T2720W2670               777 SCRIBE        JOHNSONR 0 1374089196 2026-08-11 07:23:48.820 -05:00
T2720W2670 2026-08-11 07:23:48.820 TWCR 90100      04518 PT 78       T_SCRIBE61300000         777 SCRIBE        JOHNSONR 0 1374089197 2026-08-11 07:23:48.820 -05:00
T2720W2670 2026-08-11 07:23:48.820 TWCR 90100      04518 RE 80 RJ                            777 SCRIBE        JOHNSONR 0 1374089199 2026-08-11 07:23:48.820 -05:00
T2720W2670 2026-08-11 07:23:48.820 TWCR 90100      04518 WF 120   1 61300000 Rev: C         777 SCRIBE        JOHNSONR 0 1374089239 2026-08-11 07:23:48.820 -05:00

T2720W2670 2026-08-11 07:23:49.540 TWKT 90100      04518 KL 51 5582T8469 20T_SCRIBE61300000 777 SCRIBE        JOHNSONR 0 1374089298 2026-08-11 07:23:49.540 -05:00
T2720W2670 2026-08-11 07:23:49.540 TWKT 90100      04518 RT 52 8SCRB_3000                   777 SCRIBE        JOHNSONR 0 1374089299 2026-08-11 07:23:49.540 -05:00

T2720W2670 2026-08-18 13:22:04.680 TRNF 99999      03414 CM 46 New Status: INV              777 TW_ROOM       VANGA    0 1376373787 2026-08-18 13:22:04.680 -05:00
T2720W2670 2026-08-18 13:22:04.680 TRNF 99999      03414 DK 21                5582T8469    777 TW_ROOM       VANGA    0 1376373762 2026-08-18 13:22:04.680 -05:00

T2720W2670 2026-08-18 13:23:05.210 TWKT -100       03414 CM 217 Lot attr MACHINE added; value
T2720W2670 2026-08-18 13:23:05.210 TWKT -100       03414 CM 218 set to METAL_TEST.
T2720W2670 2026-08-18 13:23:05.210 TWKT -100       03414 CM 219 Lot attr EVENT added; value
T2720W2670 2026-08-18 13:23:05.210 TWKT -100       03414 CM 220 set to M_STARTUP_PARTICLE_5.

T2720W2670 2026-08-18 13:23:05.210 TWKT 94941      03414 EM 90 METAL_TEST                   999 QUAL_CHECK(1)  VANGA    0 1376374022
T2720W2670 2026-08-18 13:23:05.210 TWKT 94941      03414 EN 91 M_STARTUP_PARTICLE_5         999 QUAL_CHECK(1)  VANGA    0 1376374023
T2720W2670 2026-08-18 13:23:05.210 TWKT 94941      03414 KL 88 6530TH020 6T_SCRIBE61300000 999 QUAL_CHECK(1)  VANGA    0 1376374020
T2720W2670 2026-08-18 13:23:05.210 TWKT 94941      03414 RT 89 T_M_STRTUP                   999 QUAL_CHECK(1)  VANGA    0 1376374021

T2720W2670 2026-08-22 00:06:27.220 TRNF 99999      05852 CM 86 New Status: USDEVNT          999 TW_ROOM       ALALI
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 203 Lot attr MACHINE status set
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 204 to Inactive.
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 205 Lot attr EVENT status set to
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 206 Inactive.

T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 307 Lot attr EVENT added; value
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 308 set to M_STARTUP_PARTICLE_5.
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 309 Lot attr MACHINE added; value
T2720W2670 2026-08-22 00:06:27.220 TRNF -100       05852 CM 310 set to METAL_TEST.

T2720W2670 2026-08-22 00:06:27.220 TRNF 99999      05852 DK 21                6530TH020
T2720W2670 2026-08-22 00:06:27.220 TRNF 99999      05852 PT 84 NEW: T_M_8STACK
T2720W2670 2026-08-22 00:06:27.220 TRNF 99999      05852 PT 85 OLD: T_SCRIBE61300000

T2720W2670 2026-08-22 01:10:08.430 TWKT 71000      05234 KL 50 6530TH051 20T_M_8STACK       888 WET_DIP_E DEMISSIEB
T2720W2670 2026-08-22 01:10:08.430 TWKT 71000      05234 RT 51 T_M_8C_STK                   888 WET_DIP_E DEMISSIEB
```

---

# Understanding the Table Structure

## Key Observation

`dbo.LOTHIST` is not a current-state table.

It is an audit/history table that records every event performed on a lot during its lifecycle.

A single manufacturing action often creates multiple history records.

The meaning of a record is determined primarily by:

- COMMAND
- HISTCODE
- COMMENT
- HISTORDER

---

# Column-by-Column Analysis

---

## LOT

### Purpose

Unique semiconductor lot identifier.

### Example

```text
T2720W2670
```

### Meaning

All rows belong to this manufacturing lot.

---

## DATE_TIME

### Purpose

Timestamp of the transaction.

### Example

```text
2026-08-18 13:23:05.210
```

### Meaning

Rows sharing the same timestamp generally belong to the same MES transaction.

---

## COMMAND

### Purpose

Indicates what action occurred.

### Examples

```text
TWCR
TWKT
TRNF
```

### Likely Meaning

| COMMAND | Possible Meaning |
|----------|------------------|
| TWCR | Create Traveler / Create Lot |
| TWKT | Track Transaction / Work Transaction |
| TRNF | Transfer Lot |
| HOLD | Possible Hold Transaction |
| SPLT | Possible Split Transaction |
| MRGE | Possible Merge Transaction |

### Validation Query

```sql
SELECT COMMAND,
       COUNT(*)
FROM dbo.LOTHIST
GROUP BY COMMAND
ORDER BY COUNT(*) DESC;
```

---

## OPER

### Purpose

Manufacturing operation number.

### Examples

```text
90100
94941
71000
```

### Interpretation

Represents process route steps.

Examples:

```text
90100 = SCRIBE
94941 = QUAL_CHECK(1)
71000 = WET_DIP_E
```

Evidence:

```text
OPER      OPERDESC
90100     SCRIBE
94941     QUAL_CHECK(1)
71000     WET_DIP_E
```

---

## MACHINE

### Purpose

Likely equipment/tool identifier.

### Example

Mostly blank in sample.

### Possible Values

```text
ETCH01
TESTER05
WET_BENCH_3
```

### Validation Query

```sql
SELECT DISTINCT MACHINE
FROM dbo.LOTHIST
WHERE MACHINE IS NOT NULL;
```

---

## EMPID

### Purpose

Operator employee ID.

### Examples

```text
04518
03414
05852
05234
```

### Notes

Represents numeric employee identity.

---

## HISTCODE

### Purpose

Most important field in the table.

Defines what type of history record is being stored.

### Examples

```text
CM
CR
PT
RE
WF
KL
RT
EM
EN
DK
```

### Interpretation

Each HISTCODE tells how to interpret COMMENT.

---

## HISTORDER

### Purpose

Display sequence inside a transaction.

### Example

```text
77
78
79
80
```

### Meaning

Controls the order in which records appear for the same timestamp.

---

## COMMENT

### Purpose

Holds the actual event value.

### Examples

```text
Fab Area: ALL
New Status: INV
METAL_TEST
M_STARTUP_PARTICLE_5
```

### Notes

This field contains most business information.

---

## MASK_LVL

### Purpose

Likely semiconductor layer or mask level information.

### Examples

```text
777
888
999
```

### Interpretation

May represent:

- Mask level
- Process layer
- Manufacturing stage

Exact meaning requires process documentation.

---

## OPERDESC

### Purpose

Human-readable operation description.

### Examples

```text
SCRIBE
QUAL_CHECK(1)
TW_ROOM
WET_DIP_E
```

---

## USERNAME

### Purpose

User login who performed the transaction.

### Examples

```text
JOHNSONR
VANGA
ALALI
DEMISSIEB
```

---

## IS_DUPLICATE

### Purpose

Duplicate-row indicator.

### Values

```text
0 = Normal
1 = Duplicate
```

Sample only contains:

```text
0
```

---

## LotHistId

### Purpose

Unique primary key.

### Example

```text
1374089198
```

---

## UTCDate_Time

### Purpose

Timezone-aware timestamp.

### Example

```text
2026-08-22 01:10:08.430 -05:00
```

### Benefit

Supports cross-site reporting and timezone consistency.

---

# Reconstructed Life Cycle of Lot T2720W2670

---

## Event 1 – Lot Creation

### Timestamp

```text
2026-08-11 07:23:48
```

### Command

```text
TWCR
```

### Operation

```text
90100
SCRIBE
```

### Associated Records

```text
CR = 1 T2720W2670
PT = T_SCRIBE61300000
CM = Fab Area: ALL
RE = RJ
WF = 1 61300000 Rev: C
```

### Interpretation

The lot appears to be created or initialized in the SCRIBE operation.

---

## Event 2 – Additional Routing Information

### Timestamp

```text
2026-08-11 07:23:49
```

### Command

```text
TWKT
```

### Associated Records

```text
KL = 5582T8469 20T_SCRIBE61300000
RT = 8SCRB_3000
```

### Interpretation

Product, routing, mask, or recipe-related information was attached to the lot.

---

## Event 3 – Transfer to Inventory Status

### Timestamp

```text
2026-08-18 13:22:04
```

### Command

```text
TRNF
```

### Comment

```text
New Status: INV
```

### Interpretation

Lot transferred to Inventory status.

---

## Event 4 – Quality Check Classification

### Timestamp

```text
2026-08-18 13:23:05
```

### Operation

```text
94941
QUAL_CHECK(1)
```

### Added Attributes

```text
MACHINE = METAL_TEST
EVENT = M_STARTUP_PARTICLE_5
```

### Evidence

```text
EM = METAL_TEST
EN = M_STARTUP_PARTICLE_5
```

### Interpretation

Lot was flagged for qualification/testing activities related to startup particle monitoring.

---

## Event 5 – Status and Product Change

### Timestamp

```text
2026-08-22 00:06:27
```

### Status

```text
USDEVNT
```

### Product Changes

```text
OLD: T_SCRIBE61300000
NEW: T_M_8STACK
```

### Interpretation

Lot was reassigned from one process/product route to another.

---

## Event 6 – Entry into WET_DIP_E

### Timestamp

```text
2026-08-22 01:10:08
```

### Operation

```text
71000
WET_DIP_E
```

### Associated Data

```text
KL = 6530TH051 20T_M_8STACK
RT = T_M_8C_STK
```

### Interpretation

Lot entered the WET_DIP_E processing step under the new product route.

---

# Current Best Interpretation of HISTCODE Values

Based solely on transaction behavior observed in the sample data.

| HISTCODE | Likely Meaning |
|-----------|----------------|
| CM | Comment Message |
| CR | Creation Record |
| PT | Product Type / Process Type |
| RE | Revision / Recipe Revision |
| WF | Workflow Definition |
| KL | Product / Route Key Information |
| RT | Route or Recipe Type |
| DK | Device Key / Prior Product Key |
| EM | MACHINE Attribute Value |
| EN | EVENT Attribute Value |

---

# Recommended Next Step

The table `dbo.HISTCODES` likely contains the authoritative definition of every HISTCODE.

Run:

```sql
SELECT *
FROM dbo.HISTCODES
ORDER BY HISTCODE;
```

Also identify all codes in use:

```sql
SELECT DISTINCT HISTCODE
FROM dbo.LOTHIST
ORDER BY HISTCODE;
```

Then join the tables:

```sql
SELECT
    l.HISTCODE,
    h.*
FROM dbo.LOTHIST l
LEFT JOIN dbo.HISTCODES h
       ON l.HISTCODE = h.HISTCODE
WHERE l.LOT = 'T2720W2670';
```

---

# Final Conclusion

`dbo.LOTHIST` functions as an MES audit trail/history table for semiconductor lot processing.

The primary business meaning comes from combinations of:

- COMMAND
- HISTCODE
- COMMENT
- OPER
- OPERDESC

The analyzed lot progressed through:

1. Lot creation (`TWCR`)
2. Route assignment (`TWKT`)
3. Inventory transfer (`TRNF`)
4. Qualification tagging (`QUAL_CHECK`)
5. Product/route reassignment
6. Entry into downstream manufacturing operations (`WET_DIP_E`)

To produce a complete and authoritative data dictionary for `dbo.LOTHIST`, the definitions stored in `dbo.HISTCODES` and the SQL definition of `dbo.LOTHISTV` should be analyzed next.
