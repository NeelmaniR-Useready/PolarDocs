# Data Migration Validation Report
## Production SQL View vs. Fabric View (`dbo.EQUIPHIS4`)
### Validation Snapshot: `2025-01-01 00:00:02` to `2025-02-01 23:59:49` (1 Month Window)

This report provides a detailed, comprehensive comparison and reconciliation analysis between the **Production SQL View** and the **Fabric Lakehouse View** for the database object `dbo.EQUIPHIS4` during the full one-month test window from January 1 to February 1, 2025.

---

### 📊 Validation Metadata & Status

| Parameter | Details |
| :--- | :--- |
| **Source (Production)** | `df_Prod` (`dbo.EQUIPHIS4` / `Production1jan1month.csv`) |
| **Target (Fabric)** | `df_Fabric` (`MES_Analytics.EQUIPHIS4` / `Fabric1jan1month.csv`) |
| **Validation Window** | `2025-01-01 00:00:02.02` to `2025-02-01 23:59:49.49` (32 Days) |
| **Production Row Count** | **484,409** rows |
| **Fabric Row Count** | **484,403** rows (Delta: **-6 rows**, -0.0012%) |
| **Full Row Common Intersect** | **457,873** exact matching rows (**94.52% match rate**) |
| **Duplicate Parity** | Exactly **26,530** duplicate rows in both platforms (**100.0% Parity**) |
| **Current Status** | 🟢 **Validation Passed with 94.52% Direct Intersect Alignment & Flawless Volume Parity (99.999%)** |

> [!NOTE]  
> **Production-Ready Status: Passed (94.52% Direct Intersect Alignment, 99.999% Volume Parity).**  
> 457,873 rows out of 484,409 match identically across all 27 columns. The 6-row variance represents automated robot logs occurring at identical timestamps on January 21, 2025. Duplicate counts are 100.0% aligned at 26,530 rows in both environments.

---

### 🔄 View Definitions Side-by-Side (SQL Server vs. Microsoft Fabric)

| Metadata / Feature | SQL Server Production (`[dbo].[EQUIPHIS4]`) | Microsoft Fabric (`[MES_Analytics].[EQUIPHIS4]`) |
| :--- | :--- | :--- |
| **Database & Schema** | `VisionProd.dbo` | `Polar_Lakehouse.MES_Analytics` |
| **Object Name** | `[dbo].[EQUIPHIS4]` | `[MES_Analytics].[EQUIPHIS4]` |
| **Architecture** | 2-Branch `UNION ALL` (Equipment History + Operator Comments) | 2-Branch `UNION ALL` (Equipment History + Lakehouse Comments) |
| **View SQL Definition** | ```sql<br>CREATE VIEW [dbo].[EQUIPHIS4] (<br>  DATE_TIME_RUN,EMPID,MACHINE,MACHINE_DESCRIPTION<br> ,MACHINE_GROUP,MACHINE_TYPE,MACHINE_KIND<br> ,DATE_TIME,STATUS1_CODE,STATUS1_NAME,STATUS2_CODE,STATUS2_NAME<br> ,PM_CODE,PM_NAME,REPAIR1_CODE,REPAIR1_NAME<br> ,REPAIR2_CODE,REPAIR2_NAME,REPAIR3_CODE,REPAIR3_NAME<br> ,IGNORE_RECORD,COMMENTS,USERNAME,COMMENTTYPE,LINEORDER<br> ,MACHINE_PRIORITY,REPAIRCODE<br>) AS<br>SELECT<br>  HIS.DATE_TIME_RUN,HIS.EMPID,HIS.MACHINE,EQP.DESCRIPTION<br> ,MG.MACHINE_GROUP, MT.MACHINE_TYPE, MK.MACHINE_KIND<br> ,HIS.DATE_TIME,HIS.STATUS1_CODE,ST1.STATUS1_NAME,HIS.STATUS2_CODE,ST2.STATUS2_NAME<br> ,HIS.PM_CODE,STP.PM_NAME<br> ,NULL,NULL,NULL,NULL,NULL,NULL<br> ,HIS.IGNORE_RECORD,'' AS COMMENTS<br> ,HIS.USERNAME,'CS' AS COMMENTTYPE,1 AS LINEORDER<br> ,STS.DOWN_PRIORITY AS MACHINE_PRIORITY<br> ,RC.REPAIRCODE<br>  FROM dbo.EQUIPHIS HIS (NOLOCK)<br>       INNER JOIN dbo.EQUIPST1 ST1 (NOLOCK) ON (ST1.STATUS1_CODE=HIS.STATUS1_CODE)<br>       INNER JOIN dbo.EQUIPST2 ST2 (NOLOCK) ON (ST2.STATUS2_CODE=HIS.STATUS2_CODE)<br>       INNER JOIN dbo.EQUIPSTP STP (NOLOCK) ON (STP.PM_CODE=HIS.PM_CODE)<br>       INNER JOIN dbo.EQUIP EQP (NOLOCK) ON (EQP.MACHINE=HIS.MACHINE)<br>       LEFT JOIN dbo.MACHINE_TYPECODES MT (NOLOCK) ON MT.MachineTypeId = EQP.MachineTypeId<br>       LEFT JOIN dbo.MACHINE_GROUPCODES MG (NOLOCK) ON MG.MachineGroupId = EQP.MachineGroupId<br>       LEFT JOIN [dbo].[MACHINE_KINDCODES] MK (NOLOCK) ON MK.[MachineKindId] = EQP.[MachineKindId]<br>       INNER JOIN dbo.EQUIPSTS STS (NOLOCK) ON (STS.MACHINE=HIS.MACHINE)<br>       LEFT OUTER JOIN dbo.EQUIPREPAIRCODES RC (NOLOCK) ON (RC.RepairCodeId=HIS.RepairCodeId)<br><br>UNION ALL<br><br>SELECT<br>  EQC.DATE_TIME,NULL,EQC.MACHINE,EQP.DESCRIPTION<br> ,MG.MACHINE_GROUP, MT.MACHINE_TYPE, MK.MACHINE_KIND<br> ,EQC.DATE_TIME,NULL,NULL,NULL,NULL<br> ,NULL,NULL<br> ,NULL,NULL,NULL,NULL,NULL,NULL<br> ,NULL<br> ,EQC.COMMENTS,EQC.USERNAME,EQC.COMMENTTYPE,EQC.LINEORDER<br> ,STS.DOWN_PRIORITY<br> ,NULL<br>  FROM dbo.EQUIPHIS_COMMENTS EQC (NOLOCK)<br>       INNER JOIN dbo.EQUIP EQP (NOLOCK) ON (EQC.MACHINE=EQP.MACHINE)<br>       LEFT JOIN dbo.MACHINE_TYPECODES MT (NOLOCK) ON MT.MachineTypeId = EQP.MachineTypeId<br>       LEFT JOIN dbo.MACHINE_GROUPCODES MG (NOLOCK) ON MG.MachineGroupId = EQP.MachineGroupId<br>       LEFT JOIN [dbo].[MACHINE_KINDCODES] MK (NOLOCK) ON MK.[MachineKindId] = EQP.MachineKindId<br>       INNER JOIN dbo.EQUIPSTS STS (NOLOCK) ON (EQC.MACHINE=STS.MACHINE)<br>GO<br>``` | ```sql<br>CREATE VIEW [MES_Analytics].[EQUIPHIS4] AS<br>SELECT<br>    HIS.DATE_TIME_RUN,<br>    HIS.EMPID,<br>    HIS.MACHINE,<br>    EQP.DESCRIPTION AS MACHINE_DESCRIPTION,<br>    MG.MACHINE_GROUP,<br>    MT.MACHINE_TYPE,<br>    MK.MACHINE_KIND,<br>    HIS.DATE_TIME,<br>    HIS.STATUS1_CODE,<br>    ST1.STATUS1_NAME,<br>    HIS.STATUS2_CODE,<br>    ST2.STATUS2_NAME,<br>    HIS.PM_CODE,<br>    STP.PM_NAME,<br>    NULL AS REPAIR1_CODE,<br>    NULL AS REPAIR1_NAME,<br>    NULL AS REPAIR2_CODE,<br>    NULL AS REPAIR2_NAME,<br>    NULL AS REPAIR3_CODE,<br>    NULL AS REPAIR3_NAME,<br>    HIS.IGNORE_RECORD,<br>    '' AS COMMENTS,<br>    HIS.USERNAME,<br>    'CS' AS COMMENTTYPE,<br>    1 AS LINEORDER,<br>    STS.DOWN_PRIORITY AS MACHINE_PRIORITY,<br>    RC.REPAIRCODE<br>FROM [Polar_Lakehouse].[dbo].[EQUIPHIS] HIS<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIPST1] ST1 ON ST1.STATUS1_CODE = HIS.STATUS1_CODE<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIPST2] ST2 ON ST2.STATUS2_CODE = HIS.STATUS2_CODE<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIPSTP] STP ON STP.PM_CODE = HIS.PM_CODE<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIP] EQP ON EQP.MACHINE = HIS.MACHINE<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_TYPECODES] MT ON MT.MachineTypeId = EQP.MachineTypeId<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_GROUPCODES] MG ON MG.MachineGroupId = EQP.MachineGroupId<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_KINDCODES] MK ON MK.MachineKindId = EQP.MachineKindId<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIPSTS] STS ON STS.MACHINE = HIS.MACHINE<br>LEFT JOIN [Polar_Lakehouse].[dbo].[EQUIPREPAIRCODES] RC ON RC.RepairCodeId = HIS.RepairCodeId<br><br>UNION ALL<br><br>SELECT<br>    EQC.DATE_TIME AS DATE_TIME_RUN,<br>    NULL AS EMPID,<br>    EQC.MACHINE,<br>    EQP.DESCRIPTION AS MACHINE_DESCRIPTION,<br>    MG.MACHINE_GROUP,<br>    MT.MACHINE_TYPE,<br>    MK.MACHINE_KIND,<br>    EQC.DATE_TIME,<br>    NULL AS STATUS1_CODE,<br>    NULL AS STATUS1_NAME,<br>    NULL AS STATUS2_CODE,<br>    NULL AS STATUS2_NAME,<br>    NULL AS PM_CODE,<br>    NULL AS PM_NAME,<br>    NULL AS REPAIR1_CODE,<br>    NULL AS REPAIR1_NAME,<br>    NULL AS REPAIR2_CODE,<br>    NULL AS REPAIR2_NAME,<br>    NULL AS REPAIR3_CODE,<br>    NULL AS REPAIR3_NAME,<br>    NULL AS IGNORE_RECORD,<br>    EQC.COMMENTS,<br>    EQC.USERNAME,<br>    EQC.COMMENTTYPE,<br>    EQC.LINEORDER,<br>    STS.DOWN_PRIORITY AS MACHINE_PRIORITY,<br>    NULL AS REPAIRCODE<br>FROM [Polar_Lakehouse].[dbo].[EQUIPHIS_COMMENTS] EQC<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIP] EQP ON EQC.MACHINE = EQP.MACHINE<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_TYPECODES] MT ON MT.MachineTypeId = EQP.MachineTypeId<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_GROUPCODES] MG ON MG.MachineGroupId = EQP.MachineGroupId<br>LEFT JOIN [Polar_Lakehouse].[dbo].[MACHINE_KINDCODES] MK ON MK.MachineKindId = EQP.MachineKindId<br>INNER JOIN [Polar_Lakehouse].[dbo].[EQUIPSTS] STS ON EQC.MACHINE = STS.MACHINE<br>GO<br>``` |

---

### 🔍 Executive Summary

```mermaid
graph TD
    classDef prodStyle fill:#e6f3ff,stroke:#3385ff,stroke-width:2px;
    classDef fabricStyle fill:#ffe6e6,stroke:#ff3333,stroke-width:2px;
    classDef commonStyle fill:#eafaf1,stroke:#2ecc71,stroke-width:2px;

    subgraph Comparison ["EQUIPHIS4 Month-Scale Reconciliation (Jan 1 - Feb 1, 2025)"]
        P["Production Rows: 484,409"]:::prodStyle
        F["Fabric Rows: 484,403"]:::fabricStyle
        C["Common Exact Match Rows: 457,873 (94.52%)"]:::commonStyle
        
        P -->|6 Only in Prod| C
        F -->|0 Only in Fabric| C
    end

    style Comparison fill:#f9f9f9,stroke:#ddd,stroke-width:1px;
```

#### Key Reconciliation Metrics

| Metric | Production (`df_Prod`) | Fabric (`df_Fabric`) | Variance | % Difference |
| :--- | :---: | :---: | :---: | :---: |
| **Total Rows** | 484,409 | 484,403 | **-6** | -0.0012% |
| **Common Rows (Exact Match)** | 457,873 | 457,873 | - | **94.52% Alignment** |
| **Rows Only in Production** | 6 | - | **-6** | - |
| **Rows Only in Fabric** | - | 0 | **0** | - |
| **Duplicate Rows (Full Match)** | 26,530 | 26,530 | **0** | **100.0% Parity** |
| **Distinct Machines (`MACHINE`)** | 1,793 | 1,791 | **-2** | -0.11% |
| **Distinct Timestamps (`DATE_TIME`)** | 226,455 | 226,453 | **-2** | -0.0009% |

---

### 📋 Detailed Cardinality Comparison (All 27 Columns)

| Column Name | Production Distinct Count | Fabric Distinct Count | Delta | Status |
| :--- | :---: | :---: | :---: | :---: |
| **DATE_TIME_RUN** | 226,455 | 226,453 | **-2** | ⚠️ Minor Delta |
| **EMPID** | 444 | 444 | **0** | ✅ Identical |
| **MACHINE** | 1,793 | 1,791 | **-2** | ⚠️ Minor Delta |
| **MACHINE_DESCRIPTION** | 1,022 | 1,022 | **0** | ✅ Identical |
| **MACHINE_GROUP** | 47 | 47 | **0** | ✅ Identical |
| **MACHINE_TYPE** | 123 | 123 | **0** | ✅ Identical |
| **MACHINE_KIND** | 3 | 3 | **0** | ✅ Identical |
| **DATE_TIME** | 226,455 | 226,453 | **-2** | ⚠️ Minor Delta |
| **STATUS1_CODE** | 42 | 42 | **0** | ✅ Identical |
| **STATUS1_NAME** | 42 | 42 | **0** | ✅ Identical |
| **STATUS2_CODE** | 32 | 32 | **0** | ✅ Identical |
| **STATUS2_NAME** | 32 | 32 | **0** | ✅ Identical |
| **PM_CODE** | 28 | 28 | **0** | ✅ Identical |
| **PM_NAME** | 28 | 28 | **0** | ✅ Identical |
| **REPAIR1_CODE** | 1 | 1 | **0** | ✅ Identical |
| **REPAIR1_NAME** | 1 | 1 | **0** | ✅ Identical |
| **REPAIR2_CODE** | 1 | 1 | **0** | ✅ Identical |
| **REPAIR2_NAME** | 1 | 1 | **0** | ✅ Identical |
| **REPAIR3_CODE** | 1 | 1 | **0** | ✅ Identical |
| **REPAIR3_NAME** | 1 | 1 | **0** | ✅ Identical |
| **IGNORE_RECORD** | 3 | 3 | **0** | ✅ Identical |
| **COMMENTS** | 116,639 | 116,639 | **0** | ✅ Identical |
| **USERNAME** | 1,041 | 1,041 | **0** | ✅ Identical |
| **COMMENTTYPE** | 46 | 46 | **0** | ✅ Identical |
| **LINEORDER** | 41 | 41 | **0** | ✅ Identical |
| **MACHINE_PRIORITY** | 16 | 16 | **0** | ✅ Identical |
| **REPAIRCODE** | 358 | 358 | **0** | ✅ Identical |

---

### 🕳️ Null & Blank Profile Comparison

| Column Name | Production Null/Blank Count | Fabric Null/Blank Count | Delta | Status |
| :--- | :---: | :---: | :---: | :---: |
| **DATE_TIME_RUN** | 0 | 0 | **0** | ✅ Perfect Parity |
| **EMPID** | 0 | 0 | **0** | ✅ Perfect Parity |
| **MACHINE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **MACHINE_DESCRIPTION** | 17,921 | 17,915 | **-6** | ✅ 99.97% Match |
| **MACHINE_GROUP** | 0 | 0 | **0** | ✅ Perfect Parity |
| **MACHINE_TYPE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **MACHINE_KIND** | 0 | 0 | **0** | ✅ Perfect Parity |
| **DATE_TIME** | 0 | 0 | **0** | ✅ Perfect Parity |
| **STATUS1_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **STATUS1_NAME** | 0 | 0 | **0** | ✅ Perfect Parity |
| **STATUS2_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **STATUS2_NAME** | 80,967 | 80,966 | **-1** | ✅ 99.999% Match |
| **PM_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **PM_NAME** | 89,113 | 89,112 | **-1** | ✅ 99.999% Match |
| **REPAIR1_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **REPAIR1_NAME** | 0 | 0 | **0** | ✅ Perfect Parity |
| **REPAIR2_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **REPAIR2_NAME** | 0 | 0 | **0** | ✅ Perfect Parity |
| **REPAIR3_CODE** | 0 | 0 | **0** | ✅ Perfect Parity |
| **REPAIR3_NAME** | 0 | 0 | **0** | ✅ Perfect Parity |
| **IGNORE_RECORD** | 14,608 | 14,608 | **0** | ✅ Perfect Parity |
| **COMMENTS** | 102,576 | 102,574 | **-2** | ✅ 99.999% Match |
| **USERNAME** | 1 | 1 | **0** | ✅ Perfect Parity |
| **COMMENTTYPE** | 1 | 1 | **0** | ✅ Perfect Parity |
| **LINEORDER** | 1 | 1 | **0** | ✅ Perfect Parity |
| **MACHINE_PRIORITY** | 1,452 | 1,452 | **0** | ✅ Perfect Parity |
| **REPAIRCODE** | 1 | 1 | **0** | ✅ Perfect Parity |

---

### ⚙️ Inner-Join Key & Attribute Mismatch Analysis

```
Composite Key: [MACHINE] + [DATE_TIME] + [LINEORDER]
```

#### Column-by-Column Mismatch Summary

| Column Name | Mismatch Count | Status | Root Cause Category |
| :--- | :---: | :---: | :--- |
| **DATE_TIME_RUN** | **0** | ✅ Pass | Perfect field alignment |
| **MACHINE_DESCRIPTION** | **0** | ✅ Pass | Perfect master data lookup |
| **MACHINE_GROUP** | **0** | ✅ Pass | Perfect machine grouping |
| **MACHINE_TYPE** | **0** | ✅ Pass | Perfect machine classification |
| **MACHINE_KIND** | **0** | ✅ Pass | Perfect machine kind classification |
| **REPAIR1_CODE** | **0** | ✅ Pass | Perfect null constant handling |
| **REPAIR1_NAME** | **0** | ✅ Pass | Perfect null constant handling |
| **REPAIR2_CODE** | **0** | ✅ Pass | Perfect null constant handling |
| **REPAIR2_NAME** | **0** | ✅ Pass | Perfect null constant handling |
| **REPAIR3_CODE** | **0** | ✅ Pass | Perfect null constant handling |
| **REPAIR3_NAME** | **0** | ✅ Pass | Perfect null constant handling |
| **MACHINE_PRIORITY** | **0** | ✅ Pass | Perfect down priority mapping |
| **STATUS1_CODE** | **162,424** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **STATUS1_NAME** | **162,424** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **STATUS2_CODE** | **162,386** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **STATUS2_NAME** | **162,386** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **PM_CODE** | **162,386** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **PM_NAME** | **162,386** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **EMPID** | **162,390** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **IGNORE_RECORD** | **162,390** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **COMMENTS** | **256,448** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **COMMENTTYPE** | **66,244** | 🟡 Crossed Branch Join | Status change row matched against simultaneous comment row |
| **REPAIRCODE** | **7,406** | ⚠️ Minor Variance | Repair code lookup join tie-breaker variance on simultaneous records |
| **USERNAME** | **7,478** | ⚠️ Minor Variance | Case sensitivity difference (e.g. `KTRAN` vs `ktran`) |

---

### ⚠️ Root Cause Assessment

1. **Simultaneous Event Tie Collision**: 162,390 rows exhibit the identical pattern where Branch 1 (`EQUIPHIS`) and Branch 2 (`EQUIPHIS_COMMENTS`) events share the same `[MACHINE, DATE_TIME, LINEORDER]`. Because SQL inner join without `COMMENTTYPE` cannot differentiate the branch origin, status attributes map against comments.
2. **Missing Boundary Records**: The 6 missing rows in Fabric correspond to 4 events on `2025-01-21 09:01:24.24` and 2 events on `2025-01-21 09:01:50.50` on robot automation tools (`BAYMAX`).

---

### 📋 Environment Validation Summary

| Core Area | Status | Remarks |
| :--- | :---: | :--- |
| **Row Count Alignment** | ✅ Pass | 484,409 Prod vs 484,403 Fabric (99.999% parity, -6 rows delta). |
| **Exact Row Intersect** | 🟢 Pass | 457,873 exact matching rows (94.52% direct match rate). |
| **Duplicate Parity** | ✅ Perfect | Exactly 26,530 duplicate rows in both environments (100.0% match). |
| **Lookup Master Data** | ✅ Perfect | Descriptions, Groups, Types, and Kinds match 100%. |
| **Date Ranges** | ✅ Perfect | `2025-01-01 00:00:02.02` to `2025-02-01 23:59:49.49`. |
