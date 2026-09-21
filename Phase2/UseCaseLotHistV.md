# LotHistV Copilot Agent - High Value Customer Use Cases

## Overview

The `TrainingVision.LotHistV` view contains semiconductor manufacturing lot history and event tracking information. The Copilot Agent should translate natural language questions into optimized SQL queries and return business-friendly explanations.

---

# Use Case 1

## Lot History Investigation

### Question

Show complete history of lot 6514A4L8

### Query

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE LOT = '6514A4L8'
AND IS_DUPLICATE = 0
ORDER BY DATE_TIME, HISTORDER;
```

### Purpose

Used by Manufacturing Engineers and Production Supervisors to understand the complete lifecycle of a lot.

---

# Use Case 2

## Latest Status of a Lot

### Question

What is the current status of lot 7412A0A7?

### Query

```sql
SELECT TOP 1 *
FROM TrainingVision.LotHistV
WHERE LOT = '7412A0A7'
AND IS_DUPLICATE = 0
ORDER BY DATE_TIME DESC, HISTORDER DESC;
```

### Purpose

Used to quickly determine the most recent activity recorded for a lot.

---

# Use Case 3

## Current Lot Location

### Question

Where is lot 6905A002 currently located?

### Query

```sql
SELECT TOP 1
       LOT,
       DATE_TIME,
       MACHINE,
       HIST_REC
FROM TrainingVision.LotHistV
WHERE LOT = '6905A002'
  AND TRANS = 'LOCATION'
ORDER BY DATE_TIME DESC;
```

### Purpose

Allows operators to locate a lot in WIP, machine queue, shelf location, or processing equipment.

---

# Use Case 4

## Lot Traceability

### Question

Show all machine movements for lot 6514A4L8.

### Query

```sql
SELECT DATE_TIME,
       TRANS,
       MACHINE,
       USERNAME,
       OPERDESC
FROM TrainingVision.LotHistV
WHERE LOT = '6514A4L8'
  AND TRANS IN
      (
          'MOVE IN',
          'MOVE OUT',
          'TRANSITION',
          'ARRIVAL'
      )
ORDER BY DATE_TIME;
```

### Purpose

Provides a complete movement trail through the manufacturing process.

---

# Use Case 5

## Why Is My Lot Waiting?

### Question

Why is lot 7407A043 waiting?

### Query

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE LOT = '7407A043'
ORDER BY DATE_TIME DESC,
         HISTORDER DESC;
```

### Purpose

Determines whether a lot is waiting in queue, waiting for operator action, or blocked by equipment.

---

# Use Case 6

## Operator Activity Analysis

### Question

Which operators moved the most lots today?

### Query

```sql
SELECT USERNAME,
       COUNT(*) AS MoveCount
FROM TrainingVision*LotHistV
WHERE TRANS IN ('MOVE IN'* 'MOVE OUT')
  AND CAST*DATE_TIME AS DATE) = CAST(GETDATE(* AS DATE)
GROUP BY USERNAME
ORDER *Y MoveCount DESC;
```

*##*Purpose

Measures operator product*vity and workload.

---

# Use Cas* 7

## Equipment Utilization

### *uestion

Which machines processed *he highest number of lots today?

*## Query

```sql
SELECT MACHINE,
 *     COUNT(DISTINCT LOT) AS LotsPr*cessed
FROM TrainingVision.LotHist*
WHERE MACHINE IS NOT NULL
GROUP B* MACHINE
ORDER BY LotsProcessed DE*C;
```

### Purpose

Identifies he*vily utilized equipment.

---

# U*e Case 8

## Dispatch Monitoring

*## Question

How many lots were di*patched in the last hour?

### Que*y

```sql
SELECT COUNT(*) AS Dispa*chCount
FROM TrainingVision.LotHis*V
WHERE TRANS = 'DISPATCH'
  AND D*TE_TIME >= DATEADD(HOUR, -1, GETDA*E());
```

### Purpose

Tracks man*facturing flow and dispatch effici*ncy.

---

# Use Case 9

## Machin*-Specific Tracking

### Question

*how all lots processed on WB305 to*ay.

### Query

```sql
SELECT DIST*NCT
       LOT
FROM TrainingVision*LotHistV
WHERE MACHINE = 'WB305'
 *AND CAST(DATE_TIME AS DATE) = CAST*GETDATE() AS DATE);
```

### Purpo*e

Provides machine-level producti*n visibility.

---

# Use Case 10
*## Queue Monitoring

### Question
*Which lots are currently in queue?*
### Query

```sql
SELECT LOT,
   *   MAX(DATE_TIME) AS LastEventTime*FROM TrainingVision.LotHistV
WHERE*HIST_REC LIKE '%InQueue%'
GROUP BY*LOT;
```

### Purpose

Allows supe*visors*to identify bottlenecks.

---

# U*e Case 11

## Automated vs*Manual Actions

### Question

How *uch activity was automated versus *anual?

### Query

```sql
SELECT
 *  CASE
        WHEN USERNAME = 'SY*TEM'
        THEN*'Automated'
        ELSE 'Manual'
*   END AS ActivityType,
   *COUNT(*) AS TotalEvents
FROM Train*ngVision.LotHistV
GROUP BY
    CAS*
        WHEN USERNAME = 'SYSTEM'
*       THEN 'Automated'
        EL*E 'Manual'
    END;
```

*## Purpose

Measures automation ef*iciency.

---

* Use Case 12

## Lot Split / Dekit*Tracking

### Question

Show all*dekit activities during the select*d period.

### Query

```sql
SELEC* *
FROM TrainingVision.LotHistV
WH*RE TRANS = 'LOT DEKIT'
ORDER BY DA*E_TIME DESC;
```

*## Purpose

Tracks lot splitting a*d wafer relationship history.

---*
* Use Case 13

## Employee Activity*Investigation

### Question

Show *ll activity performed by employee *5880.

### Query

```sql
SELECT *
*ROM TrainingVision.LotHistV
WHERE *MPID = '05880'
ORDER BY DATE_TIME *ESC;
```

### Purpose

Useful for*audits and*operator activity reviews.

---

#*Use Case 14

## Machine Event Hist*ry

### Question

Show everything*that happened on machine NOVEL315 *oday.

### Query

```sql
SELECT *
*ROM TrainingVision.LotHistV
WHERE *ACHINE = 'NOVEL315'
  AND CAST*DATE_TIME AS DATE) = CAST(GETDATE(* AS DATE)
ORDER BY DATE_TIME DESC;*```

### Purpose

Provides complet* machine history for troubleshooti*g.

---

# Advanced Analytics Use *ases

---

# Use Case 15

## Lots *tuck in Queue for More Than X Hour*

### Question

Which lots have be*n waiting for more than 2 hours?

*## Query

```sql
WITH LatestEvent *S
(
    SELECT LOT,
           MAX*DATE_TIME) AS LastEventTime*    FROM TrainingVision.LotHistV
 *  GROUP BY LOT
)
SELECT *
FROM Lat*stEvent
WHERE D*TEDIFF(HOUR,
               LastEv*ntTime,
               GET*ATE()) > 2;
```

### Purpose

Iden*ifies aging*WIP and potential bott*enecks.

---

# Use Case 16

## Pr*cess Bottleneck Analysis

### Ques*ion

Which operation*has the most waiting lots?

### Qu*ry

```sql
SELECT OPERDESC,
      *COUNT(DISTINCT LOT) AS WaitingLots*FROM TrainingVision.LotHistV
WHERE*HIST_REC LIKE '%InQueue%'
GROUP BY*OPERDESC
ORDER BY WaitingLots DESC*
```

### Purpose

Finds*bottleneck operations across the f*ctory.

---

# Use Case 17

## Equ*pment Loading Analysis

### Questi*n

Which machines currently have t*e most active lots?

### Query

``*sql
SELECT MACHINE,
       COUNT(D*STINCT LOT) AS ActiveLots
FROM*TrainingVision.LotHistV
WHERE MACH*NE IS NOT NULL
GROUP BY MACHINE
OR*ER BY ActiveLots DESC;
```

### Pu*pose

Detects overloaded tools.

-*-

# Use Case 18

## Cycle Time An*lysis

### Question

What*is the*average time between Move In and M*ve Out events by operation?

### Q*ery

```sql
-- Conceptual Query

S*LECT OPERDESC,
       AVG*DATEDIFF(MINUTE,
                 *  MoveInTime,
                    *oveOutTime))
       AS AvgProcessM*nutes
FROM ProcessEvents
GROUP BY *PERDESC;
```

*## Purpose

Measures process effic*ency and cycle time.

---

# Use C*se 19

## Rework Detection

### Qu*stion

Which lots visited the same*operation multiple times?

### Que*y

```sql
SELECT LOT,
       OPERD*SC,
       COUNT(*) AS VisitCount
*ROM TrainingVision.LotHistV
GROUP *Y LOT,
         OPERDESC
HAVING CO*NT(*) > 1
ORDER BY VisitCount DESC*
```

### Purpose

Detects*potential rework loops.

---

# Us* Case 20

## Production Throughput*
### Question

How many*lots completed each operation toda*?

### Query

```sql
SELECT OPERDE*C,
       COUNT(DISTINCT LOT) AS L*tsCompleted
FROM TrainingVision.Lo*HistV
WHERE TRANS*IN ('MOVE OUT','LOTCOMPLET')
  AND*CAST(DATE_TIME*AS DATE) =
      CAST(GETDATE() AS*DATE)
GROUP BY OPERDESC
ORDER BY L*tsCompleted DESC;
```

### Purpose*
Measures production throughput.

*--

# Use Case 21

## Shift Compar*son

### Question

Compare product*on activity between day shift and *ight shift.

### Query

```sql
SEL*CT
    CASE
        WHEN DATE*ART(HOUR, DATE_TIME)
            *BETWEEN 6 AND 17
        THEN*'Day*Shift'
        ELSE 'Night Shift'
*   END AS ShiftName,
    COUNT(*) *S EventCount
FROM Training*ision.LotHistV
GROUP BY
    CASE
 *      WHEN DATE*ART(HOUR, DATE_TIME)
             *ETWEEN 6 AND 17
        THEN 'Day *hift'
        ELSE 'Night Shift'
 *  END;
```

### Purpose

Compares *orkload and productivity by shift.*
---

# Use Case 22

## W*P Trend Analysis

### Question

Ho* has*queue volume changed over the last*24 hours?

### Query

```sql
SELEC*
    DATEADD(HOUR,
           *DATED*FF(HOUR,0,DATE_TIME),
            *) AS HourBucket,
    COUNT**) AS QueueEvents
FROM TrainingVision.LotHistV
*HERE HIST_REC LIKE '%InQueue%'
  A*D DATE_TIME >= DATEADD(HOUR,-24,GE*DATE())
GROUP BY*DATEADD(HOUR,
                 D*TEDIFF(HOUR,0,DATE_TIME*,
                 0)
ORDER BY Hou*Bucket;
```

### Purpose

Provides*queue trend analytics.

---

* Business Questions Customers Can *ctually Ask

- Where is my lot rig*t now?
- What*happened to lot 6514A4L8?
- Show*complete history of lot XXXXX*
- Who moved lot XXXXX?
- What*machine*processed this lot?
- Why is my lo* waiting?
- Which*lots are currently in queue?
- Whi*h*lots were dispatched today?
- Whic**lots were moved today?
- Show all *ctivity on machine WB305.
- Show a*l activity on operator MJAMA.
- Wh*ch operator processed the most lot* today?
- Which*machine processed the most lots to*ay?
- Which operation currently ha* the longest queue?
- Which lots a*e stuck?
- Which lots have not mov*d in the last 4 hours?
- Show*all automated moves.
- Show*all manual moves.
- Which*lots were split or dekitted*
- Show*equipment utilization across the f*b.
- Which*lots revisited the same operation?*- What is the average cycle time f*r IMPLANT?
- What*is the throughput by operation?
- *ompare*Day Shift versus Night Shift produ*tivity.
- Which machines are overl*aded?
- Which lots*are waiting for Move Out?
- Which*lots were moved by*employee 05880?
- Show*all transitions performed by TGADO*E.
- Show*all checksum verification activiti*s.
- Which*machine*generated the most alarms/comments*

---

# Gold Standard Test Questi*n

## User Question

Where is*lot 6905A002 and what happened to *t most recently?

## Generated SQL*
```sql
SELECT TOP 10
       LOT,
*      DATE_TIME,
       TRANS,
   *   MACHINE,
       OPERDESC,
     * HIST_REC
FROM TrainingVision.LotH*stV
WHERE LOT = '6905A002'
  AND I*_DUPLICATE = 0
ORDER BY DATE_TIME *ESC,
         HISTORDER DESC;
*``

## Dummy Result

```text*Lot: 6905A002

Current Location:
S*elf 3319-F

Current Area:
*2_WIP

Current Status*
In Queue

Latest Event:
LOCATION
*Operation:
METR_FILMS

Timestamp*
2026*08-10 04:59:32

Interpretation:
Th* lot is currently in WIP Area 32_W*P,
stored on*Shelf 3319-F and waiting for the n*xt
processing step. No newer*movement or processing
events have*been recorded after this location *pdate.
```

*--

# Recommended Copilot Agent Te*ting Parameters

## Entities

- LO*
- DATE_TIME
- TRANS
- MACHINE
- O*ER
- OPERDESC
- USERNAME
- EMPID
-*COMMAND
* HISTCODE
- HIST_REC
- SHORTREPORT*- IS_DUPLICATE

##*Default Filters

```*ql
IS_DUPLICATE = 0
```

*# Time Interpretation*Rules

- today
- yesterday
-*last hour
- last 24 hours
- this*shift*- previous shift
- this*week*- last*7 days
- custom date range

## Bus*ness Synonym Mapping

| User Says * Map To |
|------------|---------|*| lot | LOT |
| wafer | LOT |
| ba*ch | LOT |
| tool | MACHINE |
| eq*ipment | MACHINE |
| machine | MAC*INE |
| operator | USERNAME |
| em*loyee | EMPID |
| process | OPERDE*C |
| step | OPERDESC |
| action |*TRANS |
| event*| TRANS/HIST_REC |
| location*| TRANS='LOCATION' |
| moved*| MOVE*IN / MOVE OUT |
| dispatched | DIS*ATCH |
| waiting | InQueue / WAIT *VOU |
|*history |*All records*ordered by DATE_TIME,HISTORDER |

*```*
