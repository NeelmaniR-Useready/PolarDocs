Production DB view –[TrainingVision].[dbo.LOTHISTV] 



Fabric view –[Polar_Warehouse].[TrainingVision].[LotHistV] 



 

Time stamp: DATETIME >= ‘2026-08-10 04:00:00’ AND 
	DATE_TIME < ‘2026-08-10 05:00:00’ 

 

```python
# LOGIC: 
# Load two CSV extracts (Prod and Fabric) from the Lakehouse for the given time window. 
# Then compute and print total row counts for each dataset to see basic volume alignment. 
 
df_Prod = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Prod-08-10(04-05).csv", 
    header=True, 
    inferSchema=True 
) 
 
df_Fabric = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Fabric-08-10(04-05).csv", 
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
 

 
 
```

```text
### ROW COUNTS

Prod Rows   : 9421 
Fabric Rows : 9224 
 

 
 
 
```

```python
# LOGIC: 
# For each column in Prod and Fabric, compute the distinct count. 
# This profiles cardinality per column and helps spot missing/extra variety between systems. 
 
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
 

 
 
```

```text
### PROD DISTINCT COUNTS

-RECORD 0------------ 
 LOT          | 1126  
 DATE_TIME    | 3229  
 HISTORDER    | 140   
 TRANS        | 48    
 OPER         | 465   
 MASK_LVL     | 63    
 OPERDESC     | 159   
 OPERLONGDESC | 563   
 MACHINE      | 194   
 USERNAME     | 248   
 HIST_REC     | 1449  
 HISTCODE     | 48    
 COMMAND      | 22    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 63    
 
### FABRIC DISTINCT COUNTS

-RECORD 0------------ 
 LOT          | 1104  
 DATE_TIME    | 3229  
 HISTORDER    | 140   
 TRANS        | 48    
 OPER         | 465   
 MASK_LVL     | 63    
 OPERDESC     | 159   
 OPERLONGDESC | 563   
 MACHINE      | 194   
 USERNAME     | 248   
 HIST_REC     | 1449  
 HISTCODE     | 48    
 COMMAND      | 22    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 63    
 

 
 
 
```

```python
# LOGIC: 
# Do a full row-level comparison: 
#  - common_rows  : number of identical rows in both Prod and Fabric 
#  - only_prod    : rows that appear only in Prod 
#  - only_fabric  : rows that appear only in Fabric 
# Then compute match % (common_rows vs Prod volume) and count duplicate rows in each dataset. 
 
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
 

 
 
```

```text
### FULL ROW COMPARISON

Common rows         : 9084 
Only in Prod        : 197 
Only in Fabric      : 0 
 
### MATCH PERCENTAGE

Prod Match % : 96.42 
 
### DUPLICATE ROW ANALYSIS

Prod duplicates   : 140 
Fabric duplicates : 140 
 

 
 
 
```

```python
# LOGIC: 
# Identify key-level duplicates in Prod by (LOT, DATE_TIME, HISTORDER). 
# These represent multiple records for the same logical event/key in Prod. 
 
key_cols = ["LOT", "DATE_TIME", "HISTORDER"] 
 
print("\n" + "=" * 80) 
print("DUPLICATE KEYS IN PROD") 
print("=" * 80) 
 
df_Prod.groupBy(*key_cols) \ 
    .count() \ 
    .filter("count > 1") \ 
    .orderBy(F.desc("count")) \ 
    .show(50, False) 
 

 
 
```

```text
### DUPLICATE KEYS IN PROD

+--------+---------+---------+-----+ 
|LOT     |DATE_TIME|HISTORDER|count| 
+--------+---------+---------+-----+ 
|6515A0R7|02:25.5  |2        |5    | 
|6935A1D5|21:13.8  |2        |5    | 
|6339A2K4|24:36.4  |2        |5    | 
|6514A4H4|40:39.4  |1        |5    | 
|6514A4H4|40:39.7  |2        |5    | 
|6514A3V0|43:58.6  |1        |5    | 
|6515A0R7|02:25.5  |1        |5    | 
|6339A2M5|36:33.7  |2        |5    | 
|6339A2M5|36:34.0  |2        |5    | 
|6514A3V0|43:58.4  |2        |5    | 
|6339A2L4|48:47.8  |2        |5    | 
|6339A2M5|36:33.9  |1        |5    | 
|6514A3V0|43:58.5  |2        |5    | 
|6514A3V0|43:58.4  |1        |5    | 
|6514A4H4|40:39.4  |2        |5    | 
|6514A4H4|40:39.5  |1        |5    | 
|6339A2L4|48:47.8  |1        |5    | 
|6339A2L4|48:48.0  |1        |5    | 
|6514A3V0|43:58.5  |1        |5    | 
|6339A2L4|48:47.9  |2        |5    | 
|6514A453|33:53.8  |1        |5    | 
|6339A2M5|36:33.9  |2        |5    | 
|6514A4H4|40:39.7  |1        |5    | 
|6339A2M5|36:33.7  |1        |5    | 
|6339A2L4|48:48.0  |2        |5    | 
|6935A1D5|21:13.8  |1        |5    | 
|6515A0R7|02:25.3  |2        |5    | 
|6514A453|33:53.8  |2        |5    | 
|6339A2K4|24:36.4  |1        |5    | 
|6514A3V0|43:58.6  |2        |5    | 
|6515A0R7|02:25.3  |1        |5    | 
|6339A2M5|36:34.0  |1        |5    | 
|6514A4H4|40:39.5  |2        |5    | 
|6339A2L4|48:47.9  |1        |5    | 
|6935A1D5|21:13.9  |2        |4    | 
|6514A472|33:53.6  |2        |4    | 
|6515A0R7|02:25.4  |1        |4    | 
|6515A0R7|02:25.2  |1        |4    | 
|6339A2M5|36:33.8  |2        |4    | 
|6515A0R7|02:25.4  |2        |4    | 
|6339A2M5|36:33.8  |1        |4    | 
|6514A4H4|40:39.6  |1        |4    | 
|6514A472|33:53.6  |1        |4    | 
|6515A0T7|29:56.4  |1        |4    | 
|6514A3V0|43:58.7  |2        |4    | 
|6515A0R7|02:25.2  |2        |4    | 
|6514A3V0|43:58.7  |1        |4    | 
|7412A0C1|55:55.1  |2        |4    | 
|6514A3V0|43:58.8  |2        |4    | 
|6339A2L4|48:47.7  |2        |4    | 
+--------+---------+---------+-----+ 
only showing top 50 rows 
 

 
 
 
```

```python
# LOGIC: 
# Compute null counts per column for Prod and Fabric. 
# This shows completeness/nullable patterns and highlights any differences. 
 
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
 MACHINE      | 2905  
 USERNAME     | 0     
 HIST_REC     | 0     
 HISTCODE     | 0     
 COMMAND      | 0     
 SHORTREPORT  | 0     
 VIEWFLAG     | 0     
 Is_Person    | 0     
 IS_DUPLICATE | 0     
 EMPID        | 0     
 
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
 MACHINE      | 2879  
 USERNAME     | 0     
 HIST_REC     | 0     
 HISTCODE     | 0     
 COMMAND      | 0     
 SHORTREPORT  | 0     
 VIEWFLAG     | 0     
 Is_Person    | 0     
 IS_DUPLICATE | 0     
 EMPID        | 0     
 

 
 
 
```

```python
# LOGIC: 
# Compare date/time coverage across the two extracts. 
# This validates that both Prod and Fabric span the same DATE_TIME range. 
 
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
 

 
 
```

```text
### DATE RANGE COMPARISON

Prod 
+-------------+-------------+ 
|MIN_DATE_TIME|MAX_DATE_TIME| 
+-------------+-------------+ 
|00:00.2      |59:58.8      | 
+-------------+-------------+ 
 
Fabric 
+-------------+-------------+ 
|MIN_DATE_TIME|MAX_DATE_TIME| 
+-------------+-------------+ 
|00:00.2      |59:58.8      | 
+-------------+-------------+ 
 

 
 
 
```

```python
# LOGIC: 
# List distinct LOT values that only appear in Prod or only in Fabric 
# based on full-row comparison (only_prod, only_fabric). 
 
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
 

 
 
```

```text
### LOTS ONLY IN PROD

+---------+ 
|LOT      | 
+---------+ 
|5245A001A| 
|5630A001A| 
|5630A003A| 
|5773A1B1A| 
|5801A009A| 
|6098A0E1A| 
|6121A001A| 
|6226A081A| 
|6352AC23D| 
|6352AC31A| 
|6554A0A0 | 
|6554A0A1 | 
|6786A191A| 
|7417A001A| 
|7417A002A| 
|7423A021A| 
|7499A057 | 
|7499A058 | 
|7582A001A| 
|7610A2J5A| 
|7851AD83A| 
|7851AD84A| 
+---------+ 
 
### LOTS ONLY IN FABRIC

+---+ 
|LOT| 
+---+ 
+---+ 
 

 
 
 
```

```python
# LOGIC: 
# Key-based comparison using (LOT, DATE_TIME, HISTORDER): 
#  - common_keys   : keys in both Prod and Fabric 
#  - prod_missing  : keys present in Prod but missing in Fabric 
#  - fabric_missing: keys present in Fabric but missing in Prod 
# Then list distinct LOTs associated with missing keys on either side. 
 
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
 

 
 
```

```text
### KEY COMPARISON (LOT, DATE_TIME, HISTORDER)

Common Keys      : 12292 
Prod Missing     : 197 
Fabric Missing   : 0 
 
### DISTINCT LOTS MISSING IN FABRIC

+---------+ 
|LOT      | 
+---------+ 
|5245A001A| 
|5630A001A| 
|5630A003A| 
|5773A1B1A| 
|5801A009A| 
|6098A0E1A| 
|6121A001A| 
|6226A081A| 
|6352AC23D| 
|6352AC31A| 
|6554A0A0 | 
|6554A0A1 | 
|6786A191A| 
|7417A001A| 
|7417A002A| 
|7423A021A| 
|7499A057 | 
|7499A058 | 
|7582A001A| 
|7610A2J5A| 
|7851AD83A| 
|7851AD84A| 
+---------+ 
 
### DISTINCT LOTS MISSING IN PROD

+---+ 
|LOT| 
+---+ 
+---+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare distribution of HISTCODE between Prod and Fabric: 
#  - prod_hist / fabric_hist : counts per HISTCODE 
#  - difference              : fabric_count - prod_count for each code. 
 
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
 

 
 
```

```text
### HISTCODE COMPARISON

+--------+----------+------------+----------+ 
|HISTCODE|prod_count|fabric_count|difference| 
+--------+----------+------------+----------+ 
|CM      |3117      |3066        |-51       | 
|LL      |1421      |1394        |-27       | 
|TX      |509       |495         |-14       | 
|WM      |349       |336         |-13       | 
|OI      |338       |326         |-12       | 
|SR      |405       |394         |-11       | 
|CS      |331       |321         |-10       | 
|IN      |385       |375         |-10       | 
|LA      |417       |407         |-10       | 
|MV      |412       |402         |-10       | 
|DL      |399       |390         |-9        | 
|RS      |226       |218         |-8        | 
|AR      |242       |235         |-7        | 
|UD      |80        |78          |-2        | 
|SE      |34        |33          |-1        | 
|SK      |38        |37          |-1        | 
|UX      |8         |7           |-1        | 
|AF      |3         |3           |0         | 
|AS      |7         |7           |0         | 
|CR      |7         |7           |0         | 
|DD      |7         |7           |0         | 
|DK      |160       |160         |0         | 
|EM      |8         |8           |0         | 
|EN      |8         |8           |0         | 
|HC      |46        |46          |0         | 
|HT      |7         |7           |0         | 
|KL      |79        |79          |0         | 
|KW      |79        |79          |0         | 
|LC      |34        |34          |0         | 
|OC      |3         |3           |0         | 
|OP      |2         |2           |0         | 
|OW      |7         |7           |0         | 
|PT      |101       |101         |0         | 
|R1      |1         |1           |0         | 
|R2      |1         |1           |0         | 
|RB      |2         |2           |0         | 
|RE      |7         |7           |0         | 
|RH      |2         |2           |0         | 
|RM      |4         |4           |0         | 
|RP      |23        |23          |0         | 
|RT      |86        |86          |0         | 
|RW      |1         |1           |0         | 
|SA      |1         |1           |0         | 
|SC      |2         |2           |0         | 
|TC      |1         |1           |0         | 
|TN      |7         |7           |0         | 
|VA      |7         |7           |0         | 
|WR      |7         |7           |0         | 
+--------+----------+------------+----------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare distribution of OPER between Prod and Fabric. 
# This validates that operation-level event counts are consistent. 
 
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
 

 
 
```

```text
### OPER DISTRIBUTION

+-----+----------+------------+ 
|OPER |prod_count|fabric_count| 
+-----+----------+------------+ 
|-100 |568       |568         | 
|21070|192       |192         | 
|22188|32        |32          | 
|22236|8         |8           | 
|22632|5         |5           | 
|22741|4         |4           | 
|24320|3         |3           | 
|26620|3         |3           | 
|30165|3         |3           | 
|31106|2         |2           | 
|40200|19        |19          | 
|40203|23        |23          | 
|40206|40        |40          | 
|40207|13        |13          | 
|40208|23        |23          | 
|40211|58        |58          | 
|40213|15        |12          | 
|40214|14        |14          | 
|40216|6         |6           | 
|40217|6         |6           | 
|40248|23        |23          | 
|40301|4         |4           | 
|40303|3         |3           | 
|40305|4         |4           | 
|40306|48        |48          | 
|40307|4         |4           | 
|40308|20        |20          | 
|40310|13        |13          | 
|40311|8         |8           | 
|40313|10        |5           | 
|40314|55        |55          | 
|40316|10        |10          | 
|40317|3         |3           | 
|40318|74        |74          | 
|40321|30        |30          | 
|40322|81        |81          | 
|40323|18        |18          | 
|40324|20        |20          | 
|40325|46        |46          | 
|40327|12        |12          | 
|40334|78        |78          | 
|40349|9         |9           | 
|40351|6         |6           | 
|40352|14        |14          | 
|40353|111       |111         | 
|40354|22        |22          | 
|40355|13        |13          | 
|40357|8         |8           | 
|40358|8         |8           | 
|40359|3         |3           | 
|40360|60        |60          | 
|40363|19        |19          | 
|40366|59        |59          | 
|40368|38        |38          | 
|40371|5         |5           | 
|40403|16        |16          | 
|40404|10        |10          | 
|40405|3         |3           | 
|40406|14        |14          | 
|40407|3         |3           | 
|40411|16        |16          | 
|40414|3         |3           | 
|40416|15        |15          | 
|40420|13        |13          | 
|40421|5         |5           | 
|40500|16        |16          | 
|40501|57        |54          | 
|40502|2         |2           | 
|40503|9         |9           | 
|40510|22        |22          | 
|40600|9         |9           | 
|40601|25        |25          | 
|40602|19        |19          | 
|40603|6         |6           | 
|40604|17        |15          | 
|40606|11        |11          | 
|40609|12        |12          | 
|40610|5         |5           | 
|40620|11        |11          | 
|40621|13        |13          | 
|40623|2         |2           | 
|40632|32        |32          | 
|40638|24        |12          | 
|40640|1         |1           | 
|45004|9         |9           | 
|45005|1         |1           | 
|45017|10        |10          | 
|45022|8         |8           | 
|49801|9         |9           | 
|49802|1         |1           | 
|49805|5         |5           | 
|49806|8         |8           | 
|49808|8         |8           | 
|49809|7         |7           | 
|49811|27        |27          | 
|49812|6         |6           | 
|49815|2         |2           | 
|49817|2         |2           | 
|49819|10        |10          | 
|49820|12        |12          | 
|49821|7         |7           | 
|49822|17        |17          | 
|49823|10        |10          | 
|49825|27        |27          | 
|49900|1         |1           | 
|49901|22        |22          | 
|49902|9         |9           | 
|49904|5         |5           | 
|49905|1         |1           | 
|49906|38        |38          | 
|49907|3         |3           | 
|49908|21        |21          | 
|49909|7         |7           | 
|49910|24        |24          | 
|49911|57        |57          | 
|49914|30        |30          | 
|49917|16        |16          | 
|49918|3         |3           | 
|49919|5         |5           | 
|49920|25        |25          | 
|49924|5         |5           | 
|49925|1         |1           | 
|49928|22        |22          | 
|49929|7         |7           | 
|49931|41        |41          | 
|49933|9         |9           | 
|49934|4         |4           | 
|49942|4         |4           | 
|49950|1         |1           | 
|49953|1         |1           | 
|49954|9         |9           | 
|49960|23        |19          | 
|49961|7         |7           | 
|49963|1         |1           | 
|49978|17        |17          | 
|49982|12        |12          | 
|49991|1         |1           | 
|50100|43        |43          | 
|50101|1         |1           | 
|50102|8         |8           | 
|50103|19        |19          | 
|50104|27        |27          | 
|50105|37        |37          | 
|50106|21        |21          | 
|50107|80        |80          | 
|50108|4         |4           | 
|50109|18        |18          | 
|50110|13        |13          | 
|50111|48        |45          | 
|50112|22        |22          | 
|50113|54        |54          | 
|50115|7         |7           | 
|50116|5         |5           | 
|50118|9         |9           | 
|50120|6         |6           | 
|50121|29        |29          | 
|50122|11        |11          | 
|50123|5         |5           | 
|50133|50        |50          | 
|50141|9         |9           | 
|50151|12        |12          | 
|50152|13        |13          | 
|50153|25        |25          | 
|50201|38        |38          | 
|50202|3         |3           | 
|50203|29        |22          | 
|50300|8         |8           | 
|50301|13        |13          | 
|50302|59        |59          | 
|50303|36        |36          | 
|50304|50        |50          | 
|50306|27        |27          | 
|50308|4         |4           | 
|50321|11        |11          | 
|50322|7         |7           | 
|50323|28        |28          | 
|50324|18        |18          | 
|50325|20        |20          | 
|50326|12        |12          | 
|50327|6         |6           | 
|50328|10        |10          | 
|50329|31        |31          | 
|50330|18        |18          | 
|50331|17        |17          | 
|50334|59        |59          | 
|50406|19        |19          | 
|59902|10        |10          | 
|59903|1         |1           | 
|59904|1         |1           | 
|59905|23        |23          | 
|59906|3         |3           | 
|59909|45        |45          | 
|59913|33        |33          | 
|59914|33        |33          | 
|59917|1         |1           | 
|59918|10        |10          | 
|60100|14        |14          | 
|60101|17        |17          | 
|60102|47        |47          | 
|60103|44        |44          | 
|60104|24        |24          | 
|60105|26        |26          | 
|60106|18        |18          | 
|60108|5         |5           | 
|60109|20        |10          | 
|60110|32        |32          | 
|60111|68        |68          | 
|60112|6         |6           | 
|60113|1         |1           | 
|60114|17        |17          | 
|60115|21        |21          | 
|60116|39        |39          | 
|60120|38        |38          | 
|60122|5         |5           | 
|62941|94        |94          | 
|62942|8         |8           | 
|64321|5         |5           | 
|64381|8         |8           | 
|70100|50        |46          | 
|70102|1         |1           | 
|70103|53        |42          | 
|70104|51        |51          | 
|70105|12        |12          | 
|70106|37        |37          | 
|70107|103       |100         | 
|70108|10        |10          | 
|70109|51        |40          | 
|70110|15        |15          | 
|70111|6         |6           | 
|70112|12        |12          | 
|70113|5         |5           | 
|70114|39        |39          | 
|70115|9         |9           | 
|70116|9         |9           | 
|70117|3         |3           | 
|70118|24        |17          | 
|70119|15        |15          | 
|70120|14        |14          | 
|70122|40        |40          | 
|70124|16        |16          | 
|70125|31        |31          | 
|70127|18        |18          | 
|70128|3         |3           | 
|70130|18        |18          | 
|70131|13        |13          | 
|70132|15        |15          | 
|70134|6         |6           | 
|70137|7         |7           | 
|70138|12        |12          | 
|70139|54        |54          | 
|70143|8         |8           | 
|70190|31        |31          | 
|70200|1         |1           | 
|70202|1         |1           | 
|70203|44        |44          | 
|70204|1         |1           | 
|70205|26        |26          | 
|70206|12        |12          | 
|70209|7         |7           | 
|70210|6         |6           | 
|70212|4         |4           | 
|70213|31        |31          | 
|70214|17        |17          | 
|70216|14        |14          | 
|70218|18        |18          | 
|70219|9         |9           | 
|70220|33        |33          | 
|70221|28        |28          | 
|70222|6         |6           | 
|70223|20        |20          | 
|70224|16        |16          | 
|70226|11        |11          | 
|70227|3         |3           | 
|70230|12        |12          | 
|70231|44        |44          | 
|70232|2         |2           | 
|70234|12        |6           | 
|70239|18        |9           | 
|70249|17        |17          | 
|70300|4         |4           | 
|70301|27        |27          | 
|70302|4         |4           | 
|70303|14        |14          | 
|70304|19        |19          | 
|70306|1         |1           | 
|70308|4         |4           | 
|70310|2         |2           | 
|70400|7         |7           | 
|70401|26        |26          | 
|70403|13        |13          | 
|70404|29        |29          | 
|70405|19        |19          | 
|70406|31        |31          | 
|70407|21        |21          | 
|70408|9         |9           | 
|70411|3         |3           | 
|70412|4         |4           | 
|70501|5         |5           | 
|70502|15        |15          | 
|70503|23        |19          | 
|70504|36        |36          | 
|70620|4         |4           | 
|70700|12        |12          | 
|70701|12        |12          | 
|70702|14        |14          | 
|70703|7         |7           | 
|70704|31        |31          | 
|70900|6         |6           | 
|70950|5         |5           | 
|71000|14        |14          | 
|71001|117       |117         | 
|71002|13        |13          | 
|71003|10        |10          | 
|71004|8         |8           | 
|71005|30        |30          | 
|71006|6         |6           | 
|71007|4         |2           | 
|71009|5         |5           | 
|71013|2         |2           | 
|71114|3         |3           | 
|71115|3         |3           | 
|71116|19        |19          | 
|71117|21        |21          | 
|71121|4         |4           | 
|71414|6         |6           | 
|71416|8         |8           | 
|71425|4         |2           | 
|71700|23        |23          | 
|75000|9         |9           | 
|75008|7         |7           | 
|75016|10        |10          | 
|75017|8         |8           | 
|75028|8         |8           | 
|79801|9         |9           | 
|79803|7         |7           | 
|79804|10        |10          | 
|79805|15        |15          | 
|79808|19        |19          | 
|79809|15        |15          | 
|79810|23        |23          | 
|79811|8         |8           | 
|79812|15        |15          | 
|79813|7         |7           | 
|79814|6         |6           | 
|79816|7         |7           | 
|79819|2         |2           | 
|79821|10        |10          | 
|79829|3         |3           | 
|79831|56        |44          | 
|79901|30        |30          | 
|79902|6         |6           | 
|79903|10        |10          | 
|79904|9         |9           | 
|79905|3         |3           | 
|79906|6         |6           | 
|79907|96        |70          | 
|79908|41        |21          | 
|79910|23        |23          | 
|79911|9         |9           | 
|79912|6         |6           | 
|79913|10        |10          | 
|79914|5         |5           | 
|79915|17        |17          | 
|79917|1         |1           | 
|79919|1         |1           | 
|79927|5         |5           | 
|79928|5         |5           | 
|79929|1         |1           | 
|79932|2         |1           | 
|79944|1         |1           | 
|79999|4         |4           | 
|80100|12        |12          | 
|80101|50        |50          | 
|80102|21        |21          | 
|80105|13        |13          | 
|80221|39        |35          | 
|80222|25        |18          | 
|80223|4         |4           | 
|80231|16        |16          | 
|80242|44        |44          | 
|80252|18        |18          | 
|80255|39        |39          | 
|80265|24        |24          | 
|80270|2         |2           | 
|80300|26        |26          | 
|80301|8         |8           | 
|80302|8         |8           | 
|80304|5         |5           | 
|80305|4         |4           | 
|80306|4         |4           | 
|80307|3         |3           | 
|80400|60        |60          | 
|80401|16        |16          | 
|80402|38        |38          | 
|80403|20        |20          | 
|80404|37        |37          | 
|80405|82        |69          | 
|80406|36        |36          | 
|80407|86        |86          | 
|80408|5         |5           | 
|80410|73        |71          | 
|80411|9         |9           | 
|80412|11        |11          | 
|80413|4         |4           | 
|80414|2         |2           | 
|80415|4         |4           | 
|80422|6         |6           | 
|80636|9         |9           | 
|89707|7         |7           | 
|89710|4         |4           | 
|89712|5         |5           | 
|89901|1         |1           | 
|89902|7         |7           | 
|89903|4         |4           | 
|89904|1         |1           | 
|89905|1         |1           | 
|89906|8         |8           | 
|89908|1         |1           | 
|89910|1         |1           | 
|89913|3         |3           | 
|89914|1         |1           | 
|89917|5         |5           | 
|89919|30        |30          | 
|89921|10        |10          | 
|89927|3         |3           | 
|90100|19        |19          | 
|90106|3         |3           | 
|90400|26        |26          | 
|90404|4         |4           | 
|90405|12        |12          | 
|90500|12        |12          | 
|90601|6         |6           | 
|90602|24        |24          | 
|90603|26        |26          | 
|90611|22        |22          | 
|90612|8         |8           | 
|90622|2         |2           | 
|90631|5         |5           | 
|90640|31        |27          | 
|90641|5         |5           | 
|90642|4         |4           | 
|90650|3         |3           | 
|90652|4         |4           | 
|90701|3         |3           | 
|92150|36        |36          | 
|92151|38        |38          | 
|92152|18        |18          | 
|92154|10        |10          | 
|92156|20        |20          | 
|92159|1         |1           | 
|92160|15        |15          | 
|94008|1         |1           | 
|94040|6         |6           | 
|94941|178       |178         | 
|94942|72        |72          | 
|94952|52        |52          | 
|94954|11        |11          | 
|94956|8         |8           | 
|94959|48        |48          | 
|94966|3         |3           | 
|94990|1         |1           | 
|94997|4         |4           | 
|95050|1         |1           | 
|95051|19        |19          | 
|99999|448       |448         | 
+-----+----------+------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare EMPID distribution between Prod and Fabric. 
# This checks that events are assigned to employees in similar proportions. 
 
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
 

 
 
```

```text
### EMPID DISTRIBUTION

+-----+----------+------------+ 
|EMPID|prod_count|fabric_count| 
+-----+----------+------------+ 
|SYSTM|2203      |2155        | 
|0    |2055      |1991        | 
|5881 |976       |976         | 
|99999|484       |482         | 
|5234 |413       |413         | 
|5973 |182       |172         | 
|14105|164       |158         | 
|5980 |139       |139         | 
|8198 |136       |136         | 
|5661 |127       |125         | 
|4502 |124       |118         | 
|5241 |116       |116         | 
|5736 |100       |100         | 
|5688 |99        |99          | 
|6128 |98        |98          | 
|6020 |95        |88          | 
|5746 |87        |87          | 
|5840 |91        |86          | 
|8038 |83        |83          | 
|5717 |77        |77          | 
|3879 |75        |75          | 
|14074|77        |73          | 
|5887 |70        |70          | 
|14030|72        |68          | 
|5823 |72        |68          | 
|14102|69        |67          | 
|5982 |65        |65          | 
|6134 |70        |63          | 
|6116 |71        |62          | 
|14073|65        |60          | 
|5683 |63        |59          | 
|14019|54        |52          | 
|14038|50        |50          | 
|5532 |49        |49          | 
|14018|47        |47          | 
|5880 |51        |45          | 
|5927 |44        |44          | 
|4403 |43        |43          | 
|14046|42        |42          | 
|5406 |42        |42          | 
|2895 |38        |38          | 
|8263 |37        |37          | 
|2854 |35        |35          | 
|4475 |33        |33          | 
|5889 |30        |30          | 
|5550 |29        |29          | 
|4594 |27        |27          | 
|5870 |22        |22          | 
|5338 |18        |18          | 
|5930 |15        |15          | 
|5972 |14        |14          | 
|4602 |13        |13          | 
|3918 |10        |10          | 
|5260 |10        |10          | 
|3440 |9         |9           | 
|3742 |9         |9           | 
|5742 |8         |8           | 
|6155 |7         |7           | 
|6066 |6         |6           | 
|6008 |4         |4           | 
|N2152|3         |3           | 
|4660 |2         |2           | 
|6175 |2         |2           | 
+-----+----------+------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare MACHINE distribution between Prod and Fabric. 
# This checks whether each tool/machine sees comparable volume in both datasets. 
 
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
 

 
 
```

```text
### MACHINE DISTRIBUTION

+--------------+----------+------------+ 
|MACHINE       |prod_count|fabric_count| 
+--------------+----------+------------+ 
|NULL          |2905      |0           | 
|NULL          |0         |2879        | 
|ALPHA307      |7         |7           | 
|ALPHA311      |21        |21          | 
|ALPHA312      |4         |4           | 
|ALPHA314      |14        |14          | 
|ALPHA804      |6         |6           | 
|ALPHA805      |6         |6           | 
|AME10         |13        |13          | 
|AME11         |10        |10          | 
|AME15         |42        |42          | 
|AME20         |15        |15          | 
|AME21         |9         |9           | 
|AME22         |11        |11          | 
|AME309        |6         |6           | 
|AME312        |15        |15          | 
|AME313        |30        |30          | 
|AME314        |15        |15          | 
|AME317        |10        |10          | 
|AME324        |11        |11          | 
|AME325        |26        |26          | 
|AME326        |19        |19          | 
|AME327        |7         |7           | 
|AME328        |24        |20          | 
|AME329        |9         |9           | 
|ASML100-301   |28        |28          | 
|ASML100-303   |4         |4           | 
|ASML100-304   |17        |17          | 
|ASML100-305   |21        |21          | 
|ASML100-308   |31        |27          | 
|ASML100-318   |15        |15          | 
|ASML100-319   |15        |15          | 
|ATLAS305      |12        |12          | 
|BAGSEALER302  |12        |12          | 
|CDSEM304      |2         |2           | 
|CMP2          |6         |6           | 
|CMP4          |19        |19          | 
|CMP5          |19        |19          | 
|ECLIPSE308    |8         |8           | 
|ELIPS10       |61        |61          | 
|ELIPS311      |149       |149         | 
|ENDURA211     |23        |23          | 
|ENDURA3       |13        |9           | 
|ENDURA306     |36        |36          | 
|ENDURA307     |4         |4           | 
|ENDURA308     |47        |47          | 
|ENDURA4       |2         |2           | 
|ENDURA5       |26        |19          | 
|EPI7          |4         |4           | 
|EPI8          |7         |7           | 
|FAAST301      |3         |3           | 
|FTIR1         |6         |6           | 
|FUSION302     |26        |26          | 
|FUSION303     |11        |11          | 
|FUSION304     |4         |4           | 
|FUSION305     |9         |9           | 
|FUSION306     |8         |8           | 
|FUSION307     |16        |16          | 
|FUSION310     |8         |5           | 
|FUSION311     |4         |4           | 
|GSD1          |9         |9           | 
|GSD2          |19        |19          | 
|GSD3          |30        |30          | 
|GSD306        |16        |16          | 
|GSD4          |27        |27          | 
|GSD7          |53        |43          | 
|HD302         |4         |4           | 
|HDP301        |21        |19          | 
|HE301         |50        |50          | 
|INSP14        |12        |12          | 
|INSP311       |129       |117         | 
|INSP312       |90        |90          | 
|INSP315       |116       |116         | 
|IVS303        |44        |42          | 
|IVS304        |65        |61          | 
|IVS305        |90        |73          | 
|KEITHLY1      |14        |14          | 
|KEITHLY4      |2         |2           | 
|KLA8930-301   |3         |3           | 
|KLA8935-302   |4         |4           | 
|KLARITY_DEFECT|43        |43          | 
|KLASEM301     |201       |183         | 
|KLASEM302     |184       |184         | 
|KLASP1-301    |55        |55          | 
|KLASP1-302    |89        |89          | 
|LINK-15       |109       |109         | 
|LINK-16       |88        |88          | 
|LINK-17       |93        |93          | 
|LINK-304      |129       |129         | 
|LINK-305      |107       |107         | 
|LINK-307      |39        |39          | 
|LINK-308      |58        |58          | 
|LINK-309      |1         |1           | 
|LINK-310      |21        |21          | 
|LINK-311      |34        |34          | 
|LINK-312      |41        |41          | 
|LINK-313      |18        |15          | 
|LINK-320      |57        |57          | 
|LINK-321      |54        |54          | 
|MERCURY1      |16        |16          | 
|MERCURY10     |87        |87          | 
|MERCURY3      |96        |96          | 
|MERCURY302    |112       |112         | 
|MERCURY306    |67        |64          | 
|MERCURY311    |52        |52          | 
|MERCURY312    |72        |70          | 
|MERCURY313    |16        |16          | 
|MERCURY314    |43        |43          | 
|MERCURY315    |68        |68          | 
|MERCURY316    |51        |51          | 
|MERCURY317    |110       |108         | 
|MERCURY318    |72        |59          | 
|MERCURY5      |54        |54          | 
|MERCURY8      |54        |54          | 
|MRL15.2       |6         |6           | 
|MRL15.3       |22        |22          | 
|MRL18.2       |4         |4           | 
|MRL22.3       |4         |4           | 
|MRL323.2      |1         |1           | 
|MRL324.2      |9         |9           | 
|MRL325.3      |71        |71          | 
|MRL326.4      |32        |32          | 
|MRL327.1      |153       |153         | 
|NK301         |27        |27          | 
|NOVEL22       |36        |36          | 
|NOVEL23       |73        |73          | 
|NOVEL24       |6         |6           | 
|NOVEL307      |30        |30          | 
|NOVEL309      |8         |8           | 
|NOVEL311      |20        |20          | 
|NOVEL312      |94        |94          | 
|NOVEL313      |27        |27          | 
|NOVEL314      |32        |19          | 
|NOVEL315      |42        |42          | 
|NOVEL316      |22        |22          | 
|NOVEL321      |28        |28          | 
|NOVEL325      |18        |18          | 
|NOVEL327      |24        |24          | 
|OMNIMAP       |12        |12          | 
|OVEN306       |14        |14          | 
|PEP1          |37        |37          | 
|PEP3          |11        |11          | 
|PEP302        |15        |15          | 
|PEP309        |42        |36          | 
|PEP310        |32        |32          | 
|PEP311        |44        |44          | 
|PEP312        |18        |18          | 
|PEP313        |29        |29          | 
|PEP315        |31        |31          | 
|PEP5          |44        |35          | 
|PEP6          |49        |49          | 
|PEP7          |21        |21          | 
|PEP8          |16        |16          | 
|RESMAP302     |13        |13          | 
|RTA301        |9         |9           | 
|RTA303        |33        |33          | 
|RTA305        |2         |2           | 
|RTA306        |12        |12          | 
|RTA309        |36        |29          | 
|RTA311        |11        |11          | 
|RTA4          |9         |9           | 
|SCRIBE303     |19        |19          | 
|SCRUB302      |62        |50          | 
|SCRUB303      |45        |45          | 
|SCRUB306      |68        |66          | 
|SWAP_BEOL     |7         |6           | 
|SWAP_FEOL     |7         |7           | 
|TEL303        |37        |37          | 
|TEL305        |20        |20          | 
|TEL307        |44        |44          | 
|TEL310        |35        |35          | 
|UV1280-101    |29        |29          | 
|UV1280-102    |4         |4           | 
|UV1280-103    |6         |6           | 
|VAF1          |9         |9           | 
|VDF2          |6         |6           | 
|VDF6          |6         |6           | 
|VIISTA301     |80        |80          | 
|VIISTA302     |87        |87          | 
|VTR1          |7         |7           | 
|VTR304        |4         |4           | 
|VTR306        |28        |28          | 
|WAFSORT3      |16        |16          | 
|WAFSORT302    |27        |27          | 
|WAFSORT304    |54        |54          | 
|WB102         |34        |34          | 
|WB301         |58        |58          | 
|WB303         |8         |8           | 
|WB304         |19        |19          | 
|WB305         |30        |19          | 
|WB389-L       |15        |15          | 
|WB81          |1         |1           | 
|WB84-Y        |13        |13          | 
|WB84-Z        |13        |13          | 
|WB85          |50        |50          | 
|WB91          |44        |44          | 
+--------------+----------+------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compute overall summary metrics for each dataset: 
#  - rows, distinct lots, distinct operations, users, machines, and empids. 
 
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
 

 
 
```

```text
### SUMMARY METRICS

Prod Summary 
+----+----+-----+-----+--------+------+ 
|rows|lots|opers|users|machines|empids| 
+----+----+-----+-----+--------+------+ 
|9421|1126|465  |248  |194     |63    | 
+----+----+-----+-----+--------+------+ 
 
Fabric Summary 
+----+----+-----+-----+--------+------+ 
|rows|lots|opers|users|machines|empids| 
+----+----+-----+-----+--------+------+ 
|9224|1104|465  |248  |194     |63    | 
+----+----+-----+-----+--------+------+ 
 

 
 
 
```

```python
# LOGIC: 
# On the set of rows matched by key (LOT, DATE_TIME, HISTORDER), 
# count how many rows have mismatched values in selected attributes: 
#  OPER, descriptions, TRANS, HIST_REC, MACHINE, USERNAME, COMMAND, EMPID. 
 
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
 

 
 
```

```text
### ATTRIBUTE MISMATCH ANALYSIS

+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|OPER_MISMATCH|OPERDESC_MISMATCH|OPERLONGDESC_MISMATCH|TRANS_MISMATCH|HIST_REC_MISMATCH|MACHINE_MISMATCH|USERNAME_MISMATCH|COMMAND_MISMATCH|EMPID_MISMATCH| 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|0            |0                |0                    |1746          |2500             |26              |54               |322             |0             | 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where TRANS differs between Prod and Fabric, 
# along with the key fields and both TRANS values. 
 
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
 

 
 
```

```text
### TRANS MISMATCH SAMPLE

+----------+---------+---------+----------+------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_TRANS|FABRIC_TRANS| 
+----------+---------+---------+----------+------------+ 
|5245A001  |00:00.2  |1        |WAIT MVOU |COMMENT     | 
|5245A001  |00:00.2  |1        |COMMENT   |WAIT MVOU   | 
|6352ACB4B |00:00.4  |1        |WAIT MVOU |COMMENT     | 
|6352ACB4B |00:00.4  |1        |COMMENT   |WAIT MVOU   | 
|6352ACA3  |00:00.6  |1        |WAIT MVOU |COMMENT     | 
|6352ACA3  |00:00.6  |1        |COMMENT   |WAIT MVOU   | 
|7745A1F2  |00:00.8  |1        |WAIT MVOU |COMMENT     | 
|7745A1F2  |00:00.8  |1        |COMMENT   |WAIT MVOU   | 
|5773A1C6  |00:00.9  |1        |DISPATCH  |COMMENT     | 
|5773A1C6  |00:00.9  |1        |COMMENT   |DISPATCH    | 
|7862A072  |00:03.2  |1        |WAIT MVOU |COMMENT     | 
|7862A072  |00:03.2  |1        |COMMENT   |WAIT MVOU   | 
|5499A028  |00:03.6  |1        |WAIT MVOU |COMMENT     | 
|5499A028  |00:03.6  |1        |COMMENT   |WAIT MVOU   | 
|6352AC98  |00:04.0  |1        |WAIT MVOU |COMMENT     | 
|6352AC98  |00:04.0  |1        |COMMENT   |WAIT MVOU   | 
|6352AC71  |00:04.4  |1        |WAIT MVOU |COMMENT     | 
|6352AC71  |00:04.4  |1        |COMMENT   |WAIT MVOU   | 
|7579A1D2  |00:05.8  |1        |DISPATCH  |COMMENT     | 
|7579A1D2  |00:05.8  |1        |COMMENT   |DISPATCH    | 
|6541T14146|00:32.3  |1        |TRANSITION|COMMENT     | 
|6541T14146|00:32.3  |1        |COMMENT   |TRANSITION  | 
|6541T14146|00:34.1  |1        |MOVE IN   |COMMENT     | 
|6541T14146|00:34.1  |1        |COMMENT   |MOVE IN     | 
|7677A036  |00:37.8  |1        |TRANSITION|COMMENT     | 
|7677A036  |00:37.8  |1        |COMMENT   |TRANSITION  | 
|7677A036  |00:39.0  |1        |MOVE IN   |COMMENT     | 
|7677A036  |00:39.0  |1        |COMMENT   |MOVE IN     | 
|6814A040  |00:52.4  |1        |SUGG RECIP|MOVE IN     | 
|6814A040  |00:52.4  |1        |MOVE IN   |SUGG RECIP  | 
|6814A042  |00:52.6  |1        |SUGG RECIP|MOVE IN     | 
|6814A042  |00:52.6  |1        |MOVE IN   |SUGG RECIP  | 
|7738A016  |00:52.8  |1        |HOLD CODE |REPOSITION  | 
|7738A016  |00:52.8  |1        |HOLD CODE |COMMENT     | 
|5941A0Y6  |00:52.8  |1        |SUGG RECIP|MOVE IN     | 
|5941A0Y6  |00:52.8  |1        |MOVE IN   |SUGG RECIP  | 
|7738A016  |00:52.8  |1        |REPOSITION|HOLD CODE   | 
|7738A016  |00:52.8  |1        |REPOSITION|COMMENT     | 
|7738A016  |00:52.8  |1        |COMMENT   |HOLD CODE   | 
|7738A016  |00:52.8  |1        |COMMENT   |REPOSITION  | 
|7423A021  |00:56.2  |1        |WAIT MVOU |COMMENT     | 
|7423A021  |00:56.2  |1        |COMMENT   |WAIT MVOU   | 
|5703A044  |01:26.6  |1        |TRANSITION|COMMENT     | 
|5703A044  |01:26.6  |1        |COMMENT   |TRANSITION  | 
|5703A044  |01:40.5  |1        |UNTRANSITN|COMMENT     | 
|5703A044  |01:40.5  |1        |COMMENT   |UNTRANSITN  | 
|6541T14146|01:46.1  |1        |TRANSITION|COMMENT     | 
|6541T14146|01:46.1  |1        |COMMENT   |TRANSITION  | 
|6541T14146|01:48.0  |1        |SUGG RECIP|MOVE IN     | 
|6541T14146|01:48.0  |1        |SUGG RECIP|COMMENT     | 
|6541T14146|01:48.0  |1        |MOVE IN   |COMMENT     | 
|6541T14146|01:48.0  |1        |MOVE IN   |SUGG RECIP  | 
|6541T14146|01:48.0  |1        |COMMENT   |MOVE IN     | 
|6541T14146|01:48.0  |1        |COMMENT   |SUGG RECIP  | 
|6515A0S6  |02:08.2  |1        |DISPATCH  |COMMENT     | 
|6515A0S6  |02:08.2  |1        |COMMENT   |DISPATCH    | 
|5941A113  |02:08.5  |1        |DISPATCH  |COMMENT     | 
|5941A113  |02:08.5  |1        |COMMENT   |DISPATCH    | 
|7421A146  |02:08.8  |1        |DISPATCH  |COMMENT     | 
|7421A146  |02:08.8  |1        |COMMENT   |DISPATCH    | 
|5941A0W5  |02:09.0  |1        |DISPATCH  |COMMENT     | 
|5941A0W5  |02:09.0  |1        |COMMENT   |DISPATCH    | 
|6339A2L4  |02:10.3  |1        |DISPATCH  |COMMENT     | 
|6339A2L4  |02:10.3  |1        |COMMENT   |DISPATCH    | 
|7499A036  |02:12.4  |1        |DISPATCH  |COMMENT     | 
|7499A036  |02:12.4  |1        |COMMENT   |DISPATCH    | 
|7862A063  |02:18.3  |1        |DISPATCH  |COMMENT     | 
|7862A063  |02:18.3  |1        |COMMENT   |DISPATCH    | 
|7060T4L0  |02:24.1  |1        |DISPATCH  |COMMENT     | 
|7060T4L0  |02:24.1  |1        |COMMENT   |DISPATCH    | 
|5941A0W8  |02:33.3  |1        |DISPATCH  |COMMENT     | 
|5941A0W8  |02:33.3  |1        |COMMENT   |DISPATCH    | 
|5703A044  |02:35.4  |1        |TRANSITION|COMMENT     | 
|5703A044  |02:35.4  |1        |COMMENT   |TRANSITION  | 
|7584A018  |02:39.6  |1        |WAIT MVOU |COMMENT     | 
|7584A018  |02:39.6  |1        |COMMENT   |WAIT MVOU   | 
|6352ACB3  |02:39.8  |1        |WAIT MVOU |COMMENT     | 
|6352ACB3  |02:39.8  |1        |COMMENT   |WAIT MVOU   | 
|6352AC87  |02:40.0  |1        |WAIT MVOU |COMMENT     | 
|6352AC87  |02:40.0  |1        |COMMENT   |WAIT MVOU   | 
|7412A0F6  |02:40.2  |1        |WAIT MVOU |COMMENT     | 
|7412A0F6  |02:40.2  |1        |COMMENT   |WAIT MVOU   | 
|6786A180  |02:40.9  |1        |DISPATCH  |COMMENT     | 
|6786A180  |02:40.9  |1        |COMMENT   |DISPATCH    | 
|6814A035  |02:42.3  |1        |DISPATCH  |COMMENT     | 
|6814A035  |02:42.3  |1        |COMMENT   |DISPATCH    | 
|5849A026  |02:46.2  |1        |TRANSITION|COMMENT     | 
|5849A026  |02:46.2  |1        |COMMENT   |TRANSITION  | 
|5849A026  |02:47.2  |1        |MOVE IN   |COMMENT     | 
|5849A026  |02:47.2  |1        |COMMENT   |MOVE IN     | 
|5200A3A8  |02:54.8  |1        |WAIT MVOU |COMMENT     | 
|5200A3A8  |02:54.8  |1        |COMMENT   |WAIT MVOU   | 
|6833A049  |02:59.8  |1        |TRANSITION|COMMENT     | 
|6833A049  |02:59.8  |1        |COMMENT   |TRANSITION  | 
|5814A003  |03:13.2  |1        |DISPATCH  |COMMENT     | 
|5814A003  |03:13.2  |1        |COMMENT   |DISPATCH    | 
|4562AZB0  |03:15.4  |1        |DISPATCH  |COMMENT     | 
|4562AZB0  |03:15.4  |1        |COMMENT   |DISPATCH    | 
|5703A046  |03:23.2  |1        |TRANSITION|COMMENT     | 
|5703A046  |03:23.2  |1        |COMMENT   |TRANSITION  | 
|6352AC69  |03:26.9  |1        |WAIT MVOU |COMMENT     | 
|6352AC69  |03:26.9  |1        |COMMENT   |WAIT MVOU   | 
|6641AW55  |03:36.7  |1        |WAIT MVOU |COMMENT     | 
|6641AW55  |03:36.7  |1        |COMMENT   |WAIT MVOU   | 
|7610A2P4  |03:47.0  |1        |TRANSITION|COMMENT     | 
|7610A2P4  |03:47.0  |1        |COMMENT   |TRANSITION  | 
|6802A167  |03:48.4  |1        |DISPATCH  |COMMENT     | 
|6802A167  |03:48.4  |1        |COMMENT   |DISPATCH    | 
|5620AG08  |03:52.4  |1        |TRANSITION|COMMENT     | 
|5620AG08  |03:52.4  |1        |COMMENT   |TRANSITION  | 
|5620AG08  |03:55.0  |1        |SUGG RECIP|COMMENT     | 
|5620AG08  |03:55.0  |1        |SUGG RECIP|MOVE IN     | 
|5620AG08  |03:55.0  |1        |MOVE IN   |COMMENT     | 
|5620AG08  |03:55.0  |1        |MOVE IN   |SUGG RECIP  | 
|5620AG08  |03:55.0  |1        |COMMENT   |SUGG RECIP  | 
|5620AG08  |03:55.0  |1        |COMMENT   |MOVE IN     | 
... 
only showing top 200 rows 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where COMMAND differs between Prod and Fabric, 
# along with the key fields and both COMMAND values. 
 
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
 

 
 
```

```text
### COMMAND MISMATCH SAMPLE

+----------+---------+---------+------------+--------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_COMMAND|FABRIC_COMMAND| 
+----------+---------+---------+------------+--------------+ 
|6814A040  |00:52.4  |1        |LMVR        |MVIN          | 
|6814A040  |00:52.4  |1        |MVIN        |LMVR          | 
|6814A042  |00:52.6  |1        |LMVR        |MVIN          | 
|6814A042  |00:52.6  |1        |MVIN        |LMVR          | 
|7738A016  |00:52.8  |1        |HOLD        |RPOS          | 
|7738A016  |00:52.8  |1        |HOLD        |RPOS          | 
|5941A0Y6  |00:52.8  |1        |LMVR        |MVIN          | 
|5941A0Y6  |00:52.8  |1        |MVIN        |LMVR          | 
|7738A016  |00:52.8  |1        |RPOS        |HOLD          | 
|7738A016  |00:52.8  |1        |RPOS        |HOLD          | 
|6541T14146|01:48.0  |1        |LMVR        |MVIN          | 
|6541T14146|01:48.0  |1        |LMVR        |MVIN          | 
|6541T14146|01:48.0  |1        |MVIN        |LMVR          | 
|6541T14146|01:48.0  |1        |MVIN        |LMVR          | 
|5620AG08  |03:55.0  |1        |LMVR        |MVIN          | 
|5620AG08  |03:55.0  |1        |LMVR        |MVIN          | 
|5620AG08  |03:55.0  |1        |MVIN        |LMVR          | 
|5620AG08  |03:55.0  |1        |MVIN        |LMVR          | 
|6833A049  |04:50.4  |1        |LMVR        |MVIN          | 
|6833A049  |04:50.4  |1        |MVIN        |LMVR          | 
|5703A044  |04:50.8  |1        |LMVR        |MVIN          | 
|5703A044  |04:50.8  |1        |MVIN        |LMVR          | 
|7192A1F6  |06:47.9  |1        |LMVR        |MVIN          | 
|7192A1F6  |06:47.9  |1        |LMVR        |MVIN          | 
|7192A1F6  |06:47.9  |1        |MVIN        |LMVR          | 
|7192A1F6  |06:47.9  |1        |MVIN        |LMVR          | 
|6514A4D2  |07:16.3  |1        |RPOS        |CMNT          | 
|6514A4D2  |07:16.3  |1        |CMNT        |RPOS          | 
|6541T14142|07:28.2  |1        |LMVR        |MVIN          | 
|6541T14142|07:28.2  |1        |LMVR        |MVIN          | 
|6541T14142|07:28.2  |1        |MVIN        |LMVR          | 
|6541T14142|07:28.2  |1        |MVIN        |LMVR          | 
|5582T8430 |08:32.4  |1        |LMVR        |MVIN          | 
|5582T8430 |08:32.4  |1        |LMVR        |MVIN          | 
|5582T8430 |08:32.4  |1        |MVIN        |LMVR          | 
|5582T8430 |08:32.4  |1        |MVIN        |LMVR          | 
... 
only showing top 200 rows 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where HIST_REC differs between Prod and Fabric, 
# capturing how the history text is split or ordered differently. 
 
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
 

 
 
```

```text
### HIST_REC MISMATCH SAMPLE

+----------+---------+---------+--------------------------------------+--------------------------------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_HIST_REC                         |FABRIC_HIST_REC                       | 
+----------+---------+---------+--------------------------------------+--------------------------------------+ 
|5245A001  |00:00.2  |1        |QTY:  6                               |Automated Move                        | 
|5245A001  |00:00.2  |1        |Automated Move                        |QTY:  6                               | 
|6352ACB4B |00:00.4  |1        |QTY: 25                               |Automated Move                        | 
|6352ACB4B |00:00.4  |1        |Automated Move                        |QTY: 25                               | 
|6352ACA3  |00:00.6  |1        |QTY: 25                               |Automated Move                        | 
|6352ACA3  |00:00.6  |1        |Automated Move                        |QTY: 25                               | 
|7745A1F2  |00:00.8  |1        |QTY:  6                               |Automated Move                        | 
|7745A1F2  |00:00.8  |1        |Automated Move                        |QTY:  6                               | 
|5773A1C6  |00:00.9  |1        |QTY: 25                               |Dispatched by Local Rules.            | 
|5773A1C6  |00:00.9  |1        |Dispatched by Local Rules.            |QTY: 25                               | 
|7862A072  |00:03.2  |1        |QTY: 25                               |Automated Move                        | 
|7862A072  |00:03.2  |1        |Automated Move                        |QTY: 25                               | 
|5499A028  |00:03.6  |1        |QTY: 25                               |Automated Move                        | 
|5499A028  |00:03.6  |1        |Automated Move                        |QTY: 25                               | 
|6352AC98  |00:04.0  |1        |QTY: 25                               |Automated Move                        | 
|6352AC98  |00:04.0  |1        |Automated Move                        |QTY: 25                               | 
|6352AC71  |00:04.4  |1        |QTY: 25                               |Automated Move                        | 
|6352AC71  |00:04.4  |1        |Automated Move                        |QTY: 25                               | 
|7579A1D2  |00:05.8  |1        |QTY: 25                               |Dispatched by Local Rules.            | 
|7579A1D2  |00:05.8  |1        |Dispatched by Local Rules.            |QTY: 25                               | 
|6541T14146|00:32.3  |1        |QTY:  4 TD                            |LotMoveIn Part 1                      | 
|6541T14146|00:32.3  |1        |LotMoveIn Part 1                      |QTY:  4 TD                            | 
... 
only showing top 200 rows 


 

 

 

 

 

 

 

 

 

 

 

 
```


---

## 📋 Comprehensive Observations & Data Validation Analysis

Based on the verification results extracted above, we present the detailed data analysis and root cause findings:

### 1. Data Volume & Exact Match Discrepancy
* **Total Rows in Production (SQL View):** 9,421
* **Total Rows in Fabric (Warehouse View):** 9,224
* **Common Rows (Exact Match on all columns):** 9,084
* **Discrepancy (Production Excess):** 197 rows (representing 2.09% of Production data)
* **Reconciliation Rate:** **96.42%** exact row match.

> [!CAUTION]
> **Validation Failure:** 197 rows are present in Production but missing completely from Fabric. Fabric contains no extra records (0 rows unique to Fabric).

---

### 2. Missing Lots and Suffix/Padding Issue
* **Distinct LOT count in Prod:** 1,126
* **Distinct LOT count in Fabric:** 1,104
* **Delta:** 22 LOTs are completely missing in Fabric.
* **The 22 Missing LOTs:**
  `5245A001A`, `5630A001A`, `5630A003A`, `5773A1B1A`, `5801A009A`, `6098A0E1A`, `6121A001A`, `6226A081A`, `6352AC23D`, `6352AC31A`, `6554A0A0 `, `6554A0A1 `, `6786A191A`, `7417A001A`, `7417A002A`, `7423A021A`, `7499A057 `, `7499A058 `, `7582A001A`, `7610A2J5A`, `7851AD83A`, `7851AD84A`

#### 🔍 Analysis:
* **Trailing Spaces:** Several of these missing lots (e.g., `6554A0A0 `, `7499A057 `) end with a trailing space. In SQL Server (Production), string comparisons automatically ignore trailing spaces or treat them leniently. In Spark/Fabric, comparisons are binary-exact. If the replication or the Fabric view uses string trimming or strict matching, these records fail to join and are dropped.
* **Suffixes:** Suffixes like `A` or `D` at the end of lot IDs might be filtered or trimmed during replication if there is a data cleansing step or database type mismatch (e.g., varchar length constraints or casting).

---

### 3. Duplication Source Check
* **Duplicates in Prod:** 140 rows
* **Duplicates in Fabric:** 140 rows
* **Analysis:** Both datasets contain exactly 140 duplicate rows (where the entire row is identical). This confirms that duplication is not introduced by the replication pipeline, but is inherited from the source database view.

---

### 4. Shuffled Attributes (Non-Deterministic Joins)
When performing a key-based join using `(LOT, DATE_TIME, HISTORDER)`, we observe massive mismatch rates on the row attributes:
* **HIST_REC Mismatches:** 2,500 rows
* **TRANS Mismatches:** 1,746 rows
* **COMMAND Mismatches:** 322 rows
* **USERNAME Mismatches:** 54 rows
* **MACHINE Mismatches:** 26 rows

#### 🔍 Why this happens (Join Key Instability):
* The combination of `(LOT, DATE_TIME, HISTORDER)` is **not unique** (as demonstrated by the *DUPLICATE KEYS IN PROD* section, where some composite keys return a count of 5 for different events).
* Because multiple transaction logs share the exact same timestamp and history order, there is no secondary ordering identifier (like a surrogate ID or physical sequence key) in the Fabric view joins.
* Consequently, when Spark compiles the Fabric view, it pairs tables non-deterministically. 
* *Example:* For `LOT = 5245A001`, `DATE_TIME = 00:00.2`, `HISTORDER = 1`, Production pairs `TRANS = WAIT MVOU` with `HIST_REC = QTY:  6` and `TRANS = COMMENT` with `HIST_REC = Automated Move`. Fabric mixes them up and pairs `TRANS = WAIT MVOU` with `HIST_REC = Automated Move` and `TRANS = COMMENT` with `HIST_REC = QTY:  6`. This causes values to transpose/shuffle across rows.

---

### 5. Machine NULL Discrepancy
* **Prod Machine Nulls:** 2,905 NULLs
* **Fabric Machine Nulls:** 2,879 NULLs
* **Delta:** 26 rows
* **Analysis:** This exactly aligns with the **26 MACHINE mismatches** from the attribute analysis. The non-deterministic join key alignment randomly paired rows with a NULL machine in Prod to rows with a populated machine in Fabric, leading to a minor count shift.

---

### 🛠️ Strategic Recommendations
1. **Enhance Join Conditions in Fabric View:** Update the view definitions in Fabric to join on a unique Transaction ID or sequence identifier (`TXN_ID` or a combination of `HISTORDER` and a sequence column) to ensure stable, deterministic attribute mapping.
2. **Standardize String Treatment (Trimming):** Implement consistent trimming logic (`TRIM(LOT)`) across all views and replication stages in Fabric to prevent trailing spaces from breaking joins and dropping lots.
3. **Verify Replication Filter Rules:** Confirm that the migration replication logic is not intentionally or accidentally excluding Lot IDs ending with suffix characters (like `A` or `D`).
