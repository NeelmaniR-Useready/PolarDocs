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
FROM TrainingVision.LotHistV
WHERE TRANS IN ('MOVE IN', 'MOVE OUT')
  AND CAST(DATE_TIME AS DATE) = CAST(GETDATE() AS DATE)
GROUP BY USERNAME
ORDER BY MoveCount DESC;
```

### Purpose

Measures operator productivity and workload.

---

# Use Case 7

## Equipment Utilization

### Question

Which machines processed the highest number of lots today?

### Query

```sql
SELECT MACHINE,
       COUNT(DISTINCT LOT) AS LotsProcessed
FROM TrainingVision.LotHistV
WHERE MACHINE IS NOT NULL
  AND CAST(DATE_TIME AS DATE) = CAST(GETDATE() AS DATE)
GROUP BY MACHINE
ORDER BY LotsProcessed DESC;
```

### Purpose

Identifies heavily utilized equipment.

---

# Use Case 8

## Dispatch Monitoring

### Question

How many lots were dispatched in the last hour?

### Query

```sql
SELECT COUNT(*) AS DispatchCount
FROM TrainingVision.LotHistV
WHERE TRANS = 'DISPATCH'
  AND DATE_TIME >= DATEADD(HOUR, -1, GETDATE());
```

### Purpose

Tracks manufacturing flow and dispatch efficiency.

---

# Use Case 9

## Machine-Specific Tracking

### Question

Show all lots processed on WB305 today.

### Query

```sql
SELECT DISTINCT
       LOT
FROM TrainingVision.LotHistV
WHERE MACHINE = 'WB305'
  AND CAST(DATE_TIME AS DATE) = CAST(GETDATE() AS DATE);
```

### Purpose

Provides machine-level production visibility.

---

# Use Case 10

## Queue Monitoring

### Question

Which lots are currently in queue?

### Query

```sql
SELECT LOT,
       MAX(DATE_TIME) AS LastEventTime
FROM TrainingVision.LotHistV
WHERE HIST_REC LIKE '%InQueue%'
GROUP BY LOT;
```

### Purpose

Allows supervisors to identify bottlenecks.

---

# Use Case 11

## Automated vs Manual Actions

### Question

How much activity was automated versus manual?

### Query

```sql
SELECT
    CASE
        WHEN USERNAME = 'SYSTEM'
        THEN 'Automated'
        ELSE 'Manual'
    END AS ActivityType,
    COUNT(*) AS TotalEvents
FROM TrainingVision.LotHistV
GROUP BY
    CASE
        WHEN USERNAME = 'SYSTEM'
        THEN 'Automated'
        ELSE 'Manual'
    END;
```

### Purpose

Measures automation efficiency.

---

# Use Case 12

## Lot Split / Dekit Tracking

### Question

Show all dekit activities during the selected period.

### Query

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE TRANS = 'LOT DEKIT'
ORDER BY DATE_TIME DESC;
```

### Purpose

Tracks lot splitting and wafer relationship history.

---

# Use Case 13

## Employee Activity Investigation

### Question

Show all activity performed by employee 05880.

### Query

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE EMPID = '05880'
ORDER BY DATE_TIME DESC;
```

### Purpose

Useful for audits and operator activity reviews.

---

# Use Case 14

## Machine Event History

### Question

Show everything that happened on machine NOVEL315 today.

### Query

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE MACHINE = 'NOVEL315'
  AND CAST(DATE_TIME AS DATE) = CAST(GETDATE() AS DATE)
ORDER BY DATE_TIME DESC;
```

### Purpose

Provides complete machine history for troubleshooting.

---

# Advanced Analytics Use Cases

---

# Use Case 15

## Lots Stuck in Queue for More Than X Hours

### Question

Which lots have been waiting for more than 2 hours?

### Query

```sql
WITH LatestEvent AS
(
    SELECT LOT,
           MAX(DATE_TIME) AS LastEventTime
    FROM TrainingVision.LotHistV
    GROUP BY LOT
)
SELECT *
FROM LatestEvent
WHERE DATEDIFF(HOUR,
               LastEventTime,
               GETDATE()) > 2;
```

### Purpose

Identifies aging WIP and potential bottlenecks.

---

# Use Case 16

## Process Bottleneck Analysis

### Question

Which operation has the most waiting lots?

### Query

```sql
SELECT OPERDESC,
       COUNT(DISTINCT LOT) AS WaitingLots
FROM TrainingVision.LotHistV
WHERE HIST_REC LIKE '%InQueue%'
GROUP BY OPERDESC
ORDER BY WaitingLots DESC;
```

### Purpose

Finds bottleneck operations across the factory.

---

# Use Case 17

## Equipment Loading Analysis

### Question

Which machines currently have the most active lots?

### Query

```sql
SELECT MACHINE,
       COUNT(DISTINCT LOT) AS ActiveLots
FROM TrainingVision.LotHistV
WHERE MACHINE IS NOT NULL
GROUP BY MACHINE
ORDER BY ActiveLots DESC;
```

### Purpose

Detects overloaded tools.

---

# Use Case 18

## Cycle Time Analysis

### Question

What is the average time between Move In and Move Out events by operation?

### Query

```sql
-- Conceptual Query

SELECT OPERDESC,
       AVG(DATEDIFF(MINUTE,
                    MoveInTime,
                    MoveOutTime))
       AS AvgProcessMinutes
FROM ProcessEvents
GROUP BY OPERDESC;
```

### Purpose

Measures process efficiency and cycle time.

---

# Use Case 19

## Rework Detection

### Question

Which lots visited the same operation multiple times?

### Query

```sql
SELECT LOT,
       OPERDESC,
       COUNT(*) AS VisitCount
FROM TrainingVision.LotHistV
GROUP BY LOT,
         OPERDESC
HAVING COUNT(*) > 1
ORDER BY VisitCount DESC;
```

### Purpose

Detects potential rework loops.

---

# Use Case 20

## Production Throughput

### Question

How many lots completed each operation today?

### Query

```sql
SELECT OPERDESC,
       COUNT(DISTINCT LOT) AS LotsCompleted
FROM TrainingVision.LotHistV
WHERE TRANS IN ('MOVE OUT', 'LOTCOMPLET')
  AND CAST(DATE_TIME AS DATE) =
      CAST(GETDATE() AS DATE)
GROUP BY OPERDESC
ORDER BY LotsCompleted DESC;
```

### Purpose

Measures production throughput.

---

# Use Case 21

## Shift Comparison

### Question

Compare production activity between day shift and night shift.

### Query

```sql
SELECT
    CASE
        WHEN DATEPART(HOUR, DATE_TIME)
            BETWEEN 6 AND 17
        THEN 'Day Shift'
        ELSE 'Night Shift'
    END AS ShiftName,
    COUNT(*) AS EventCount
FROM TrainingVision.LotHistV
GROUP BY
    CASE
        WHEN DATEPART(HOUR, DATE_TIME)
            BETWEEN 6 AND 17
        THEN 'Day Shift'
        ELSE 'Night Shift'
    END;
```

### Purpose

Compares workload and productivity by shift.

---

# Use Case 22

## WIP Trend Analysis

### Question

How has queue volume changed over the last 24 hours?

### Query

```sql
SELECT
    DATEADD(HOUR,
            DATEDIFF(HOUR, 0, DATE_TIME),
            0) AS HourBucket,
    COUNT(*) AS QueueEvents
FROM TrainingVision.LotHistV
WHERE HIST_REC LIKE '%InQueue%'
  AND DATE_TIME >= DATEADD(HOUR, -24, GETDATE())
GROUP BY DATEADD(HOUR,
                 DATEDIFF(HOUR, 0, DATE_TIME),
                 0)
ORDER BY HourBucket;
```

### Purpose

Provides queue trend analytics.

---

# Business Questions Customers Can Actually Ask

- Where is my lot right now?
- What happened to lot 6514A4L8?
- Show complete history of lot XXXXX.
- Who moved lot XXXXX?
- What machine processed this lot?
- Why is my lot waiting?
- Which lots are currently in queue?
- Which lots were dispatched today?
- Which lots were moved today?
- Show all activity on machine WB305.
- Show all activity on operator MJAMA.
- Which operator processed the most lots today?
- Which machine processed the most lots today?
- Which operation currently has the longest queue?
- Which lots are stuck?
- Which lots have not moved in the last 4 hours?
- Show all automated moves.
- Show all manual moves.
- Which lots were split or dekitted?
- Show equipment utilization across the fab.
- Which lots revisited the same operation?
- What is the average cycle time for IMPLANT?
- What is the throughput by operation?
- Compare Day Shift versus Night Shift productivity.
- Which machines are overloaded?
- Which lots are waiting for Move Out?
- Which lots were moved by employee 05880?
- Show all transitions performed by TGADOE.
- Show all checksum verification activities.
- Which machine generated the most alarms/comments?

---

# Gold Standard Test Question

## User Question

Where is lot 6905A002 and what happened to it most recently?

## Generated SQL

```sql
SELECT TOP 10
       LOT,
       DATE_TIME,
       TRANS,
       MACHINE,
       OPERDESC,
       HIST_REC
FROM TrainingVision.LotHistV
WHERE LOT = '6905A002'
  AND IS_DUPLICATE = 0
ORDER BY DATE_TIME DESC,
         HISTORDER DESC;
```

## Dummy Result

```text
Lot: 6905A002

Current Location:
Shelf 3319-F

Current Area:
32_WIP

Current Status:
In Queue

Latest Event:
LOCATION

Operation:
METR_FILMS

Timestamp:
2026-08-10 04:59:32

Interpretation:
The lot is currently in WIP Area 32_WIP, stored on Shelf 3319-F and waiting for the next processing step. No newer movement or processing events have been recorded after this location update.
```

---

# Recommended Copilot Agent Testing Parameters

## Entities

- LOT
- DATE_TIME
- TRANS
- MACHINE
- OPER
- OPERDESC
- USERNAME
- EMPID
- COMMAND
- HISTCODE
- HIST_REC
- SHORTREPORT
- IS_DUPLICATE

## Default Filters

```sql
IS_DUPLICATE = 0
```

## Time Interpretation Rules

- today
- yesterday
- last hour
- last 24 hours
- this shift
- previous shift
- this week
- last 7 days
- custom date range

## Business Synonym Mapping

| User Says | Map To |
|---|---|
| lot | LOT |
| wafer | LOT |
| batch | LOT |
| tool | MACHINE |
| equipment | MACHINE |
| machine | MACHINE |
| operator | USERNAME |
| employee | EMPID |
| process | OPERDESC |
| step | OPERDESC |
| action | TRANS |
| event | TRANS / HIST_REC |
| location | TRANS='LOCATION' |
| moved | MOVE IN / MOVE OUT |
| dispatched | DISPATCH |
| waiting | InQueue / WAIT VOU |
| history | All records ordered by DATE_TIME, HISTORDER |
