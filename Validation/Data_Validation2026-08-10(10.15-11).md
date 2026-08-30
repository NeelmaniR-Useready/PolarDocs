<h1 align="center" style="color: #1a365d;">📊 Production DB vs. Fabric Data Validation & Reconciliation Report</h1>

---

<h2 style="color: #2b6cb0;">📋 1. Scope & Execution Metadata</h2>

<div style="background-color: #f7fafc; padding: 15px; border-left: 5px solid #3182ce; border-radius: 4px; margin-bottom: 20px;">
  <p style="margin: 4px 0;"><b>🗄️ Production DB View:</b> <code style="color: #c53030; background-color: #fff5f5; padding: 2px 6px; border-radius: 3px;">[TrainingVision].[dbo.LOTHISTV]</code></p>
  <p style="margin: 4px 0;"><b>☁️ Fabric View:</b> <code style="color: #2b6cb0; background-color: #ebf8ff; padding: 2px 6px; border-radius: 3px;">[Polar_Warehouse].[TrainingVision].[LotHistV]</code></p>
  <p style="margin: 4px 0;"><b>⏱️ Filter Time Window:</b> <code style="color: #22543d; background-color: #f0fff4; padding: 2px 6px; border-radius: 3px;">DATETIME >= '2026-08-10 10:15:00' AND DATE_TIME < '2026-08-10 11:00:00'</code></p>
</div>

---

<h2 style="color: #2b6cb0;">📌 2. Detailed Observations, Root Cause Analysis & Recommendations</h2>

<h3 style="color: #2c5282;">A. Summary of Key Findings & Data Discrepancies</h3>
<ul>
  <li><span style="color: #e53e3e; font-weight: bold;">Volume Imbalance:</span> The Production view contains <b>10,023</b> records, while the Fabric warehouse copy contains <b>9,841</b> records (-182 net difference).</li>
  <li><span style="color: #38a169; font-weight: bold;">Common Match Rate:</span> Exact intersect row-level match is <b>9,673</b> rows, representing <b>96.51%</b> of Production records.</li>
  <li><span style="color: #dd6b20; font-weight: bold;">Orphaned / Extra Records:</span>
    <ul>
      <li><b>182 rows</b> exist <i>only in Production</i> and failed to land in Fabric.</li>
      <li><b>0 rows</b> exist <i>only in Fabric</i> and do not match the current Production view.</li>
    </ul>
  </li>
  <li><span style="color: #805ad5; font-weight: bold;">Higher Production Cardinality:</span> Production has higher distinct counts for Lots (<b>1,294</b> vs <b>1,265</b>) due to <b>29 missing LOTs</b> in Fabric.</li>
  <li><span style="color: #d69e2e; font-weight: bold;">Exact Duplicates:</span> Production contains <b>168 duplicate rows</b>; Fabric contains <b>168 duplicate rows</b>.</li>
</ul>

<h3 style="color: #2c5282;">B. Root Cause Analysis: Why Are There Duplicates and Missing/Extra Data?</h3>

<table style="width: 100%; border-collapse: collapse; margin: 15px 0;">
  <thead>
    <tr style="background-color: #2b6cb0; color: white; text-align: left;">
      <th style="padding: 10px; border: 1px solid #cbd5e0;">Category</th>
      <th style="padding: 10px; border: 1px solid #cbd5e0;">Observed Symptom</th>
      <th style="padding: 10px; border: 1px solid #cbd5e0;">Probable Technical Root Cause</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background-color: #f7fafc;">
      <td style="padding: 8px; border: 1px solid #cbd5e0; font-weight: bold; color: #c53030;">Duplicate Keys</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;">Composite key <code>[LOT, DATE_TIME, HISTORDER]</code> appears up to 5+ times for the same event in Prod & Fabric.</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;"><b>1. Sub-second Timestamp Truncation:</b> <code>DATE_TIME</code> is stored without full microsecond precision, collapsing multiple atomic events into one second bucket.<br><b>2. View Fan-Out:</b> The SQL view joins tables without proper distinct deduplication.</td>
    </tr>
    <tr>
      <td style="padding: 8px; border: 1px solid #cbd5e0; font-weight: bold; color: #dd6b20;">Data in Prod NOT in Fabric</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;"><b>182 rows</b> and <b>29 unique LOTs</b> are present in Prod but missing from Fabric.</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;"><b>1. Trailing Space Incompatibility:</b> Spark SQL matches LOT IDs strictly, while SQL Server ignores trailing spaces. LOT values like <code>6554A0A0 </code> fail to join and sync.<br><b>2. Character Suffix Filters:</b> Suffix characters (like `A`, `K`, `N`) may be truncated or filtered during migration.</td>
    </tr>
    <tr style="background-color: #f7fafc;">
      <td style="padding: 8px; border: 1px solid #cbd5e0; font-weight: bold; color: #805ad5;">Attribute Payload Mismatches</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;">Mismatches on keys: <b>2,204 HIST_REC</b>, <b>1,466 TRANS</b>, and <b>318 COMMAND</b>.</td>
      <td style="padding: 8px; border: 1px solid #cbd5e0;"><b>Non-Deterministic Joins:</b> Because composite key <code>(LOT, DATE_TIME, HISTORDER)</code> contains duplicate records, Spark joins tables non-deterministically, transposing/shuffling values across duplicate rows.</td>
    </tr>
  </tbody>
</table>

<h3 style="color: #2c5282;">C. Recommended Remediation Plan</h3>
<ol>
  <li><b>Standardize String Trimming:</b> Implement <code>TRIM(LOT)</code> across all view definitions and ingestion pipelines in Fabric to resolve trailing space discrepancies.</li>
  <li><b>Add True Surrogate Keys:</b> Re-engineer the views to include auto-incrementing transaction sequence IDs (e.g. <code>TXN_SEQUENCE</code>) to resolve non-deterministic joins.</li>
  <li><b>Verify Suffix Rules:</b> Inspect replication filter scripts to ensure LOT numbers ending with alphanumeric suffixes are not filtered out.</li>
</ol>

---

<h2 style="color: #2b6cb0;">💻 3. Notebook Cells: Code, Logic, Purpose & Results</h2>

### 🔹 Cell 1

#### 🎯 Logic & Purpose
Load two CSV extracts (Prod and Fabric) from the OneLake file system and count total row counts to validate basic volume alignment.
* **Data represented:** Total row counts of `df_Prod` and `df_Fabric` for the hourly window.

#### 💻 Code
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

#### 📊 Result
```text

Prod Rows   : 10023 
Fabric Rows : 9841 
 

 
 
 

2. Distinct count profile for each column
```

---

### 🔹 Cell 2

#### 🎯 Logic & Purpose
Profile cardinality by calculating distinct counts of unique values for each column in both datasets.
* **Data represented:** Cardinality profile across all schema columns in Production and Fabric.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 3

#### 🎯 Logic & Purpose
Perform full row-level set operations (`intersect`, `exceptAll`) to check row duplication and exact matches.
* **Data represented:** Exact matching rows, rows only in Production, rows only in Fabric, and duplicate record metrics.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 4

#### 🎯 Logic & Purpose
Identify logical event duplicate groups in Production by grouping on the composite natural keys `(LOT, DATE_TIME, HISTORDER)`.
* **Data represented:** Primary composite key combinations with occurrences greater than 1, sorted by frequency.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 5

#### 🎯 Logic & Purpose
Compute the count of null values per column in both Production and Fabric DataFrames to profile data completeness.
* **Data represented:** Null value count mapping across all schema columns.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 6

#### 🎯 Logic & Purpose
Validate timestamp range boundaries in both environments by checking the minimum and maximum `DATE_TIME` values.
* **Data represented:** Extremum timestamp boundaries in Production and Fabric.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 7

#### 🎯 Logic & Purpose
Isolate the distinct lot numbers associated with rows unique to Production or Fabric.
* **Data represented:** Unique LOTs present in Production but missing in Fabric, or vice versa.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 8

#### 🎯 Logic & Purpose
Compare composite keys `(LOT, DATE_TIME, HISTORDER)` between datasets to find missing keys and associated LOT values.
* **Data represented:** Common keys count, missing keys count, and distinct LOTs associated with missing keys.

#### 💻 Code
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

#### 📊 Result
```text

Common Keys      : 12691 
Prod Missing     : 182 
Fabric Missing   : 0 
 

 
 

Result (distinct lots missing in Fabric):
```

---

### 🔹 Cell 9

#### 🎯 Logic & Purpose
Analyze the distribution of records grouped by `HISTCODE` to highlight transaction-level variances.
* **Data represented:** Event count frequency per HISTCODE and their respective delta variances.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 10

#### 🎯 Logic & Purpose
Compare the operation-level event counts grouped by the `OPER` column in both datasets.
* **Data represented:** Event counts per operation step in Production and Fabric.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 11

#### 🎯 Logic & Purpose
Evaluate event assignment consistency across personnel by comparing record counts per `EMPID`.
* **Data represented:** Event count frequency per employee ID in both environments.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 12

#### 🎯 Logic & Purpose
Verify hardware-level activity by comparing record counts grouped by the `MACHINE` column.
* **Data represented:** Event count distribution per machine/tool.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 13

#### 🎯 Logic & Purpose
Calculate high-level summary metrics containing row, lot, oper, user, machine, and empid distinct counts.
* **Data represented:** High-level profiling comparison across key dimensions.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 14

#### 🎯 Logic & Purpose
Reconcile payload columns for matching composite keys to identify attribute-level mismatches.
* **Data represented:** Mismatch counts for OPER, TRANS, HIST_REC, MACHINE, USERNAME, COMMAND, and EMPID on matching keys.

#### 💻 Code
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

#### 📊 Result
```text

+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|OPER_MISMATCH|OPERDESC_MISMATCH|OPERLONGDESC_MISMATCH|TRANS_MISMATCH|HIST_REC_MISMATCH|MACHINE_MISMATCH|USERNAME_MISMATCH|COMMAND_MISMATCH|EMPID_MISMATCH| 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
|10           |10               |10                   |1466          |2204             |40              |68               |318             |0             | 
+-------------+-----------------+---------------------+--------------+-----------------+----------------+-----------------+----------------+--------------+ 
 

 
 
 

15. TRANS mismatch details (sample)
```

---

### 🔹 Cell 15

#### 🎯 Logic & Purpose
Extract a granular row sample where transaction descriptions (`TRANS`) differ for identical keys.
* **Data represented:** Side-by-side comparison of PROD_TRANS and FABRIC_TRANS values.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 16

#### 🎯 Logic & Purpose
Extract a granular row sample where system commands (`COMMAND`) differ for identical keys.
* **Data represented:** Side-by-side comparison of PROD_COMMAND and FABRIC_COMMAND values.

#### 💻 Code
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

#### 📊 Result
```text

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

---

### 🔹 Cell 17

#### 🎯 Logic & Purpose
Extract a granular row sample where history annotations (`HIST_REC`) differ for identical keys.
* **Data represented:** Side-by-side comparison of PROD_HIST_REC and FABRIC_HIST_REC strings.

#### 💻 Code
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

#### 📊 Result
```text

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