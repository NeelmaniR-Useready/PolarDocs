LOT,DATE_TIME,HISTORDER,TRANS,OPER,MASK_LVL,OPERDESC,OPERLONGDESC,MACHINE,USERNAME,HIST_REC,HISTCODE,COMMAND,SHORTREPORT,VIEWFLAG,Is_Person,IS_DUPLICATE,EMPID
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:015  MSK:60609801  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:046  MSK:60609808  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,8,HOT STATUS,90101,0,SCRIBE,000.SCRIBE,,johnsonr,HOTFLAG= 4  OLD= ,HT,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,9,SALES ORDR,90101,0,SCRIBE,000.SCRIBE,,johnsonr,SALES ORDER:BNKORD0092 LINE:4691 ,SO,LTCR,Y,I,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:078  MSK:60609811  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,1,PARAM1 FLG,90101,0,SCRIBE,000.SCRIBE,,johnsonr,1ST PARAMETRIC FLAG:ON,P1,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,6,ROUTE,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ROUTE:SBH2-01A  ,RT,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,10,CREATE LOT,90101,0,SCRIBE,000.SCRIBE,,johnsonr,QTY: 25 SCRIBED:F31082,CR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:075  MSK:60609810  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:030  MSK:60609803  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,11,RAWWFR PRT,90101,0,SCRIBE,000.SCRIBE,,johnsonr,RAW WAFER NUMBER:61332900 Rev: F,WF,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,3,OWNERCODE,90101,0,SCRIBE,000.SCRIBE,,johnsonr,OWNER:PROD,OW,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:038  MSK:60609806  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:043  MSK:60609807  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,7,VALUEADDED,90101,0,SCRIBE,000.SCRIBE,,johnsonr,NEW VALUEADDED:NORMAL,VA,LTCR,Y,I,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,COMMENT,90101,0,SCRIBE,000.SCRIBE,,johnsonr,Wfrs added to WaferSortData.,CM,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,5,PART NUM,90101,0,SCRIBE,000.SCRIBE,,johnsonr,PART:C_MG5913AP-FE,PT,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,4,TARGETPRTn,90101,0,SCRIBE,000.SCRIBE,,johnsonr,NEW TARGET PART:MG5913AP-FE,TN,LTCR,Y,E,1,1,4518
6554A0A7,2026-07-26 11:17:27.27,12,MASK/REV,90101,0,SCRIBE,000.SCRIBE,,johnsonr,ADD LVL:029  MSK:60609804  REV:A  SET: ,MR,LTCR,Y,E,1,1,4518

# Lot History Analysis - LOT 6554A0A7

## Query Used

```sql
SELECT *
FROM TrainingVision.LotHistV
WHERE LOT = '6554A0A7'
ORDER BY DATE_TIME;
```

---

# Sample Data Retrieved

LOT: **6554A0A7**

The query returned multiple lot history records representing the creation and initialization of the lot during the SCRIBE operation.

## Key Columns Present

- LOT
- DATE_TIME
- HISTORDER
- TRANS
- OPER
- MASK_LVL
- OPERDESC
- OPERLONGDESC
- MACHINE
- USERNAME
- HIST_REC
- HISTCODE
- COMMAND
- SHORTREPORT
- VIEWFLAG
- Is_Person
- IS_DUPLICATE
- EMPID

---

# Sample Events Observed

| HISTORDER | TRANS | HIST_REC |
|------------|---------|------------|
| 10 | CREATE LOT | QTY: 25 SCRIBED:F31082 |
| 11 | RAWWFR PRT | RAW WAFER NUMBER:61332900 Rev:F |
| 6 | ROUTE | ROUTE:SBH2-01A |
| 5 | PART NUM | PART:C_MG5913AP-FE |
| 4 | TARGETPRTn | NEW TARGET PART:MG5913AP-FE |
| 7 | VALUEADDED | NEW VALUEADDED:NORMAL |
| 8 | HOT STATUS | HOTFLAG=4 |
| 9 | SALES ORDR | SALES ORDER:BNKORD0092 |
| 3 | OWNERCODE | OWNER:PROD |
| 12 | MASK/REV | Multiple mask level assignments |

---

# Primary Business Columns

The following columns are expected to answer the majority of customer and manufacturing support questions.

## Tier 1 - Most Important

| Column | Why It Matters |
|----------|----------------|
| LOT | Primary identifier for all investigations |
| DATE_TIME | Enables historical timeline reconstruction |
| TRANS | Transaction/Event type |
| HIST_REC | Detailed event information |
| OPER | Manufacturing operation number |
| OPERDESC | Process step description |
| MACHINE | Equipment used |
| USERNAME | User responsible for action |
| HISTCODE | Event classification code |

---

## Tier 2 - Frequently Used

| Column | Why It Matters |
|----------|----------------|
| SHORTREPORT | Reporting category |
| EMPID | Employee identifier |
| VIEWFLAG | Visibility of record |
| COMMAND | Command executed during transaction |

---

## Tier 3 - Investigative / Specialized

| Column | Why It Matters |
|----------|----------------|
| MASK_LVL | Mask tracking and layer analysis |
| OPERLONGDESC | Extended operation description |
| Is_Person | User classification |
| IS_DUPLICATE | Duplicate detection analysis |

---

# Critical Copilot Questions

## Lot Traceability

1. Show the complete history of Lot 6554A0A7.
2. What events occurred for Lot 6554A0A7 in chronological order?
3. When was Lot 6554A0A7 created?
4. Who created Lot 6554A0A7?
5. Which machine processed Lot 6554A0A7?
6. What route was assigned to Lot 6554A0A7?
7. What was the quantity at lot creation?

---

## Product & Part Investigation

8. What part number is associated with Lot 6554A0A7?
9. What target part was assigned to the lot?
10. What raw wafer number was used for this lot?
11. Which revision was used for the wafer?
12. What owner code was assigned to the lot?

---

## Mask and Revision Analysis

13. What masks were assigned to Lot 6554A0A7?
14. Which mask levels were added during lot creation?
15. Show all MASK/REV transactions for the lot.
16. What mask revisions are associated with the lot?
17. Which mask was added at level 015?
18. Which mask was added at level 078?

---

## Manufacturing Process Questions

19. Which operation processed the lot?
20. What process step description is associated with the lot?
21. Which manufacturing route was selected?
22. What transactions occurred during SCRIBE?
23. Show all events performed at operation 90101.

---

## User Activity Investigation

24. Which user performed actions on Lot 6554A0A7?
25. Show all activities performed by user johnsonr.
26. Which employee ID performed the transactions?
27. Which lots were processed by employee 4518?

---

## Sales Order & Planning

28. What sales order is associated with the lot?
29. Which line item is associated with sales order BNKORD0092?
30. Show lots linked to sales order BNKORD0092.

---

## Priority & Status Questions

31. Is the lot marked as a Hot Lot?
32. What hot status was assigned to the lot?
33. What value-added classification was assigned?
34. Which lots have HOTFLAG = 4?
35. Show all hot lots created during the same period.

---

## Timeline & Audit Questions

36. Show the first recorded event for the lot.
37. Show the last recorded event for the lot.
38. What events occurred at the same timestamp?
39. Reconstruct the exact sequence of lot creation events.
40. Show all transactions performed on 2026-07-26.

---

## Root Cause & Investigation Questions

41. Which user modified mask assignments on the lot?
42. When were masks added to the lot?
43. Were any duplicate records found?
44. What comments were recorded against this lot?
45. Show all COMMENT transactions.
46. Which events changed lot attributes?
47. Which transactions impacted manufacturability?

---

## Cross-Lot Manufacturing Questions

48. Find all lots using Route SBH2-01A.
49. Find all lots using Part MG5913AP-FE.
50. Find all lots using Raw Wafer 61332900.
51. Find all lots created by johnsonr.
52. Find all lots processed through operation 90101.
53. Find all lots having the same mask set.
54. Find all lots associated with sales order BNKORD0092.

---

# Most Valuable Columns for Copilot Semantic Search

1. LOT
2. DATE_TIME
3. TRANS
4. HIST_REC
5. OPER
6. OPERDESC
7. MACHINE
8. USERNAME
9. EMPID
10. HISTCODE
11. PART information embedded in HIST_REC
12. ROUTE information embedded in HIST_REC
13. SALES ORDER information embedded in HIST_REC
14. RAW WAFER information embedded in HIST_REC
15. MASK / REV information embedded in HIST_REC

These columns collectively answer approximately 90-95% of manufacturing support, lot genealogy, traceability, mask tracking, process audit, operator investigation, route analysis, and customer inquiry scenarios.
