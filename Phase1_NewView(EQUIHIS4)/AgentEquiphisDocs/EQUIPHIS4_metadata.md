# EQUIPHIS4 Metadata

## Source and scope

This document preserves the table details, all 28 column metadata entries, and both sample rows supplied from the query results.

- **Fully qualified table:** `[MES_Analytics].[TrainingVision_EQUIPHIS4]`
- **Platform context:** Microsoft Fabric
- **Object:** Table, not a view
- **Definition availability:** The supplied results contain table details and column metadata, not an original `CREATE TABLE` statement.
- **Sample scope:** Two sample rows only; these do not establish the full table contents or latest records.
- **Value notation:** `NULL` represents a supplied null value. An empty cell represents a supplied blank value, distinct from `NULL`.

## 1. Table details

| SchemaName | TableName | ObjectID | ObjectType | CreatedDate | ModifiedDate |
| --- | --- | --- | --- | --- | --- |
| MES_Analytics | TrainingVision_EQUIPHIS4 | 1212895738 | USER_TABLE | 2026-10-01T19:51:26.290 | 2026-10-01T19:51:26.290 |

## 2. Column metadata

| OrdinalPosition | ColumnName | DataType | MaxLengthBytes | PrecisionValue | ScaleValue | IsNullable | IsIdentity | IsComputed |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | DATE_TIME_RUN | datetime2 | 8 | 26 | 6 | 1 | 0 | 0 |
| 2 | EMPID | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 3 | MACHINE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 4 | MACHINE_DESCRIPTION | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 5 | MACHINE_GROUP | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 6 | MACHINE_TYPE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 7 | MACHINE_KIND | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 8 | DATE_TIME | datetime2 | 8 | 26 | 6 | 1 | 0 | 0 |
| 9 | STATUS1_CODE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 10 | STATUS1_NAME | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 11 | STATUS2_CODE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 12 | STATUS2_NAME | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 13 | PM_CODE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 14 | PM_NAME | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 15 | REPAIR1_CODE | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 16 | REPAIR1_NAME | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 17 | REPAIR2_CODE | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 18 | REPAIR2_NAME | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 19 | REPAIR3_CODE | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 20 | REPAIR3_NAME | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 21 | IGNORE_RECORD | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 22 | COMMENTS | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 23 | USERNAME | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 24 | COMMENTTYPE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 25 | LINEORDER | int | 4 | 10 | 0 | 1 | 0 | 0 |
| 26 | MACHINE_PRIORITY | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 27 | REPAIRCODE | varchar | 8000 | 0 | 0 | 1 | 0 | 0 |
| 28 | MachineId | int | 4 | 10 | 0 | 1 | 0 | 0 |

## 3. Sample rows

### Complete query-result layout

| DATE_TIME_RUN | EMPID | MACHINE | MACHINE_DESCRIPTION | MACHINE_GROUP | MACHINE_TYPE | MACHINE_KIND | DATE_TIME | STATUS1_CODE | STATUS1_NAME | STATUS2_CODE | STATUS2_NAME | PM_CODE | PM_NAME | REPAIR1_CODE | REPAIR1_NAME | REPAIR2_CODE | REPAIR2_NAME | REPAIR3_CODE | REPAIR3_NAME | IGNORE_RECORD | COMMENTS | USERNAME | COMMENTTYPE | LINEORDER | MACHINE_PRIORITY | REPAIRCODE | MachineId |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-07-27 03:54:32.980000 | NULL | ENDURA306.SIP_3 | only chamber3 | METAL | ENDURA | FAB | 2026-07-27 03:54:32.980000 | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | Virtual tool status changed based on machine status change. | KIMY | CS | 4 | 0 | NULL | 8933 |
| 2025-12-25 14:03:06.760000 | NULL | ENDURA210.TI_TIN |  | METAL | ENDURA | FAB | 2025-12-25 14:03:06.760000 | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | Virtual tool status changed based on machine status change. | SMESMER | CS | 4 | 0 | NULL | 10330 |

### Field-by-field layout for readability

The same two records are reproduced below without changing their values.

| ColumnName | Sample row 1 | Sample row 2 |
| --- | --- | --- |
| DATE_TIME_RUN | 2026-07-27 03:54:32.980000 | 2025-12-25 14:03:06.760000 |
| EMPID | NULL | NULL |
| MACHINE | ENDURA306.SIP_3 | ENDURA210.TI_TIN |
| MACHINE_DESCRIPTION | only chamber3 |  |
| MACHINE_GROUP | METAL | METAL |
| MACHINE_TYPE | ENDURA | ENDURA |
| MACHINE_KIND | FAB | FAB |
| DATE_TIME | 2026-07-27 03:54:32.980000 | 2025-12-25 14:03:06.760000 |
| STATUS1_CODE | NULL | NULL |
| STATUS1_NAME | NULL | NULL |
| STATUS2_CODE | NULL | NULL |
| STATUS2_NAME | NULL | NULL |
| PM_CODE | NULL | NULL |
| PM_NAME | NULL | NULL |
| REPAIR1_CODE | NULL | NULL |
| REPAIR1_NAME | NULL | NULL |
| REPAIR2_CODE | NULL | NULL |
| REPAIR2_NAME | NULL | NULL |
| REPAIR3_CODE | NULL | NULL |
| REPAIR3_NAME | NULL | NULL |
| IGNORE_RECORD | NULL | NULL |
| COMMENTS | Virtual tool status changed based on machine status change. | Virtual tool status changed based on machine status change. |
| USERNAME | KIMY | SMESMER |
| COMMENTTYPE | CS | CS |
| LINEORDER | 4 | 4 |
| MACHINE_PRIORITY | 0 | 0 |
| REPAIRCODE | NULL | NULL |
| MachineId | 8933 | 10330 |

## 4. Metadata observations

- All 28 columns have `IsNullable = 1`, `IsIdentity = 0`, and `IsComputed = 0`.
- `DATE_TIME_RUN` and `DATE_TIME` are `datetime2` columns with scale 6.
- All supplied `varchar` columns have `MaxLengthBytes = 8000`.
- `REPAIR1_NAME`, `REPAIR2_NAME`, and `REPAIR3_NAME` are reported as `int`; their types are preserved exactly as supplied despite the `NAME` suffix.
- `MACHINE_PRIORITY` is reported as `varchar`; its sample value is `0` in both records.
- The second sample row has a blank `MACHINE_DESCRIPTION`, not the literal value `NULL`.
- Both sample comments read: "Virtual tool status changed based on machine status change."
- No timezone, keys, indexes, constraints, or business definitions were supplied in these results.
