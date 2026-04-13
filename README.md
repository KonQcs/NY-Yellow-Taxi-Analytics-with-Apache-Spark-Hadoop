<img width="158" height="158" alt="UTH_greek_logo" src="https://github.com/user-attachments/assets/9dd2a451-bca2-4688-aa4a-61c447407a13" />

# NY Yellow Taxi Analytics with Apache Spark & Hadoop (2024–2025)

> **Implemented by:** Michail Solomonidis & Konstantinos Kiousis
> **Course:** Big Data Management Systems — University of Thessaly
> **Academic Year:** 2024–2025

This repository contains the code and final report for a semester project on large-scale data processing using Apache Spark and Apache Hadoop. The project focuses on **information extraction and performance evaluation** over real New York City taxi trip data using:

- MapReduce with the **Spark RDD API**
- **SparkSQL** with the **DataFrame API**
- Different storage formats (**CSV vs Parquet**)
- Catalyst Optimizer analysis and **join strategy evaluation**

---

## 📊 Key Results at a Glance

| Query | Best Method | Best Time | Worst Method | Worst Time | Speedup |
|-------|-------------|-----------|--------------|------------|---------|
| Q1 | RDD | 245s | DataFrame (UDF) | crashed (OOMKilled) | — |
| Q2 | RDD | 135s | DataFrame + UDF | 528s | ~4× |
| Q3 | SQL + Parquet | 55s | DataFrame + CSV | 948s | **17×** |
| Q4 | SQL + Parquet | 54s | SQL + CSV | 414s | ~8× |
| Q5 | DataFrame + Parquet | 240s | DataFrame + CSV | 579s | ~2.4× |
| Q6 (scaling) | 8×1×2 executors | 352s | 2×4×8 executors | 421s | ~1.2× |

> CSV → Parquet conversion time: **376 seconds**

---

## 1. Technologies & Environment

The project is developed and executed in a distributed environment provided by the course infrastructure:

- Use the **remote Kubernetes** environment described in the course lab guides.
- Configure **Spark Job History Server** locally (via Docker & Docker Compose) to read job logs from HDFS on the remote cluster.
- Software versions:
  - **Apache Hadoop**: ≥ 3.3
  - **Apache Spark**: ≥ 3.5
  - **Python** (PySpark)

---

## 2. Datasets

All datasets are real NYC yellow taxi trip data. Trip records for 2024 are available online:
https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

In the lab HDFS, the following files are provided:

| Dataset | Description | HDFS URI |
|---------|-------------|----------|
| `yellow_tripdata_2024.csv` | 2024 taxi trips (zone-based, TLC Location IDs) | `hdfs://hdfs-namenode:9000/data/yellow_tripdata_2024.csv` |
| `yellow_tripdata_2015.csv` | 2015 taxi trips (with GPS coordinates) | `hdfs://hdfs-namenode:9000/data/yellow_tripdata_2015.csv` |
| `taxi_zone_lookup.csv` | NYC zone metadata (Borough, Zone, etc.) | `hdfs://hdfs-namenode:9000/data/taxi_zone_lookup.csv` |

### 2.1. 2024 Trip Data — `yellow_tripdata_2024.csv`

Each row describes a single taxi trip in 2024. Example:

```
VendorID,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,RatecodeID,store_and_fwd_flag,PULocationID,DOLocationID,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,improvement_surcharge,total_amount,congestion_surcharge,Airport_fee
1,2024-09-01T00:05:51.000,2024-09-01T00:45:03.000,1,9.8,1,N,138,48,1,47.8,10.25,0.5,13.3,6.94,1.0,79.79,2.5,1.75
```

**Field descriptions:**

| Field | Description |
|-------|-------------|
| `VendorID` | TPEP provider: 1 = Creative Mobile Technologies; 2 = Curb Mobility; 6 = Myle Technologies; 7 = Helix |
| `tpep_pickup_datetime` | Date & time when the meter was engaged |
| `tpep_dropoff_datetime` | Date & time when the meter was disengaged |
| `passenger_count` | Number of passengers in the vehicle |
| `trip_distance` | Trip distance in miles (as reported by the meter) |
| `RatecodeID` | Final rate code: 1 = Standard; 2 = JFK; 3 = Newark; 4 = Nassau/Westchester; 5 = Negotiated; 6 = Group; 99 = Unknown |
| `store_and_fwd_flag` | Whether the trip was stored before sending (`Y`/`N`) |
| `PULocationID` | TLC zone ID for pickup location |
| `DOLocationID` | TLC zone ID for dropoff location |
| `payment_type` | 0 = Flex; 1 = Credit card; 2 = Cash; 3 = No charge; 4 = Dispute; 5 = Unknown; 6 = Voided |
| `fare_amount` | Metered fare based on time and distance |
| `extra` | Additional charges and surcharges |
| `mta_tax` | MTA tax, automatically triggered based on meter rate |
| `tip_amount` | Tip amount (credit card tips only) |
| `tolls_amount` | Total tolls paid on the trip |
| `improvement_surcharge` | Improvement surcharge (introduced 2015) |
| `total_amount` | Total amount charged (excluding cash tips) |
| `congestion_surcharge` | Total congestion surcharge collected |
| `Airport_fee` | Surcharge for pickups at LaGuardia and JFK airports |
| `cbd_congestion_fee` | Congestion relief zone fee (MTA), introduced 5 January 2025 |

---

### 2.2. 2015 Trip Data — `yellow_tripdata_2015.csv`

Contains 2015 taxi trips including **raw GPS coordinates**.

| Column | Description |
|--------|-------------|
| `VendorID` | Provider ID (1 or 2) |
| `tpep_pickup_datetime` | Pickup date and time |
| `tpep_dropoff_datetime` | Dropoff date and time |
| `passenger_count` | Number of passengers |
| `trip_distance` | Trip distance in miles |
| `pickup_longitude` | Pickup longitude (degrees) |
| `pickup_latitude` | Pickup latitude (degrees) |
| `RateCodeID` | Rate code (1 = Standard, 2 = JFK, etc.) |
| `store_and_fwd_flag` | Whether trip was stored before sending (`Y`/`N`) |
| `dropoff_longitude` | Dropoff longitude |
| `dropoff_latitude` | Dropoff latitude |
| `payment_type` | Payment method (1 = Credit, 2 = Cash, etc.) |
| `fare_amount` | Fare amount |
| `extra` | Extra charges (e.g., night surcharge) |
| `mta_tax` | MTA tax ($0.50) |
| `tip_amount` | Tip amount |
| `tolls_amount` | Toll amount |
| `improvement_surcharge` | $0.30 surcharge from 2015 onwards |
| `total_amount` | Total (fare + extras + tip + tolls + taxes) |

---

### 2.3. Zone Metadata — `taxi_zone_lookup.csv`

Maps TLC Location IDs to boroughs and zones.

```
"LocationID","Borough","Zone","service_zone"
1,"EWR","Newark Airport","EWR"
2,"Queens","Jamaica Bay","Boro Zone"
3,"Bronx","Allerton/Pelham Gardens","Boro Zone"
4,"Manhattan","Alphabet City","Yellow Zone"
```

- **LocationID**: Unique numeric ID for each zone
- **Borough**: `"Manhattan"`, `"Queens"`, `"Brooklyn"`, `"Bronx"`, `"Staten Island"`, `"EWR"`
- **Zone**: Neighborhood / zone name
- **service_zone**: `"Yellow Zone"` (yellow cabs), `"Boro Zone"` (green cabs/other), `"EWR"` (Newark Airport)

---

## 3. Project Goals

1. **Hands-on experience with distributed systems** — Installation, configuration, and management of Apache Spark and Apache Hadoop
2. **Data analysis at scale** — Using Spark APIs (RDD, DataFrame, SQL) to process large datasets
3. **Performance & optimization understanding** — Studying how data formats, cluster configurations, and Spark's Catalyst Optimizer affect query planning and execution

---

## 4. Part 1A — Information Extraction (SQL & MapReduce)

Although the data is provided as plain text (CSV), analytical queries over raw CSV are not efficient. To improve performance, Spark can load data into optimized **binary formats** (e.g., Parquet) that:

- Reduce memory and disk footprint → less I/O and faster execution
- Store auxiliary statistics (e.g., min/max per block) to skip unnecessary data during query execution

### Task

Queries are implemented in **two different ways**:

1. **MapReduce using the RDD API** — Implement MapReduce-style logic directly over the CSV files
2. **SparkSQL using the DataFrame API** — Load CSVs into DataFrames and express queries in SparkSQL

Then:

3. **Convert CSV → Parquet** — Transform text files into Parquet, reload, re-run SparkSQL variants, and measure conversion time
4. **Compare execution times** across: (a) RDD/MapReduce over CSV, (b) SQL over CSV, (c) SQL over Parquet

---

## 5. Queries Q1–Q6 — Description & Results

### Q1 — Average Pickup Coordinates by Hour (2015 dataset)

For each hour of the day (00–23), compute the **average pickup latitude and longitude**, ignoring records with zero coordinates. Sort results by hour ascending.

**Example output:**

| HourOfDay | Avg Latitude | Avg Longitude |
|-----------|-------------|---------------|
| 00 | 40.… | -73.… |
| 01 | 40.… | -73.… |

**Results:**

| Implementation | Time (sec) |
|----------------|-----------|
| RDD | 245 |
| DataFrame | 434 |
| DataFrame + UDF | ❌ OOMKilled |

**Analysis:** The RDD implementation was faster because the DataFrame + UDF approach crashed with out-of-memory errors. UDFs bypass Catalyst Optimizer optimizations and process data row-by-row, making them dangerous at scale. The plain DataFrame (without UDF) was stable but slower due to scheduling overhead. **Conclusion:** DataFrame without UDF is the most stable and safe choice; RDD is useful for low-level control.

---

### Q2 — Max Haversine Distance per Vendor (2015 dataset)

For each **VendorID**, find the trip with the maximum geodesic distance (Haversine, in km), with constraints: duration > 10 minutes and distance ≤ 50 km.

**Haversine Formula:** For two points (φ₁, λ₁) and (φ₂, λ₂) in radians:
- Δφ = φ₂ − φ₁, Δλ = λ₂ − λ₁
- a = sin²(Δφ/2) + cos(φ₁)·cos(φ₂)·sin²(Δλ/2)
- c = 2·atan2(√a, √(1−a))
- d = R·c, where R = 6371 km

**Example output:**

| VendorID | Max Haversine Distance (km) | Duration (min) |
|----------|----------------------------|----------------|
| 1 | 38.72 | 47.3 |
| 2 | 42.15 | 53.8 |

**Results:**

| Implementation | Time (sec) |
|----------------|-----------|
| RDD | 135 |
| SQL DataFrame | 311 |
| DataFrame + UDF | 528 |

**Analysis:** RDD was ~4× faster than DataFrame + UDF, an unusual reversal. The reason: Haversine requires custom math functions, and UDF implementations process data row-by-row without Catalyst optimizations. RDD used `.reduceByKey()` efficiently in pure Python logic. **Conclusion:** When UDFs are unavoidable, RDD can outperform DataFrame. If built-in Spark functions can replace the UDF, DataFrame/SQL will be faster.

---

### Q3 — Total Trips per Borough (2024 dataset)

For each NYC **Borough**, compute the total number of trips that start and end in that borough, using a join with `taxi_zone_lookup.csv`. Sort by total trips descending.

**Example output:**

| Borough | TotalTrips |
|---------|-----------|
| Manhattan | 124,520 |
| Queens | 78,250 |
| Brooklyn | 45,200 |
| Bronx | 36,710 |

**Results:**

| Implementation | Time (sec) |
|----------------|-----------|
| DataFrame + CSV | 948 |
| SQL + CSV | 471 |
| DataFrame + Parquet | 60 |
| SQL + Parquet | **55** |

**Analysis:** The **17× speedup** from SQL+Parquet vs DataFrame+CSV is the most striking result in this project. Parquet's columnar format means Spark reads only the required columns, applies pushdown filters, and avoids unnecessary shuffle. The Catalyst Optimizer can fully exploit Parquet metadata for projection and partition pruning. **Conclusion:** Storage format choice is as important as query logic — SQL + Parquet is the gold standard for join-heavy queries.

---

### Q4 — Night Trips per Vendor (2024 dataset)

Count trips that start between 23:00–23:59 or 00:00–06:59, grouped by VendorID.

**Example output:**

| VendorID | NightTrips |
|----------|-----------|
| 1 | 54,820 |
| 2 | 47,210 |

**Results:**

| Implementation | Time (sec) |
|----------------|-----------|
| SQL + CSV | 414 |
| SQL + Parquet | **54** |

**Analysis:** The identical SQL query runs ~8× faster on Parquet. Parquet allows Spark to read only `tpep_pickup_datetime` and `VendorID`, while CSV requires full-file parsing regardless of which columns are needed. **Conclusion:** For queries that touch few columns on large datasets, Parquet provides dramatic I/O savings.

---

### Q5 — Most Frequent Zone Pairs (2024 dataset)

Find the zone pairs (PickupZone → DropoffZone) with the most trips between them, excluding same-zone trips. Involves two joins, filtering, and grouping.

**Example output:**

| Pickup Zone | Dropoff Zone | TotalTrips |
|-------------|-------------|-----------|
| Midtown Center | Upper East Side | 32,140 |
| Upper West Side | Times Square | 28,790 |

**Results:**

| Implementation | Time (sec) |
|----------------|-----------|
| DataFrame + CSV | 579 |
| DataFrame + Parquet | **240** |

**Analysis:** Parquet reduced execution time by 58%. With two joins and an aggregation, Parquet's columnar format allows Spark to read only the five needed columns (`PULocationID`, `DOLocationID`, `LocationID`, `Zone`, `Borough`), and the Catalyst Optimizer generates a more efficient execution plan. **Conclusion:** For multi-join queries, Parquet combined with the DataFrame API is significantly more efficient.

---

### Q6 — Parallel Execution & Scaling (2024 dataset)

Compute trip count and average distance per month, grouped by `tpep_pickup_datetime` and aggregated on `trip_distance`. The goal was to study how different executor configurations affect performance. Total resources: 8 cores, 16 GB memory.

**Results:**

| Configuration | Executors × Cores × Memory | Time (sec) |
|---------------|---------------------------|-----------|
| 2×4×8 | 2 executors, 4 cores, 8 GB each | 421 |
| 4×2×4 | 4 executors, 2 cores, 4 GB each | 380 |
| 8×1×2 | 8 executors, 1 core, 2 GB each | **352** |

**Analysis:** Horizontal scaling (more, lighter executors) outperformed vertical scaling (fewer, stronger executors). For this embarrassingly parallel I/O + aggregation query, the 8×1×2 configuration provided better pipelining, more data partitioning parallelism, and better load distribution. **Conclusion:** "More small" > "Few powerful" when the query is embarrassingly parallel. Vertical scaling was resilient but not efficient for large datasets.

---

## 6. Part 1B — Catalyst Optimizer & Join Strategy Analysis

### Join Strategy: BroadcastHashJoin

For **Q3, Q4, and Q5**, the Catalyst Optimizer consistently selected **BroadcastHashJoin** with `BuildRight` as the build side.

**Why BroadcastHashJoin was chosen:**
- The `taxi_zone_lookup.csv` table is small (~1 MB, 263 rows), as confirmed by Physical Plan output: `Statistics(sizeInBytes ≈ 1 MB)`
- Small datasets can be broadcast to all executors cheaply, avoiding the expensive shuffle required by SortMergeJoin
- Default `spark.sql.autoBroadcastJoinThreshold` is 10 MB — the lookup table falls well within this threshold

**This strategy is theoretically correct** because BroadcastHashJoin eliminates network shuffle entirely for one side of the join, making it optimal when one dataset is small enough to fit in executor memory.

Physical plan diagrams for Q3_DF, Q3_DF_SQL, Q3_PARQ, Q5_DF, and Q5_PARQ are available in `final_report.pdf`.

---

### 6.1. Join with Limited Taxi Records

First 50 records from the taxi dataset were joined with the lookup table using SparkSQL on Parquet. Spark chose **BroadcastHashJoin** because both sides were small after the `LIMIT 50` subquery.

### 6.2. Forcing a Different Join Strategy

By adjusting `spark.sql.autoBroadcastJoinThreshold = -1` (disabling broadcast), Spark fell back to **SortMergeJoin**, resulting in higher execution time due to the additional shuffle cost. This confirmed that BroadcastHashJoin was the correct and optimal choice for this dataset size.

For reference: https://spark.apache.org/docs/latest/sql-performance-tuning.html

---

## 7. Implementation Requirements

1. **Connect to the remote Kubernetes cluster** and set up the environment as described in the course guides
2. **Configure Spark Job History Server** locally via Docker/Docker Compose reading job history from remote HDFS
3. **Write code to:**
   - Read all datasets from HDFS
   - Clean/transform as needed
   - Store in **Parquet** format at: `hdfs://hdfs-namenode:9000/user/{username}/data/parquet/`
   > Note: `yellow_tripdata_2024.csv` requires preprocessing — it is not initially in a Spark-friendly schema

4. **Implementations per query:**

| Query | Methods Implemented |
|-------|-------------------|
| Q1 | RDD API, DataFrame API, DataFrame + UDF |
| Q2 | RDD API, DataFrame API, SQL API |
| Q3 | DataFrame API (CSV + Parquet), SQL API (CSV + Parquet) |
| Q4 | SQL API (CSV + Parquet) |
| Q5 | DataFrame API (CSV + Parquet) |
| Q6 | DataFrame API with 3 executor configurations (horizontal & vertical scaling) |

---

## 8. Cluster Resource Configurations

All runs used a total of **8 cores** and **16 GB** memory:

| Config | Executors | Cores/Executor | Memory/Executor |
|--------|-----------|---------------|----------------|
| 2×4×8 | 2 | 4 | 8 GB |
| 4×2×4 | 4 | 2 | 4 GB |
| 8×1×2 | 8 | 1 | 2 GB |

---

## 9. Key Takeaways

1. **Parquet vs CSV matters enormously** — up to 17× speedup observed in Q3. Always prefer columnar formats for analytical workloads.
2. **Avoid UDFs when possible** — UDFs bypass the Catalyst Optimizer, causing OOM errors (Q1) and 4× slowdowns (Q2). Use built-in Spark functions wherever available.
3. **RDDs can win for math-heavy queries** — when UDFs are unavoidable, RDD with `.reduceByKey()` can be faster than DataFrame + UDF (Q2).
4. **SQL + Parquet is the gold standard** — for join-heavy queries with grouping, SQL on Parquet consistently outperformed all other combinations.
5. **Horizontal scaling outperforms vertical scaling** for embarrassingly parallel I/O queries — more, lighter executors beat fewer, heavier ones (Q6).
6. **BroadcastHashJoin is optimal for small lookup tables** — Catalyst correctly chose it in all join queries; the ~1 MB zone lookup table is well within broadcast thresholds.

---

## 10. Notes & Hints

1. **RDD API input format** — Each line is read as a `String`. Parse fields into proper types (`Int`, `Double`, `Timestamp`). Date parsing pattern: `'%Y-%m-%d %H:%M:%S'`
2. **Q2 speed** — Trip speed can be defined as `speed = distance / duration`
3. **SQL & Optimizer** — Use `EXPLAIN` to inspect execution plans. Tuning `spark.sql.autoBroadcastJoinThreshold` significantly changes join strategies and runtimes

---

## 📁 Repository Structure

```
Bigdata_UTH_Final/
├── Parquet_Transform.py       # CSV → Parquet conversion
├── Q1_RDD.py                  # Q1 via RDD API
├── Q1_DF.py                   # Q1 via DataFrame API
├── Q2_RDD.py                  # Q2 via RDD API
├── Q2_DF.py                   # Q2 via DataFrame API
├── Q2_DF_SQL.py               # Q2 via SQL on DataFrame
├── Q3_DF.py                   # Q3 via DataFrame + CSV
├── Q3_DF_SQL.py               # Q3 via SQL + CSV
├── Q3_PARQ.py                 # Q3 via DataFrame + Parquet
├── Q3_SQL_PARQ.py             # Q3 via SQL + Parquet
├── Q4_SQL.py                  # Q4 via SQL + CSV
├── Q4_SQL_PARQ.py             # Q4 via SQL + Parquet
├── Q5_DF.py                   # Q5 via DataFrame + CSV
├── Q5_PARQ.py                 # Q5 via DataFrame + Parquet
├── Q6_HOR.py                  # Q6 horizontal scaling
├── Q6_VER1.py                 # Q6 vertical scaling (config 1)
├── Q6_VER2.py                 # Q6 vertical scaling (config 2)
└── final_report.pdf           # Full report with charts and Physical Plan diagrams
```

---

## 👥 Authors

- **Michail Solomonidis** — AM 2121063
- **Konstantinos Kiousis** — AM 2121129

Department of Informatics and Telecommunications, University of Thessaly
