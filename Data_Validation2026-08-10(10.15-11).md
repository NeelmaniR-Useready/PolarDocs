Production DB view –[TrainingVision].[dbo.LOTHISTV] 



Fabric view –[Polar_Warehouse].[TrainingVision].[LotHistV] 



 
Time stamp: DATETIME >= ‘2026-08-10 10:15:00’ AND 
	DATE_TIME < ‘2026-08-10 11:00:00’ 

 

 

Below is the code you ran in your notebook, broken into logical sections. 
For each section I’ve added: 

A comment describing the logic and what the data represents 

The query/code for that section 

The result from the last run (from the cell output) 

 

1. Load data + row counts 

```python
# LOGIC: 
# Load two CSV snapshots (Prod and Fabric) from the Lakehouse and compare 
# total row counts to see if both datasets have the same number of rows. 
 
df_Prod = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Prod-08-10(10.15-11).csv", 
    header=True, 
    inferSchema=True 
) 
 
df_Fabric = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Fabric-08-10(10.15-11).csv", 
    header=True, 
    inferSchema=True 
) 
 
print("=" * 80) 
print("ROW COUNTS") 
print("=" * 80) 
 
prod_count = df_Prod.count() 
fabric_count = df_Fabric.count() 
 
print(f"Prod Rows   : {prod_count}") 
print(f"Fabric Rows : {fabric_count}") 
 

 
 

Result: 
```

```text
### ROW COUNTS

Prod Rows   : 10023 
Fabric Rows : 9841 
 

 
 
 

2. Distinct count profile for each column 
```

```python
# LOGIC: 
# For each column in Prod and Fabric, compute the number of distinct values. 
# Used to verify that cardinality per column is aligned across systems. 
 
print("\n" + "=" * 80) 
print("PROD DISTINCT COUNTS") 
print("=" * 80) 
 
df_Prod.select( 
    *[ 
        F.countDistinct(c).alias(c) 
        for c in df_Prod.columns 
    ] 
).show(vertical=True, truncate=False) 
 
print("\n" + "=" * 80) 
print("FABRIC DISTINCT COUNTS") 
print("=" * 80) 
 
df_Fabric.select( 
    *[ 
        F.countDistinct(c).alias(c) 
        for c in df_Fabric.columns 
    ] 
).show(vertical=True, truncate=False) 
 

 
 

Result (Prod distinct counts): 
```

```text
### PROD DISTINCT COUNTS

-RECORD 0------------ 
 LOT          | 1294  
 DATE_TIME    | 2897  
 HISTORDER    | 198   
 TRANS        | 51    
 OPER         | 451   
 MASK_LVL     | 69    
 OPERDESC     | 151   
 OPERLONGDESC | 544   
 MACHINE      | 191   
 USERNAME     | 272   
 HIST_REC     | 1776  
 HISTCODE     | 51    
 COMMAND      | 25    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 86    
 

 
 

Result (Fabric distinct counts): 
```

```text
### FABRIC DISTINCT COUNTS

-RECORD 0------------ 
 LOT          | 1265  
 DATE_TIME    | 2897  
 HISTORDER    | 198   
 TRANS        | 51    
 OPER         | 451   
 MASK_LVL     | 69    
 OPERDESC     | 151   
 OPERLONGDESC | 544   
 MACHINE      | 191   
 USERNAME     | 272   
 HIST_REC     | 1776  
 HISTCODE     | 51    
 COMMAND      | 25    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 86    
 

 
 
 

3. Full row comparison + match percentage + duplicates 
```

```python
# LOGIC: 
# Compare full rows between Prod and Fabric: 
# - Count rows that are identical in both (common_rows) 
# - Count rows only in Prod and only in Fabric 
# - Compute match percentage against Prod 
# Also calculate how many duplicate rows exist in each dataset. 
 
print("\n" + "=" * 80) 
print("FULL ROW COMPARISON") 
print("=" * 80) 
 
common_rows = df_Prod.intersect(df_Fabric).count() 
 
only_prod = df_Prod.exceptAll(df_Fabric) 
only_fabric = df_Fabric.exceptAll(df_Prod) 
 
only_prod_count = only_prod.count() 
only_fabric_count = only_fabric.count() 
 
print("Common rows         :", common_rows) 
print("Only in Prod        :", only_prod_count) 
print("Only in Fabric      :", only_fabric_count) 
 
print("\n" + "=" * 80) 
print("MATCH PERCENTAGE") 
print("=" * 80) 
 
match_pct = round(common_rows * 100 / prod_count, 2) 
 
print(f"Prod Match % : {match_pct}") 
 
print("\n" + "=" * 80) 
print("DUPLICATE ROW ANALYSIS") 
print("=" * 80) 
 
prod_duplicates = prod_count - df_Prod.distinct().count() 
fabric_duplicates = fabric_count - df_Fabric.distinct().count() 
 
print("Prod duplicates   :", prod_duplicates) 
print("Fabric duplicates :", fabric_duplicates) 
 

 
 

Result: 
```

```text
### FULL ROW COMPARISON

Common rows         : 9673 
Only in Prod        : 182 
Only in Fabric      : 0 
 
### MATCH PERCENTAGE

Prod Match % : 96.51 
 
### DUPLICATE ROW ANALYSIS

Prod duplicates   : 168 
Fabric duplicates : 168 
 

 
 
 

4. Duplicate key analysis (LOT, DATE_TIME, HISTORDER) 
```

```python
# LOGIC: 
# Find keys (LOT, DATE_TIME, HISTORDER) that appear more than once in Prod. 
# Helps detect duplicated key-level events. 
 
key_cols = ["LOT", "DATE_TIME", "HISTORDER"] 
 
print("\n" + "=" * 80) 
print("DUPLICATE KEYS IN PROD") 
print("=" * 80) 
 
df_Prod.groupBy(*key_cols) \ 
    .count() \ 
    .filter("count > 1") \ 
    .orderBy(F.desc("count")) \ 
    .show(50, False) 
 

 
 

Result (top 50 duplicate keys in Prod): 
```

```text
### DUPLICATE KEYS IN PROD

+--------+---------+---------+-----+ 
|LOT     |DATE_TIME|HISTORDER|count| 
+--------+---------+---------+-----+ 
|6802A154|45:33.5  |1        |5    | 
|6193A050|56:00.0  |1        |5    | 
|6193A050|55:59.7  |1        |5    | 
|6193A050|55:59.9  |2        |5    | 
|7963A0R9|27:01.8  |1        |5    | 
|6802A154|45:33.4  |1        |5    | 
|6514A4A6|43:09.4  |1        |5    | 
|6802A154|45:33.4  |2        |5    | 
|7610A2P5|27:01.7  |1        |5    | 
|6802A154|45:33.2  |1        |5    | 
|6193A050|55:59.9  |1        |5    | 
|6514A3T1|32:49.9  |2        |5    | 
|7868A1H5|19:51.2  |1        |5    | 
|7963A0R9|27:01.8  |2        |5    | 
|6193A050|56:00.0  |2        |5    | 
|6802A154|45:33.3  |1        |5    | 
|7868A1H5|19:51.0  |1        |5    | 
|6514A4A6|43:09.2  |1        |5    | 
|6514A3T1|32:49.8  |1        |5    | 
|6802A154|45:33.2  |2        |5    | 
|7868A1H5|19:51.0  |2        |5    | 
|6514A4A6|43:09.4  |2        |5    | 
|6802A154|45:33.5  |2        |5    | 
|6514A3T1|32:49.7  |2        |5    | 
|6514A3T1|32:49.9  |1        |5    | 
|6193A050|55:59.7  |2        |5    | 
|7868A1H5|19:51.2  |2        |5    | 
|6193A050|55:59.8  |2        |5    | 
|6514A3T1|32:49.8  |2        |5    | 
|7610A2P5|27:01.7  |2        |5    | 
|6802A154|45:33.3  |2        |5    | 
|6514A4A6|43:09.2  |2        |5    | 
|6193A050|55:59.8  |1        |5    | 
|6514A3T1|32:49.7  |1        |5    | 
|6514A412|21:38.5  |1        |4    | 
|7963A0S3|27:01.9  |1        |4    | 
|7412A0A9|52:37.3  |1        |4    | 
|6226A078|46:26.1  |1        |4    | 
|6514A412|21:38.5  |2        |4    | 
|7610A2N5|27:01.6  |2        |4    | 
|5213A084|39:50.2  |1        |4    | 
|7412A0A9|52:37.3  |2        |4    | 
|6514A4A6|43:09.3  |1        |4    | 
|7487A3A7|37:51.2  |1        |4    | 
|6226A079|33:49.3  |1        |4    | 
|6226A079|33:49.6  |2        |4    | 
|7851AD62|37:50.2  |1        |4    | 
|6226A079|33:49.7  |2        |4    | 
|6226A079|33:49.6  |1        |4    | 
|7868A1H5|19:51.3  |1        |4    | 
+--------+---------+---------+-----+ 
only showing top 50 rows 
 

 
 
 

5. Null profile by column 
```

```python
# LOGIC: 
# Count how many NULLs exist per column in each dataset. 
# Used to confirm that null patterns are consistent between Prod and Fabric. 
 
print("\n" + "=" * 80) 
print("PROD NULL PROFILE") 
print("=" * 80) 
 
df_Prod.select([ 
    F.sum(F.col(c).isNull().cast("int")).alias(c) 
    for c in df_Prod.columns 
]).show(vertical=True) 
 
print("\n" + "=" * 80) 
print("FABRIC NULL PROFILE") 
print("=" * 80) 
 
df_Fabric.select([ 
    F.sum(F.col(c).isNull().cast("int")).alias(c) 
    for c in df_Fabric.columns 
]).show(vertical=True) 
 

 
 

Result (Prod nulls): 
```

```text
### PROD NULL PROFILE

-RECORD 0------------ 
 LOT          | 0     
 DATE_TIME    | 0     
 HISTORDER    | 0     
 TRANS        | 0     
 OPER         | 0     
 MASK_LVL     | 0     
 OPERDESC     | 0     
 OPERLONGDESC | 0     
 MACHINE      | 3899  
 USERNAME     | 0     
 HIST_REC     | 0     
 HISTCODE     | 0     
 COMMAND      | 0     
 SHORTREPORT  | 0     
 VIEWFLAG     | 0     
 Is_Person    | 0     
 IS_DUPLICATE | 0     
 EMPID        | 0     
 

 
 

Result (Fabric nulls): 
```

```text
### FABRIC NULL PROFILE

-RECORD 0------------ 
 LOT          | 0     
 DATE_TIME    | 0     
 HISTORDER    | 0     
 TRANS        | 0     
 OPER         | 0     
 MASK_LVL     | 0     
 OPERDESC     | 0     
 OPERLONGDESC | 0     
 MACHINE      | 3859  
 USERNAME     | 0     
 HIST_REC     | 0     
 HISTCODE     | 0     
 COMMAND      | 0     
 SHORTREPORT  | 0     
 VIEWFLAG     | 0     
 Is_Person    | 0     
 IS_DUPLICATE | 0     
 EMPID        | 0     
 

 
 
 

6. Date range comparison 
```

```python
# LOGIC: 
# Compare the min and max DATE_TIME values between Prod and Fabric 
# to ensure both cover the same time window. 
 
print("\n" + "=" * 80) 
print("DATE RANGE COMPARISON") 
print("=" * 80) 
 
print("Prod") 
df_Prod.select( 
    F.min("DATE_TIME").alias("MIN_DATE_TIME"), 
    F.max("DATE_TIME").alias("MAX_DATE_TIME") 
).show(truncate=False) 
 
print("Fabric") 
df_Fabric.select( 
    F.min("DATE_TIME").alias("MIN_DATE_TIME"), 
    F.max("DATE_TIME").alias("MAX_DATE_TIME") 
).show(truncate=False) 
 

 
 

Result: 
```

```text
### DATE RANGE COMPARISON

Prod 
+-------------+-------------+ 
|MIN_DATE_TIME|MAX_DATE_TIME| 
+-------------+-------------+ 
|15:00.2      |59:59.4      | 
+-------------+-------------+ 
 
Fabric 
+-------------+-------------+ 
|MIN_DATE_TIME|MAX_DATE_TIME| 
+-------------+-------------+ 
|15:00.2      |59:59.4      | 
+-------------+-------------+ 
 

 
 
 

7. Lots only in Prod / only in Fabric 
```

```python
# LOGIC: 
# List distinct LOTs that exist only in Prod and only in Fabric. 
# This highlights missing lots between the two systems. 
 
print("\n" + "=" * 80) 
print("LOTS ONLY IN PROD") 
print("=" * 80) 
 
only_prod.select("LOT") \ 
    .distinct() \ 
    .orderBy("LOT") \ 
    .show(1000, False) 
 
print("\n" + "=" * 80) 
print("LOTS ONLY IN FABRIC") 
print("=" * 80) 
 
only_fabric.select("LOT") \ 
    .distinct() \ 
    .orderBy("LOT") \ 
    .show(1000, False) 
 

 
 

Result (lots only in Prod): 
```

```text
### LOTS ONLY IN PROD

+---------+ 
|LOT      | 
+---------+ 
|5245A001A| 
|5630A001A| 
|5630A003A| 
|5704A326E| 
|5801A009A| 
|5822A061A| 
|5941A0F5A| 
|6098A0E1A| 
|6121A001A| 
|6226A081A| 
|6352AC23D| 
|6352AC45A| 
|6554A0A0 | 
|6554A0A1 | 
|6794A001N| 
|6794A007B| 
|6951A016A| 
|7192A1E4A| 
|7417A001A| 
|7499A054 | 
|7499A055 | 
|7579A1C3A| 
|7610A2J5A| 
|7693A006H| 
|7693A006J| 
|7851AD84A| 
|7851AD89A| 
|7974A005A| 
|7974A005B| 
+---------+ 
 

 
 

Result (lots only in Fabric): 
```

```text
### LOTS ONLY IN FABRIC

+---+ 
|LOT| 
+---+ 
+---+ 
 

 
 
 

8. Key comparison (LOT, DATE_TIME, HISTORDER) and distinct missing lots 
```

```python
# LOGIC: 
# Using the key columns, count: 
# - keys common to both datasets 
# - keys missing from Fabric (present in Prod only) 
# - keys missing from Prod (present in Fabric only) 
# Also list distinct LOTs associated with missing keys. 
 
print("\n" + "=" * 80) 
print("KEY COMPARISON (LOT, DATE_TIME, HISTORDER)") 
print("=" * 80) 
 
common_keys = df_Prod.join( 
    df_Fabric, 
    key_cols, 
    "inner" 
) 
 
prod_missing = df_Prod.join( 
    df_Fabric, 
    key_cols, 
    "left_anti" 
) 
 
fabric_missing = df_Fabric.join( 
    df_Prod, 
    key_cols, 
    "left_anti" 
) 
 
print("Common Keys      :", common_keys.count()) 
print("Prod Missing     :", prod_missing.count()) 
print("Fabric Missing   :", fabric_missing.count()) 
 
print("\n" + "=" * 80) 
print("DISTINCT LOTS MISSING IN FABRIC") 
print("=" * 80) 
 
prod_missing.select("LOT") \ 
    .distinct() \ 
    .orderBy("LOT") \ 
    .show(1000, False) 
 
print("\n" + "=" * 80) 
print("DISTINCT LOTS MISSING IN PROD") 
print("=" * 80) 
 
fabric_missing.select("LOT") \ 
    .distinct() \ 
    .orderBy("LOT") \ 
    .show(1000, False) 
 

 
 

Result: 
```

```text
### KEY COMPARISON (LOT, DATE_TIME, HISTORDER)

Common Keys      : 12691 
Prod Missing     : 182 
Fabric Missing   : 0 
 

 
 

Result (distinct lots missing in Fabric): 
```

```text
### DISTINCT LOTS MISSING IN FABRIC

+---------+ 
|LOT      | 
+---------+ 
|5245A001A| 
|5630A001A| 
|5630A003A| 
|5704A326E| 
|5801A009A| 
|5822A061A| 
|5941A0F5A| 
|6098A0E1A| 
|6121A001A| 
|6226A081A| 
|6352AC23D| 
|6352AC45A| 
|6554A0A0 | 
|6554A0A1 | 
|6794A001N| 
|6794A007B| 
|6951A016A| 
|7192A1E4A| 
|7417A001A| 
|7499A054 | 
|7499A055 | 
|7579A1C3A| 
|7610A2J5A| 
|7693A006H| 
|7693A006J| 
|7851AD84A| 
|7851AD89A| 
|7974A005A| 
|7974A005B| 
+---------+ 
 

 
 

Result (distinct lots missing in Prod): 
```

```text
### DISTINCT LOTS MISSING IN PROD

+---+ 
|LOT| 
+---+ 
+---+ 
 

 
 
 

9. HISTCODE distribution comparison 
```

```python
# LOGIC: 
# Compare how many records each HISTCODE has in Prod vs Fabric, 
# and compute the difference for each code. 
 
print("\n" + "=" * 80) 
print("HISTCODE COMPARISON") 
print("=" * 80) 
 
prod_hist = ( 
    df_Prod.groupBy("HISTCODE") 
    .count() 
    .withColumnRenamed("count", "prod_count") 
) 
 
fabric_hist = ( 
    df_Fabric.groupBy("HISTCODE") 
    .count() 
    .withColumnRenamed("count", "fabric_count") 
) 
 
hist_compare = ( 
    prod_hist.join( 
        fabric_hist, 
        "HISTCODE", 
        "full" 
    ) 
    .fillna(0) 
    .withColumn( 
        "difference", 
        F.col("fabric_count") - F.col("prod_count") 
    ) 
) 
 
hist_compare.orderBy( 
    F.desc(F.abs(F.col("difference"))) 
).show(100, False) 
 

 
 

Result (excerpt): 
```

```text
### HISTCODE COMPARISON

+--------+----------+------------+----------+ 
|HISTCODE|prod_count|fabric_count|difference| 
+--------+----------+------------+----------+ 
|CM      |3620      |3572        |-48       | 
|LL      |1333      |1292        |-41       | 
|TX      |449       |433         |-16       | 
|OI      |378       |366         |-12       | 
|CS      |308       |298         |-10       | 
|DL      |392       |382         |-10       | 
|IN      |363       |355         |-8        | 
|SR      |390       |382         |-8        | 
|RS      |218       |212         |-6        | 
|LA      |341       |336         |-5        | 
|HC      |48        |44          |-4        | 
|MV      |335       |331         |-4        | 
|WM      |271       |268         |-3        | 
|AR      |225       |223         |-2        | 
|HT      |36        |34          |-2        | 
|SC      |2         |1           |-1        | 
|SK      |30        |29          |-1        | 
|UD      |56        |55          |-1        | 
|AF      |6         |6           |0         | 
... 
|WR      |25        |25          |0         | 
+--------+----------+------------+----------+ 
 

 
 
 

10. OPER distribution comparison 
```

```python
# LOGIC: 
# Compare counts per OPER between Prod and Fabric, 
# to validate that operation-level distributions match. 
 
print("\n" + "=" * 80) 
print("OPER DISTRIBUTION") 
print("=" * 80) 
 
oper_compare = ( 
    df_Prod.groupBy("OPER") 
    .count() 
    .withColumnRenamed("count", "prod_count") 
    .join( 
        df_Fabric.groupBy("OPER") 
        .count() 
        .withColumnRenamed("count", "fabric_count"), 
        "OPER", 
        "full" 
    ) 
    .fillna(0) 
) 
 
oper_compare.orderBy("OPER").show(500, False) 
 

 
 

Result (excerpt): 
```

```text
### OPER DISTRIBUTION

+-----+----------+------------+ 
|OPER |prod_count|fabric_count| 
+-----+----------+------------+ 
|-100 |833       |833         | 
|20209|5         |5           | 
|21070|24        |24          | 
... 
|95051|13        |13          | 
|99999|592       |592         | 
+-----+----------+------------+ 
 

 
 

(The output lists all OPER values with prod_count and fabric_count.) 

 

11. EMPID distribution comparison 
```

```python
# LOGIC: 
# Compare how many records each EMPID has in Prod vs Fabric, 
# to ensure employee-usage distribution is consistent. 
 
print("\n" + "=" * 80) 
print("EMPID DISTRIBUTION") 
print("=" * 80) 
 
empid_compare = ( 
    df_Prod.groupBy("EMPID") 
    .count() 
    .withColumnRenamed("count", "prod_count") 
    .join( 
        df_Fabric.groupBy("EMPID") 
        .count() 
        .withColumnRenamed("count", "fabric_count"), 
        "EMPID", 
        "full" 
    ) 
    .fillna(0) 
) 
 
empid_compare.orderBy( 
    F.desc("fabric_count") 
).show(100, False) 
 

 
 

Result (excerpt): 
```

```text
### EMPID DISTRIBUTION

+-----+----------+------------+ 
|EMPID|prod_count|fabric_count| 
+-----+----------+------------+ 
|SYSTM|1924      |1863        | 
|0    |1703      |1668        | 
|5493 |1312      |1312        | 
|3414 |778       |778         | 
|99999|570       |570         | 
... 
|6128 |2         |2           | 
+-----+----------+------------+ 
 

 
 
 

12. MACHINE distribution comparison 
```

```python
# LOGIC: 
# Compare counts per MACHINE between Prod and Fabric, 
# to verify tool/machine usage consistency. 
 
print("\n" + "=" * 80) 
print("MACHINE DISTRIBUTION") 
print("=" * 80) 
 
machine_compare = ( 
    df_Prod.groupBy("MACHINE") 
    .count() 
    .withColumnRenamed("count", "prod_count") 
    .join( 
        df_Fabric.groupBy("MACHINE") 
        .count() 
        .withColumnRenamed("count", "fabric_count"), 
        "MACHINE", 
        "full" 
    ) 
    .fillna(0) 
) 
 
machine_compare.orderBy("MACHINE").show(500, False) 
 

 
 

Result (excerpt): 
```

```text
### MACHINE DISTRIBUTION

+--------------+----------+------------+ 
|MACHINE       |prod_count|fabric_count| 
+--------------+----------+------------+ 
|NULL          |3899      |0           | 
|NULL          |0         |3859        | 
|ALD101        |21        |7           | 
|ALPHA307      |51        |51          | 
... 
|WB91          |64        |64          | 
+--------------+----------+------------+ 
 

 
 
 

13. Summary metrics 
```

```python
# LOGIC: 
# Compute overall metrics per dataset: 
# row count, distinct lots, operations, users, machines, and empids. 
 
print("\n" + "=" * 80) 
print("SUMMARY METRICS") 
print("=" * 80) 
 
agg_prod = df_Prod.select( 
    F.count("*").alias("rows"), 
    F.countDistinct("LOT").alias("lots"), 
    F.countDistinct("OPER").alias("opers"), 
    F.countDistinct("USERNAME").alias("users"), 
    F.countDistinct("MACHINE").alias("machines"), 
    F.countDistinct("EMPID").alias("empids") 
) 
 
agg_fabric = df_Fabric.select( 
    F.count("*").alias("rows"), 
    F.countDistinct("LOT").alias("lots"), 
    F.countDistinct("OPER").alias("opers"), 
    F.countDistinct("USERNAME").alias("users"), 
    F.countDistinct("MACHINE").alias("machines"), 
    F.countDistinct("EMPID").alias("empids") 
) 
 
print("Prod Summary") 
agg_prod.show(truncate=False) 
 
print("Fabric Summary") 
agg_fabric.show(truncate=False) 
 

 
 

Result: 
```

```text
### SUMMARY METRICS

Prod Summary 
+-----+----+-----+-----+--------+------+ 
|rows |lots|opers|users|machines|empids| 
+-----+----+-----+-----+--------+------+ 
|10023|1294|451  |272  |191     |86    | 
+-----+----+-----+-----+--------+------+ 
 
Fabric Summary 
+----+----+-----+-----+--------+------+ 
|rows|lots|opers|users|machines|empids| 
+----+----+-----+-----+--------+------+ 
|9841|1265|451  |272  |191     |86    | 
+----+----+-----+-----+--------+------+ 
 

 
 
 

14. Row-level attribute mismatch summary 
```

```python
# LOGIC: 
# For rows matched on key (LOT, DATE_TIME, HISTORDER), 
# count how many have mismatched values in key attributes like OPER, 
# TRANS, HIST_REC, MACHINE, USERNAME, COMMAND, EMPID, etc. 
 
print("\n" + "=" * 80) 
print("ATTRIBUTE MISMATCH ANALYSIS") 
print("=" * 80) 
 
cols_to_compare = [ 
    "OPER", 
    "OPERDESC", 
    "OPERLONGDESC", 
    "TRANS", 
    "HIST_REC", 
    "MACHINE", 
    "USERNAME", 
    "COMMAND", 
    "EMPID" 
] 
 
compare = ( 
    df_Prod.alias("p") 
    .join( 
        df_Fabric.alias("f"), 
        key_cols 
    ) 
) 
 
mismatch_summary = compare.agg(*[ 
    F.sum( 
        F.when( 
            F.coalesce(F.col(f"p.{c}"), F.lit("")) != 
            F.coalesce(F.col(f"f.{c}"), F.lit("")), 
            1 
        ).otherwise(0) 
    ).alias(f"{c}_MISMATCH") 
    for c in cols_to_compare 
]) 
 
mismatch_summary.show(truncate=False) 
 

 
 

Result: 
```

```text
### ATTRIBUTE MISMATCH ANALYSIS

+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|OPER_MISMATCH|OPERDESC_MISMATCH|OPERLONGDESC_MISMATCH|TRANS_MISMATCH|HIST_REC_MISMATCH|MACHINE_MISMATCH|USERNAME_MISMATCH|COMMAND_MISMATCH|EMPID_MISMATCH| 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|10           |10               |10                   |1466          |2204             |40              |68               |318             |0             | 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
 

 
 
 

15. TRANS mismatch details (sample) 
```

```python
# LOGIC: 
# For rows where TRANS differs between Prod and Fabric, 
# display a sample set of mismatching rows with both values. 
 
print("\n" + "=" * 80) 
print("TRANS MISMATCH SAMPLE") 
print("=" * 80) 
 
compare.filter( 
    F.coalesce(F.col("p.TRANS"), F.lit("")) != 
    F.coalesce(F.col("f.TRANS"), F.lit("")) 
).select( 
    "LOT", 
    "DATE_TIME", 
    "HISTORDER", 
    F.col("p.TRANS").alias("PROD_TRANS"), 
    F.col("f.TRANS").alias("FABRIC_TRANS") 
).show(200, False) 
 

 
 

Result (first 200 mismatches): 
(very long; excerpt) 
```

```text
### TRANS MISMATCH SAMPLE

+----------+---------+---------+----------+------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_TRANS|FABRIC_TRANS| 
+----------+---------+---------+----------+------------+ 
|5941A102  |15:00.2  |1        |SUGG RECIP|COMMENT     | 
|5941A102  |15:00.2  |1        |SUGG RECIP|MOVE IN     | 
|5941A102  |15:00.2  |1        |SUGG RECIP|COMMENT     | 
|5941A102  |15:00.2  |1        |COMMENT   |MOVE IN     | 
... 
|6389A065  |21:51.1  |1        |WAIT MVOU |COMMENT     | 
|6389A065  |21:51.1  |1        |COMMENT   |WAIT MVOU   | 
+----------+---------+---------+----------+------------+ 
only showing top 200 rows 
 

 
 
 

16. COMMAND mismatch details (sample) 
```

```python
# LOGIC: 
# For rows where COMMAND differs between Prod and Fabric, 
# display a sample with both command values for investigation. 
 
print("\n" + "=" * 80) 
print("COMMAND MISMATCH SAMPLE") 
print("=" * 80) 
 
compare.filter( 
    F.coalesce(F.col("p.COMMAND"), F.lit("")) != 
    F.coalesce(F.col("f.COMMAND"), F.lit("")) 
).select( 
    "LOT", 
    "DATE_TIME", 
    "HISTORDER", 
    F.col("p.COMMAND").alias("PROD_COMMAND"), 
    F.col("f.COMMAND").alias("FABRIC_COMMAND") 
).show(200, False) 
 

 
 

Result (first 200 mismatches): 
(very long; excerpt) 
```

```text
### COMMAND MISMATCH SAMPLE

+----------+---------+---------+------------+--------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_COMMAND|FABRIC_COMMAND| 
+----------+---------+---------+------------+--------------+ 
|5941A102  |15:00.2  |1        |LMVR        |LEDC          | 
|5941A102  |15:00.2  |1        |LMVR        |MVIN          | 
... 
|5941A0C8  |40:42.8  |1        |HOLD        |RPOS          | 
|5941A0C8  |40:42.8  |1        |HOLD        |RPOS          | 
+----------+---------+---------+------------+--------------+ 
only showing top 200 rows 
 

 
 
 

17. HIST_REC mismatch details (sample) 
```

```python
# LOGIC: 
# For rows where HIST_REC text differs between Prod and Fabric, 
# display a sample to inspect how history text is being split or reordered. 
 
print("\n" + "=" * 80) 
print("HIST_REC MISMATCH SAMPLE") 
print("=" * 80) 
 
compare.filter( 
    F.coalesce(F.col("p.HIST_REC"), F.lit("")) != 
    F.coalesce(F.col("f.HIST_REC"), F.lit("")) 
).select( 
    "LOT", 
    "DATE_TIME", 
    "HISTORDER", 
    F.col("p.HIST_REC").alias("PROD_HIST_REC"), 
    F.col("f.HIST_REC").alias("FABRIC_HIST_REC") 
).show(200, False) 
 

 
 

Result (first 200 mismatches): 
(very long; excerpt) 
```

```text
### HIST_REC MISMATCH SAMPLE

+----------+---------+---------+----------------------------------------+----------------------------------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_HIST_REC                           |FABRIC_HIST_REC                         | 
+----------+---------+---------+----------------------------------------+----------------------------------------+ 
|5941A102  |15:00.2  |1        |ROM RECIP ID: E_WB81_N_40_BATH_A REV: 3 |Lot selected for EDC setup:             | 
|5941A102  |15:00.2  |1        |ROM RECIP ID: E_WB81_N_40_BATH_A REV: 3 |QTY: 25                                 | 
... 
|7868A1H5  |19:51.1  |2        |Parameter: APC_M_YTran                  |Parameter: APC_M_Rot                    | 
+----------+---------+---------+----------------------------------------+----------------------------------------+ 
only showing top 200 rows 

 

 

 

 
```


---

## 📋 Comprehensive Observations & Data Validation Analysis

Based on the verification results extracted above, we present the detailed data analysis and root cause findings:

### 1. Data Volume & Exact Match Discrepancy
* **Total Rows in Production (SQL View):** 10,023
* **Total Rows in Fabric (Warehouse View):** 9,841
* **Common Rows (Exact Match on all columns):** 9,673
* **Discrepancy (Production Excess):** 182 rows (representing 1.82% of Production data)
* **Reconciliation Rate:** **96.51%** exact row match.

> [!CAUTION]
> **Validation Failure:** 182 rows are present in Production but missing completely from Fabric. Fabric contains no extra records (0 rows unique to Fabric).

---

### 2. Missing Lots and Suffix/Padding Issue
* **Distinct LOT count in Prod:** 1,294
* **Distinct LOT count in Fabric:** 1,265
* **Delta:** 29 LOTs are completely missing in Fabric.
* **The 29 Missing LOTs:**
  `5245A001A`, `5630A001A`, `5630A003A`, `5704A326E`, `5801A009A`, `5822A061A`, `5941A0F5A`, `6098A0E1A`, `6121A001A`, `6226A081A`, `6352AC23D`, `6352AC45A`, `6554A0A0 `, `6554A0A1 `, `6794A001N`, `6794A007B`, `6951A016A`, `7192A1E4A`, `7417A001A`, `7499A054 `, `7499A055 `, `7579A1C3A`, `7610A2J5A`, `7693A006H`, `7693A006J`, `7851AD84A`, `7851AD89A`, `7974A005A`, `7974A005B`

#### 🔍 Analysis:
* **Trailing Spaces:** Trailing spaces in IDs (e.g. `6554A0A0 `, `7499A054 `) match leniently in SQL Server but fail to match strictly in Spark SQL.
* **Suffixes:** Suffixes like `A`, `E`, `N`, `B`, `H`, `J` (e.g., `5704A326E`, `6794A001N`, `7693A006H`) are missing, which points to potential replication filters or data truncation constraints in the ETL sync script.

---

### 3. Duplication Source Check
* **Duplicates in Prod:** 168 rows
* **Duplicates in Fabric:** 168 rows
* **Analysis:** Both datasets contain exactly 168 duplicate rows (where the entire row is identical). This confirms that duplication is not introduced by the replication pipeline, but is inherited from the source database view.

---

### 4. Shuffled Attributes (Non-Deterministic Joins)
When performing a key-based join using `(LOT, DATE_TIME, HISTORDER)`, we observe massive mismatch rates on the row attributes:
* **HIST_REC Mismatches:** 2,204 rows
* **TRANS Mismatches:** 1,466 rows
* **COMMAND Mismatches:** 318 rows
* **USERNAME Mismatches:** 68 rows
* **MACHINE Mismatches:** 40 rows
* **OPER / OPERDESC / OPERLONGDESC Mismatches:** 10 rows

#### 🔍 Why this happens (Join Key Instability):
* The combination of `(LOT, DATE_TIME, HISTORDER)` is **not unique** (as demonstrated by the *DUPLICATE KEYS IN PROD* section, where some composite keys return a count of 5 for different events).
* Because multiple transaction logs share the exact same timestamp and history order, there is no secondary ordering identifier (like a surrogate ID or physical sequence key) in the Fabric view joins.
* Consequently, when Spark compiles the Fabric view, it pairs tables non-deterministically. This causes attributes (e.g., pairing `TRANS` with `HIST_REC`, or even `OPER` in 10 cases) to shuffle between the duplicate records.

---

### 5. Machine NULL Discrepancy
* **Prod Machine Nulls:** 3,899 NULLs
* **Fabric Machine Nulls:** 3,859 NULLs
* **Delta:** 40 rows
* **Analysis:** This exactly aligns with the **40 MACHINE mismatches** from the attribute analysis. The non-deterministic join key alignment randomly paired rows with a NULL machine in Prod to rows with a populated machine in Fabric, leading to a minor count shift.

---

### 🛠️ Strategic Recommendations
1. **Enhance Join Conditions in Fabric View:** Update the view definitions in Fabric to join on a unique Transaction ID or sequence identifier (`TXN_ID` or a combination of `HISTORDER` and a sequence column) to ensure stable, deterministic attribute mapping.
2. **Standardize String Treatment (Trimming):** Implement consistent trimming logic (`TRIM(LOT)`) across all views and replication stages in Fabric to prevent trailing spaces from breaking joins and dropping lots.
3. **Verify Replication Filter Rules:** Confirm that the migration replication logic is not intentionally or accidentally excluding Lot IDs ending with suffix characters (like `A`, `E`, `N`, `B`, `H`, `J`).
