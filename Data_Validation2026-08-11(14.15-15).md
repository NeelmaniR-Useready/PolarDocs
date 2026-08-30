 
Production DB view –[TrainingVision].[dbo.LOTHISTV] 



Fabric view –[Polar_Warehouse].[TrainingVision].[LotHistV] 



 


Time stamp: DATETIME >= ‘2026-08-11 14:15:00’ AND  
	DATE_TIME < ‘2026-08-11 15:00:00’   

Below are the queries (code) from the second cell along with their results, in the same format as before: 

Each code block starts with a # LOGIC comment explaining what it does and what data it represents 

Each code block is followed by a highlighted result block (```text) 

 

```python
# LOGIC: 
# Load two CSV extracts (Prod and Fabric) from the Lakehouse for the specified window. 
# Then compute total row counts for each dataset to validate basic volume alignment. 
 
from pyspark.sql import functions as F 
 
df_Prod = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Prod-08-11(14.15-15).csv", 
    header=True, 
    inferSchema=True 
) 
 
df_Fabric = spark.read.csv( 
    "abfss://0af8298a-814c-4982-95a7-8f9cd2c8796c@onelake.dfs.fabric.microsoft.com/6c575156-1f48-4868-9e78-593559010765/Files/Fabric-08-11(14.15-15).csv", 
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

Prod Rows   : 9889 
Fabric Rows : 9706 
 

 
 
 
```

```python
# LOGIC: 
# For each column in Prod and Fabric, compute the number of distinct values. 
# This profiles column cardinality and helps detect missing/extra variety per system. 
 
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
 LOT          | 1257  
 DATE_TIME    | 2757  
 HISTORDER    | 231   
 TRANS        | 50    
 OPER         | 460   
 MASK_LVL     | 64    
 OPERDESC     | 155   
 OPERLONGDESC | 543   
 MACHINE      | 190   
 USERNAME     | 235   
 HIST_REC     | 1817  
 HISTCODE     | 50    
 COMMAND      | 20    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 76    
 
### FABRIC DISTINCT COUNTS

-RECORD 0------------ 
 LOT          | 1230  
 DATE_TIME    | 2757  
 HISTORDER    | 231   
 TRANS        | 50    
 OPER         | 460   
 MASK_LVL     | 64    
 OPERDESC     | 155   
 OPERLONGDESC | 543   
 MACHINE      | 190   
 USERNAME     | 235   
 HIST_REC     | 1817  
 HISTCODE     | 50    
 COMMAND      | 20    
 SHORTREPORT  | 2     
 VIEWFLAG     | 2     
 Is_Person    | 2     
 IS_DUPLICATE | 2     
 EMPID        | 76    
 

 
 
 
```

```python
# LOGIC: 
# Full row-level comparison: 
#  - common_rows  : rows identical in Prod and Fabric 
#  - only_prod    : rows only in Prod 
#  - only_fabric  : rows only in Fabric 
# Then compute match percentage vs Prod, and count duplicate rows in each dataset. 
 
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

Common rows         : 9614 
Only in Prod        : 183 
Only in Fabric      : 0 
 
### MATCH PERCENTAGE

Prod Match % : 97.22 
 
### DUPLICATE ROW ANALYSIS

Prod duplicates   : 95 
Fabric duplicates : 92 
 

 
 
 
```

```python
# LOGIC: 
# Find keys (LOT, DATE_TIME, HISTORDER) that appear more than once in Prod. 
# These identify duplicated key-level events in the Prod dataset. 
 
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

+----------+---------+---------+-----+ 
|LOT       |DATE_TIME|HISTORDER|count| 
+----------+---------+---------+-----+ 
|6794A015  |31:02.8  |15       |10   | 
|6794A014  |19:23.7  |15       |10   | 
|7412A0D9  |53:03.0  |1        |6    | 
|7412A0D9  |53:03.0  |2        |6    | 
|7412A0D9  |53:02.9  |1        |5    | 
|5139A003  |36:53.1  |1        |5    | 
|5139A003  |36:53.1  |2        |5    | 
|7412A0D9  |53:03.1  |1        |5    | 
|5398A1B9  |59:58.9  |1        |5    | 
|5139A003  |36:52.8  |2        |5    | 
|5139A003  |36:52.8  |1        |5    | 
|7738A015  |25:43.8  |1        |5    | 
|5139A003  |36:52.9  |2        |5    | 
|5398A1B9  |59:58.9  |2        |5    | 
|7412A0D9  |53:03.1  |2        |5    | 
|7412A0D9  |53:02.9  |2        |5    | 
|5139A003  |36:52.9  |1        |5    | 
|7738A015  |25:43.8  |2        |5    | 
|5139A003  |36:53.0  |1        |4    | 
|5139A003  |36:52.7  |1        |4    | 
|5325A230  |21:57.9  |2        |4    | 
|5139A003  |36:52.7  |2        |4    | 
|5325A230  |21:57.9  |1        |4    | 
|5139A003  |36:53.0  |2        |4    | 
|7347A0D3  |15:51.3  |1        |3    | 
|6504A039  |48:03.0  |2        |3    | 
|7225ABT9  |37:07.5  |1        |3    | 
|7610A2N0  |35:23.2  |1        |3    | 
|7412A0D9  |53:02.8  |2        |3    | 
|6198A045  |44:06.8  |1        |3    | 
|6389A072  |52:38.9  |1        |3    | 
|6098A0H8  |47:41.0  |2        |3    | 
|5703A032  |26:46.2  |1        |3    | 
|5500A002  |16:49.1  |1        |3    | 
|7991A022  |52:03.3  |2        |3    | 
|5539A1K8  |26:44.5  |1        |3    | 
|7486A009  |48:03.1  |1        |3    | 
|5773A1C8  |48:02.8  |1        |3    | 
|5941A0R4  |42:23.3  |1        |3    | 
|7412A0D9  |53:02.8  |1        |3    | 
|5499A030  |41:05.0  |1        |3    | 
|6514A4J7  |38:43.3  |1        |3    | 
|5813A003  |25:34.8  |1        |3    | 
|6098A0B8  |18:14.6  |1        |3    | 
|7710A051  |22:25.0  |2        |3    | 
|7851ADA5  |50:02.9  |1        |3    | 
|6514A449  |47:22.4  |1        |3    | 
|5539A1L3  |47:40.9  |1        |3    | 
|5620AG34  |48:02.7  |1        |3    | 
|5544T35906|42:04.5  |1        |3    | 
+----------+---------+---------+-----+ 
only showing top 50 rows 
 

 
 
 
```

```python
# LOGIC: 
# Compute null counts for each column in Prod and Fabric. 
# This identifies completeness differences (how many nulls) per column per system. 
 
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
 MACHINE      | 4258  
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
 MACHINE      | 4217  
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
# Compare the DATE_TIME coverage in Prod vs Fabric by checking min and max values. 
# Ensures both systems span the same time window for this extract. 
 
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
|15:00.6      |59:59.0      | 
+-------------+-------------+ 
 
Fabric 
+-------------+-------------+ 
|MIN_DATE_TIME|MAX_DATE_TIME| 
+-------------+-------------+ 
|15:00.6      |59:59.0      | 
+-------------+-------------+ 
 

 
 
 
```

```python
# LOGIC: 
# List distinct LOT values that are present only in Prod and only in Fabric, 
# based on the full row comparison (only_prod, only_fabric). 
 
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
|5208A020A| 
|5245A001A| 
|5325A234A| 
|5441A001 | 
|5451A0A7A| 
|5539A1Q7A| 
|5630A001A| 
|5630A002A| 
|5819A026A| 
|5822A061A| 
|6121A001A| 
|6339A2H0A| 
|6339A2J3A| 
|6339A2K9A| 
|6554A0A0 | 
|6782A001A| 
|6794A001N| 
|6794A002K| 
|6794A007B| 
|7192A1E4A| 
|7417A002A| 
|7417A003A| 
|7499A058 | 
|7610A2J5A| 
|7693A006F| 
|7693A006G| 
|7851AD70A| 
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
#  - prod_missing  : keys in Prod but not in Fabric 
#  - fabric_missing: keys in Fabric but not in Prod 
# Then list distinct LOTs associated with missing keys on each side. 
 
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

Common Keys      : 11994 
Prod Missing     : 183 
Fabric Missing   : 0 
 
### DISTINCT LOTS MISSING IN FABRIC

+---------+ 
|LOT      | 
+---------+ 
|5208A020A| 
|5245A001A| 
|5325A234A| 
|5441A001 | 
|5451A0A7A| 
|5539A1Q7A| 
|5630A001A| 
|5630A002A| 
|5819A026A| 
|5822A061A| 
|6121A001A| 
|6339A2H0A| 
|6339A2J3A| 
|6339A2K9A| 
|6554A0A0 | 
|6782A001A| 
|6794A001N| 
|6794A002K| 
|6794A007B| 
|7192A1E4A| 
|7417A002A| 
|7417A003A| 
|7499A058 | 
|7610A2J5A| 
|7693A006F| 
|7693A006G| 
|7851AD70A| 
+---------+ 
 
### DISTINCT LOTS MISSING IN PROD

+---+ 
|LOT| 
+---+ 
+---+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare distribution of HISTCODE between Prod and Fabric. 
# For each HISTCODE, show counts in each system and the difference. 
 
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
|LL      |1137      |1089        |-48       | 
|CM      |3594      |3549        |-45       | 
|TX      |449       |436         |-13       | 
|DL      |391       |379         |-12       | 
|SR      |383       |372         |-11       | 
|OI      |408       |398         |-10       | 
|IN      |359       |350         |-9        | 
|CS      |293       |286         |-7        | 
|RS      |185       |180         |-5        | 
|AR      |218       |214         |-4        | 
|LA      |353       |349         |-4        | 
|MV      |352       |348         |-4        | 
|HC      |50        |47          |-3        | 
|WM      |274       |271         |-3        | 
|UD      |52        |50          |-2        | 
|RP      |28        |27          |-1        | 
|SE      |29        |28          |-1        | 
|SK      |29        |28          |-1        | 
|AA      |2         |2           |0         | 
|AS      |30        |30          |0         | 
|CR      |23        |23          |0         | 
|DD      |21        |21          |0         | 
|DK      |214       |214         |0         | 
|EM      |94        |94          |0         | 
|EN      |94        |94          |0         | 
|HT      |23        |23          |0         | 
|KL      |176       |176         |0         | 
|KW      |176       |176         |0         | 
|LB      |6         |6           |0         | 
|LC      |19        |19          |0         | 
|MR      |18        |18          |0         | 
|OC      |1         |1           |0         | 
|OP      |5         |5           |0         | 
|OW      |23        |23          |0         | 
|PS      |13        |13          |0         | 
|PT      |33        |33          |0         | 
|R1      |1         |1           |0         | 
|RA      |1         |1           |0         | 
|RB      |17        |17          |0         | 
|RE      |23        |23          |0         | 
|RH      |17        |17          |0         | 
|RM      |1         |1           |0         | 
|RT      |199       |199         |0         | 
|RW      |1         |1           |0         | 
|SO      |2         |2           |0         | 
|TN      |23        |23          |0         | 
|UX      |3         |3           |0         | 
|VA      |23        |23          |0         | 
|WF      |2         |2           |0         | 
|WR      |21        |21          |0         | 
+--------+----------+------------+----------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare distribution of OPER between Prod and Fabric. 
# Shows per-operation event counts in both systems to validate consistency. 
 
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
|-100 |1100      |1100        | 
|20002|5         |5           | 
|20099|52        |52          | 
|21069|9         |9           | 
|21070|19        |19          | 
|22188|6         |6           | 
|22741|95        |95          | 
|24320|4         |4           | 
|25130|11        |11          | 
|25144|7         |7           | 
|27544|10        |10          | 
|40200|31        |31          | 
|40203|2         |2           | 
|40204|8         |8           | 
|40205|4         |4           | 
|40206|6         |6           | 
|40207|19        |19          | 
|40209|19        |19          | 
|40210|4         |4           | 
|40211|20        |20          | 
|40213|25        |25          | 
|40214|12        |12          | 
|40216|6         |6           | 
|40218|8         |8           | 
|40220|10        |10          | 
|40221|1         |1           | 
|40224|15        |15          | 
|40300|1         |1           | 
|40302|10        |10          | 
|40303|11        |11          | 
|40304|12        |12          | 
|40305|6         |6           | 
|40306|13        |13          | 
|40307|14        |14          | 
|40309|3         |3           | 
|40310|5         |5           | 
|40311|8         |8           | 
|40312|12        |12          | 
|40313|15        |15          | 
|40316|16        |16          | 
|40320|12        |12          | 
|40321|3         |3           | 
|40323|13        |13          | 
|40328|3         |3           | 
|40329|8         |8           | 
|40349|10        |10          | 
|40351|38        |38          | 
|40352|68        |68          | 
|40353|14        |14          | 
|40354|62        |62          | 
|40355|22        |17          | 
|40356|72        |72          | 
|40357|7         |7           | 
|40358|9         |9           | 
|40360|52        |52          | 
|40363|38        |38          | 
|40366|17        |17          | 
|40368|8         |8           | 
|40404|23        |23          | 
|40405|3         |3           | 
|40406|2         |2           | 
|40407|19        |19          | 
|40410|3         |3           | 
|40413|5         |5           | 
|40414|9         |9           | 
|40416|5         |5           | 
|40421|28        |28          | 
|40422|4         |4           | 
|40502|12        |12          | 
|40503|21        |21          | 
|40504|26        |26          | 
|40508|4         |4           | 
|40510|10        |10          | 
|40600|7         |7           | 
|40601|3         |3           | 
|40606|3         |3           | 
|40610|5         |5           | 
|40636|4         |4           | 
|45005|5         |5           | 
|45011|4         |4           | 
|45013|5         |5           | 
|45017|6         |6           | 
|45018|13        |13          | 
|49749|12        |12          | 
|49802|4         |4           | 
|49803|1         |1           | 
|49804|23        |23          | 
|49805|11        |11          | 
|49806|3         |3           | 
|49807|18        |18          | 
|49808|5         |5           | 
|49809|43        |43          | 
|49810|15        |15          | 
|49813|6         |6           | 
|49814|14        |14          | 
|49815|7         |7           | 
|49817|9         |9           | 
|49821|8         |8           | 
|49822|3         |3           | 
|49823|2         |2           | 
|49826|4         |4           | 
|49828|4         |4           | 
|49900|23        |23          | 
|49902|11        |11          | 
|49903|16        |16          | 
|49904|10        |10          | 
|49906|2         |2           | 
|49907|28        |28          | 
|49909|1         |1           | 
|49910|20        |20          | 
|49912|1         |1           | 
|49915|1         |1           | 
|49917|2         |2           | 
|49918|11        |11          | 
|49920|2         |2           | 
|49921|7         |7           | 
|49932|10        |8           | 
|49942|18        |18          | 
|49951|12        |12          | 
|49960|25        |25          | 
|49961|1         |1           | 
|49963|12        |8           | 
|49964|1         |1           | 
|49976|3         |3           | 
|50100|22        |22          | 
|50102|18        |18          | 
|50103|21        |21          | 
|50104|10        |10          | 
|50105|4         |4           | 
|50106|57        |57          | 
|50107|29        |29          | 
|50108|18        |18          | 
|50109|8         |8           | 
|50110|18        |18          | 
|50111|4         |4           | 
|50112|26        |26          | 
|50113|14        |14          | 
|50117|4         |4           | 
|50118|10        |10          | 
|50119|4         |4           | 
|50120|29        |29          | 
|50121|4         |4           | 
|50122|41        |41          | 
|50133|6         |6           | 
|50141|26        |26          | 
|50151|21        |21          | 
|50152|20        |14          | 
|50153|6         |6           | 
... 
(remaining OPER values continue similarly) 
 

 
 
 
```

```python
# LOGIC: 
# Compare EMPID distribution: number of records per EMPID in each system. 
# Validates that events are attributed to employees in similar proportions. 
 
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
|5493 |1656      |1656        | 
|SYSTM|1701      |1634        | 
|0    |1638      |1610        | 
|3414 |1142      |1142        | 
|99999|367       |351         | 
|3761 |195       |186         | 
|5780 |164       |164         | 
|4176 |152       |147         | 
|5668 |147       |147         | 
|5147 |139       |139         | 
|8308 |134       |134         | 
|5666 |126       |126         | 
|3734 |106       |105         | 
|6105 |102       |102         | 
|14020|96        |96          | 
|5873 |95        |95          | 
|5636 |91        |91          | 
|8445 |89        |89          | 
|3806 |88        |88          | 
|6012 |93        |84          | 
|5534 |79        |79          | 
|5685 |76        |75          | 
|6003 |64        |64          | 
|5417 |62        |62          | 
|4518 |59        |59          | 
|8192 |59        |59          | 
|6119 |57        |57          | 
|5664 |55        |55          | 
|8081 |55        |55          | 
|6002 |57        |54          | 
|6137 |53        |53          | 
|14028|52        |52          | 
|3194 |52        |52          | 
|5793 |56        |50          | 
|4895 |54        |49          | 
|5540 |48        |48          | 
|14050|55        |43          | 
|6062 |48        |41          | 
|6021 |40        |40          | 
|6049 |40        |40          | 
|5703 |39        |39          | 
|5246 |36        |36          | 
|5451 |35        |30          | 
|5469 |28        |28          | 
|6076 |29        |27          | 
|5818 |23        |23          | 
|5860 |23        |23          | 
|4167 |21        |21          | 
|4117 |17        |17          | 
|4452 |19        |17          | 
|5910 |17        |17          | 
|8369 |15        |15          | 
|6019 |13        |13          | 
|4343 |12        |12          | 
|5151 |12        |12          | 
|4857 |11        |11          | 
|14023|10        |10          | 
|5847 |15        |10          | 
|6159 |8         |8           | 
|2359 |7         |7           | 
|6067 |7         |7           | 
|2831 |5         |5           | 
|3511 |5         |5           | 
|3692 |5         |5           | 
|3994 |5         |5           | 
|3398 |4         |4           | 
|4984 |4         |4           | 
|5500 |4         |4           | 
|6148 |4         |4           | 
|4488 |3         |3           | 
|5912 |3         |3           | 
|14071|2         |2           | 
|4398 |2         |2           | 
|5626 |2         |2           | 
|5986 |1         |1           | 
|N2152|1         |1           | 
+-----+----------+------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Compare MACHINE distribution between Prod and Fabric. 
# Shows how many records are associated with each machine/tool in each system. 
 
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

+-----------------+----------+------------+ 
|MACHINE          |prod_count|fabric_count| 
+-----------------+----------+------------+ 
|NULL             |4258      |0           | 
|NULL             |0         |4217        | 
|ALPHA307         |18        |18          | 
|ALPHA318         |12        |12          | 
|ALPHA806         |9         |9           | 
|AME10            |19        |19          | 
|AME15            |24        |24          | 
|AME19            |2         |2           | 
|AME20            |25        |25          | 
|AME21            |9         |9           | 
|AME22            |11        |11          | 
|AME23            |3         |3           | 
|AME312           |14        |14          | 
|AME314           |6         |6           | 
|AME317           |7         |7           | 
|AME324           |2         |2           | 
|AME325           |1         |1           | 
|AME326           |13        |13          | 
|AME327           |5         |5           | 
|AME328           |18        |18          | 
|ASML100-301      |25        |25          | 
|ASML100-304      |13        |13          | 
|ASML100-305      |2         |2           | 
|ASML100-308      |23        |23          | 
|ASML100-318      |28        |28          | 
|ASML100-319      |13        |13          | 
|ATLAS305         |267       |241         | 
|BLAP NOTCH FINDER|8         |8           | 
|CDSEM304         |2         |2           | 
|CMP2             |15        |15          | 
|CMP3             |15        |15          | 
|CMP4             |58        |58          | 
|CMP6             |17        |17          | 
|CURE302          |12        |12          | 
|DETAPE103        |7         |7           | 
|ECLIPSE106       |24        |24          | 
|ELIPS10          |114       |114         | 
|ELIPS311         |247       |236         | 
|ENDURA211        |7         |7           | 
|ENDURA306        |8         |8           | 
|ENDURA307        |3         |3           | 
|ENDURA309        |13        |13          | 
|ENDURA5          |6         |6           | 
|EPI12            |3         |3           | 
|EPI6             |1         |1           | 
|EPI8             |4         |4           | 
|FLEX1            |7         |7           | 
|FUSION302        |14        |14          | 
|FUSION304        |7         |7           | 
|FUSION305        |16        |16          | 
|FUSION306        |13        |13          | 
|FUSION307        |11        |11          | 
|FUSION310        |15        |15          | 
|GRIND104         |13        |13          | 
|GSD1             |4         |4           | 
|GSD2             |8         |8           | 
|GSD4             |20        |20          | 
|GSD5             |17        |11          | 
|GSD7             |5         |5           | 
|HD302            |15        |15          | 
|HDP301           |24        |24          | 
|INSP309          |115       |115         | 
|INSP311          |66        |65          | 
|INSP312          |13        |13          | 
|INSP315          |79        |79          | 
|IVS303           |26        |26          | 
|IVS304           |57        |55          | 
|IVS305           |4         |2           | 
|KEITHLY12        |7         |7           | 
|KEITHLY3         |10        |10          | 
|KEITHLY4         |13        |13          | 
|KEITHLY6         |3         |3           | 
|KLA8935-302      |57        |57          | 
|KLA8935-303      |3         |3           | 
|KLARITY_DEFECT   |89        |89          | 
|KLASEM301        |177       |177         | 
|KLASP1-301       |36        |36          | 
|KLASP1-302       |45        |45          | 
|LINK-15          |2         |2           | 
|LINK-16          |6         |6           | 
|LINK-303         |61        |61          | 
|LINK-304         |52        |52          | 
|LINK-306         |84        |84          | 
|LINK-307         |42        |42          | 
|LINK-308         |66        |66          | 
|LINK-309         |63        |63          | 
|LINK-310         |38        |38          | 
|LINK-311         |26        |26          | 
|LINK-312         |76        |76          | 
|LINK-313         |60        |60          | 
|LINK-320         |25        |25          | 
|LINK-321         |14        |14          | 
|MERCURY1         |29        |29          | 
|MERCURY10        |44        |44          | 
|MERCURY3         |28        |28          | 
|MERCURY302       |59        |51          | 
|MERCURY306       |26        |24          | 
|MERCURY311       |62        |60          | 
|MERCURY312       |66        |62          | 
|MERCURY313       |19        |19          | 
|MERCURY314       |48        |48          | 
|MERCURY315       |23        |23          | 
|MERCURY316       |46        |46          | 
|MERCURY317       |16        |16          | 
|MERCURY318       |50        |50          | 
|MERCURY5         |50        |50          | 
|MERCURY8         |51        |51          | 
|METAL_TEST       |6         |6           | 
|MRL15.1          |91        |91          | 
|MRL15.2          |8         |8           | 
|MRL16.3          |40        |32          | 
|MRL17.2          |3         |3           | 
|MRL17.3          |5         |5           | 
|MRL22.2          |27        |27          | 
|MRL22.4          |149       |115         | 
... 
(remaining machines continue similarly) 
 

 
 
 
```

```python
# LOGIC: 
# Compute high-level summary metrics for each dataset: 
#  - total rows, distinct lots, operations, users, machines, and empids. 
 
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
|9889|1257|460  |235  |190     |76    | 
+----+----+-----+-----+--------+------+ 
 
Fabric Summary 
+----+----+-----+-----+--------+------+ 
|rows|lots|opers|users|machines|empids| 
+----+----+-----+-----+--------+------+ 
|9706|1230|460  |235  |190     |76    | 
+----+----+-----+-----+--------+------+ 
 

 
 
 
```

```python
# LOGIC: 
# On rows matched by key (LOT, DATE_TIME, HISTORDER), 
# count how many rows have mismatching values in key attributes: 
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
|6            |6                |6                    |1418          |1950             |22              |26               |260             |0             | 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where TRANS differs between Prod and Fabric, 
# along with key fields and both TRANS values, to inspect transformation differences. 
 
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
|7480A135  |15:38.5  |1        |WAIT MVOU |COMMENT     | 
|7480A135  |15:38.5  |1        |COMMENT   |WAIT MVOU   | 
|6814A052  |15:38.7  |1        |WAIT MVOU |COMMENT     | 
|6814A052  |15:38.7  |1        |COMMENT   |WAIT MVOU   | 
|6211A062  |15:38.9  |1        |WAIT MVOU |COMMENT     | 
|6211A062  |15:38.9  |1        |COMMENT   |WAIT MVOU   | 
|7225ABU5  |15:39.1  |1        |WAIT MVOU |COMMENT     | 
|7225ABU5  |15:39.1  |1        |COMMENT   |WAIT MVOU   | 
|7985A1N3  |15:47.1  |1        |DISPATCH  |COMMENT     | 
|7985A1N3  |15:47.1  |1        |COMMENT   |DISPATCH    | 
|7347A0D3  |15:51.3  |1        |HOLD CODE |REPOSITION  | 
|7347A0D3  |15:51.3  |1        |HOLD CODE |COMMENT     | 
|7347A0D3  |15:51.3  |1        |REPOSITION|HOLD CODE   | 
|7347A0D3  |15:51.3  |1        |REPOSITION|COMMENT     | 
|7347A0D3  |15:51.3  |1        |COMMENT   |HOLD CODE   | 
|7347A0D3  |15:51.3  |1        |COMMENT   |REPOSITION  | 
... 
only showing top 200 rows 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where COMMAND differs between Prod and Fabric, 
# including the key fields and both command values for investigation. 
 
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
|7347A0D3  |15:51.3  |1        |HOLD        |RPOS          | 
|7347A0D3  |15:51.3  |1        |HOLD        |RPOS          | 
|7347A0D3  |15:51.3  |1        |RPOS        |HOLD          | 
|7347A0D3  |15:51.3  |1        |RPOS        |HOLD          | 
|7412A0D7B |16:01.9  |1        |LMVR        |MVIN          | 
|7412A0D7B |16:01.9  |1        |LMVR        |MVIN          | 
|7412A0D7B |16:01.9  |1        |MVIN        |LMVR          | 
|7412A0D7B |16:01.9  |1        |MVIN        |LMVR          | 
|5500A002  |16:49.1  |1        |LMVR        |MVIN          | 
|5500A002  |16:49.1  |1        |LMVR        |MVIN          | 
|5500A002  |16:49.1  |1        |MVIN        |LMVR          | 
|5500A002  |16:49.1  |1        |MVIN        |LMVR          | 
|5941A105  |17:30.2  |1        |LMVR        |MVIN          | 
|5941A105  |17:30.2  |1        |LMVR        |MVIN          | 
|5941A105  |17:30.2  |1        |MVIN        |LMVR          | 
|5941A105  |17:30.2  |1        |MVIN        |LMVR          | 
... 
only showing top 200 rows 
 

 
 
 
```

```python
# LOGIC: 
# Show a detailed sample (up to 200 rows) where HIST_REC differs between Prod and Fabric, 
# to inspect differences in history text splitting, ordering, or content. 
 
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

+----------+---------+---------+----------------------------------------+----------------------------------------+ 
|LOT       |DATE_TIME|HISTORDER|PROD_HIST_REC                           |FABRIC_HIST_REC                         | 
+----------+---------+---------+----------------------------------------+----------------------------------------+ 
|7480A135  |15:38.5  |1        |QTY: 25                                 |Automated Move                          | 
|7480A135  |15:38.5  |1        |Automated Move                          |QTY: 25                                 | 
|6814A052  |15:38.7  |1        |QTY: 25                                 |Automated Move                          | 
|6814A052  |15:38.7  |1        |Automated Move                          |QTY: 25                                 | 
|6211A062  |15:38.9  |1        |QTY: 25                                 |Automated Move                          | 
|6211A062  |15:38.9  |1        |Automated Move                          |QTY: 25                                 | 
|7225ABU5  |15:39.1  |1        |QTY: 25                                 |Automated Move                          | 
|7225ABU5  |15:39.1  |1        |Automated Move                          |QTY: 25                                 | 
|7985A1N3  |15:47.1  |1        |QTY: 25                                 |Dispatched by Local Rules.              | 
|7985A1N3  |15:47.1  |1        |Dispatched by Local Rules.              |QTY: 25                                 | 
... 
only showing top 200 rows 
 
```


---

## 📋 Comprehensive Observations & Data Validation Analysis

Based on the verification results extracted above, we present the detailed data analysis and root cause findings:

### 1. Data Volume & Exact Match Discrepancy
* **Total Rows in Production (SQL View):** 9,889
* **Total Rows in Fabric (Warehouse View):** 9,706
* **Common Rows (Exact Match on all columns):** 9,614
* **Discrepancy (Production Excess):** 183 rows (representing 1.85% of Production data)
* **Reconciliation Rate:** **97.22%** exact row match.

> [!CAUTION]
> **Validation Failure:** 183 rows are present in Production but missing completely from Fabric. Fabric contains no extra records (0 rows unique to Fabric).

---

### 2. Missing Lots and Suffix/Padding Issue
* **Distinct LOT count in Prod:** 1,257
* **Distinct LOT count in Fabric:** 1,230
* **Delta:** 27 LOTs are completely missing in Fabric.
* **The 27 Missing LOTs:**
  `5208A020A`, `5245A001A`, `5325A234A`, `5441A001 `, `5451A0A7A`, `5539A1Q7A`, `5630A001A`, `5630A002A`, `5819A026A`, `5822A061A`, `6121A001A`, `6339A2H0A`, `6339A2J3A`, `6339A2K9A`, `6554A0A0 `, `6782A001A`, `6794A001N`, `6794A002K`, `6794A007B`, `7192A1E4A`, `7417A002A`, `7417A003A`, `7499A058 `, `7610A2J5A`, `7693A006F`, `7693A006G`, `7851AD70A`

#### 🔍 Analysis:
* **Trailing Spaces:** Trailing spaces in IDs (e.g. `5441A001 `, `6554A0A0 `, `7499A058 `) match leniently in SQL Server but fail to match strictly in Spark SQL.
* **Suffixes:** Suffixes like `A`, `K`, `N`, `B`, `F`, `G` (e.g., `5208A020A`, `6794A002K`, `7693A006F`, `7693A006G`) are missing. This indicates potential replication filters or data truncation constraints in the ETL sync script.

---

### 3. Duplication Source Check
* **Duplicates in Prod:** 95 rows
* **Duplicates in Fabric:** 92 rows
* **Analysis:** Unlike previous runs where the duplicates count matched perfectly, there is a minor delta of 3 duplicate rows. This is explained by the missing 27 lots; some dropped lots had duplicate records in Production, which subsequently reduces the duplicate count in Fabric. However, the rest of the 92 duplicates are inherited from the source.

---

### 4. Shuffled Attributes (Non-Deterministic Joins)
When performing a key-based join using `(LOT, DATE_TIME, HISTORDER)`, we observe massive mismatch rates on the row attributes:
* **HIST_REC Mismatches:** 1,950 rows
* **TRANS Mismatches:** 1,418 rows
* **COMMAND Mismatches:** 260 rows
* **USERNAME Mismatches:** 26 rows
* **MACHINE Mismatches:** 22 rows
* **OPER / OPERDESC / OPERLONGDESC Mismatches:** 6 rows

#### 🔍 Why this happens (Join Key Instability):
* The combination of `(LOT, DATE_TIME, HISTORDER)` is **not unique** (as demonstrated by the *DUPLICATE KEYS IN PROD* section, where some composite keys return a count of 10 for different events, e.g. `6794A015` at `31:02.8`).
* Because multiple transaction logs share the exact same timestamp and history order, there is no secondary ordering identifier (like a surrogate ID or physical sequence key) in the Fabric view joins.
* Consequently, when Spark compiles the Fabric view, it pairs tables non-deterministically. This causes attributes (e.g. pairing `TRANS` with `HIST_REC`, or even `OPER` in 6 cases) to shuffle between the duplicate records.

---

### 5. Machine NULL Discrepancy
* **Prod Machine Nulls:** 4,258 NULLs
* **Fabric Machine Nulls:** 4,217 NULLs
* **Delta:** 41 rows
* **Analysis:** This minor shift in NULL distribution is caused by the non-deterministic join key alignment described above, which mismatches NULL and populated machine values.

---

### 🛠️ Strategic Recommendations
1. **Enhance Join Conditions in Fabric View:** Update the view definitions in Fabric to join on a unique Transaction ID or sequence identifier (`TXN_ID` or a combination of `HISTORDER` and a sequence column) to ensure stable, deterministic attribute mapping.
2. **Standardize String Treatment (Trimming):** Implement consistent trimming logic (`TRIM(LOT)`) across all views and replication stages in Fabric to prevent trailing spaces from breaking joins and dropping lots.
3. **Verify Replication Filter Rules:** Confirm that the migration replication logic is not intentionally or accidentally excluding Lot IDs ending with suffix characters (like `A`, `K`, `N`, `B`, `F`, `G`).
